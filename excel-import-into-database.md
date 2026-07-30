Service nằm trong folder **ProductImport**

- Hàm tổng:
```CSharp
public async Task<Response.ImportResult> ImportFromExcel(Stream fileStream)  
{  
    var result = new Response.ImportResult();  
  
    try  
    {  
        using var ms = await BufferStreamAsync(fileStream);  
        await ValidateFileStructureAsync(ms);  
  
        var rows = await ReadRowsAsync(ms);  
        result.TotalRecords = rows.Count;  
  
        var isb = await BuildUpsertBatchAsync(rows);  
        await PersistBatchAsync(isb);  
  
        result.InsertedCount = isb.ToInsert.Count;  
        result.UpdatedCount = isb.ToUpdate.Count;  
    }    catch (AppException) { throw; }  
    catch (Exception ex)  
    {        Console.WriteLine($"Import error: {ex.Message}");  
        throw new BadRequestException($"Excel processing error: {ex.Message}");  
    }  
    return result;  
}
```


Hàm này có nhiệm vụ import dữ liệu từ excel vào database của mình để có thể lưu trữ.

```Csharp
private static async Task<MemoryStream> BufferStreamAsync(Stream source)  
{  
    var ms = new MemoryStream();  
    await source.CopyToAsync(ms);  
    return ms;  
}
```
Hàm này có vai trò, tải toàn bộ nội dung của file từ luồng mạng(**network stream**) -> vào bộ nhớ ram của server (**memory stream**).

Tại sao lại làm vậy? Vì khi người dùng upload một file lên server, file đó được truyền đi ở dạng stream, chạy dọc qua mạng. Dạng stream này có một hạn chế rất lớn, nó chỉ cho phép đọc một chiều từ đầu đến cuối mà không thể quay xe(**CanSeek** = false) để có thể đọc đi đọc lại được nhiều lần.

Vai trò cơ bản của các hàm phía sau:
**ValidateFileStructureAsync**: validate cái hàm nhận vào có các đặc tính sau: 
- Không được rỗng
- Ô đầu tiên phải có chữ ID
- Đủ độ đài tối thiếu là 40 cột
```CSharp
private static async Task ValidateFileStructureAsync(MemoryStream ms)  
{  
    ms.Position = 0;  //bắt đầu từ vị trí 0 để đảm bảo đọc từ đầu
    var rawRows = await MiniExcel.QueryAsync(ms, useHeaderRow: false);
    //dùng thư viện MiniExcel để đọc các rows bên trong file excel một cách nhanh chóng, và sử dùng useHeaderRow:false, có nghĩa là đừng tự động lấy dòng đầu tiên làm tiêu đề, hãy cứ đọc nó như một dòng dữ liệu thô bình thường  
//bổ sung, vì sử dụng useHeaderRow: false không lấy dòng đầu tiên làm header(tiêu đề), nên miniExcel không biết đặt tên các col là gì -> để giải quyết tình trạng đó, nó tự động đặt tên theo chuẩn của microsoft excel luôn. 
    var headerRow = rawRows.FirstOrDefault() as IDictionary<string, object?>;  
//lấy ra các rows đầu tiên, vì nếu lấy ra được thì nó một một cục dữ liệu, máy tính không thể hiểu được, nên nói với máy tính rằng, thằng data có dữ liệu ở dạng Dictionary, với các key là và cột "A", "B", "C" (miniexcel tự thêm)
    if (headerRow is not { Count: > 0 })  
        throw new BadRequestException("Excel file is empty");  
  
    headerRow.TryGetValue("A", out var valA);  
    var colA = valA?.ToString() ?? "";  
  
    if (string.IsNullOrWhiteSpace(colA) ||  
        !colA.Contains("ID", StringComparison.OrdinalIgnoreCase) ||  
        headerRow.Count < 40)  
    {        throw new BadRequestException(  
            "Invalid file format. Please upload an Excel file exported from Nhanh.vn.");  
    }}
```



**ReadRowsAsync**: hàm này có nhiệm vụ dùng thư viện **MiniExcel** để map các rows trong file excel thành một danh sách các object trong C#.
- Điểm lưu ý: hàm này sẽ tự động tính toán và gắn số dòng thực tế trong Excel(SourceRowNumber = i + 2) vào từng object, để nếu sau này có lỗi thì biết được trong file đang lỗi ở dòng số X nào đó. 
```CSharp
//việc đọc chính thức được diễn ra ở đây
private async Task<List<Request.ExcelRow>> ReadRowsAsync(MemoryStream ms)  
{  
    ms.Position = 0;  
    var rows = (await MiniExcel.QueryAsync<Request.ExcelRow>(ms)).ToList();  
  
    if (rows.Count == 0)  
        throw new BadRequestException("Excel file contains no data");  
  
    for (var i = 0; i < rows.Count; i++)  
        rows[i].SourceRowNumber = i + 2;  
  //SourceRowNumber: dùng để truy viết khi không parse được từ file excel rồi cái response của mình.
  //cụ thể trong list bắt đầu bằng 0, và giá trị thực của file excel bắt đầu bằng dòng 2, vì vậy mà sao lại có i + 2.
    return rows;  
}
```



**BuildUpsertBatchAsync & ResolveProduct**: bộ não gom hàng
Thay vì cứ mỗi dòng trong Excel lại chạy vào db hỏi một lần(dây chậm hệ thống), hàm này gom lại tất cả **NhanhProductId** có trong file Excel lại, rồi gọi DB đúng 1 lần duy nhất để lấy ra danh sách các sản phẩm đang có sẵn (existingMap).
**Phân loại**: 
Chạy vòng lặp qua từng dòng Excel. Nếu ID dòng đó đã có nằm trong exsitingMap -> Đẩy sản phẩm đó vào danh sách chờ Cập nhật (**ToUpdate**). Ngược lại -> Tạo **Guid** mới và đẩy vào danh sách Tạo mới (**ToInsert**).

```CSharp
private struct ImportBatch  
{  
    public List<ProductEntity> ToInsert;  
    public List<ProductEntity> ToUpdate;  
}  

private async Task<ImportBatch> BuildUpsertBatchAsync(List<Request.ExcelRow> rows)  
{  
    var idsInFile = rows  
        .Select(r => NullIfEmpty(r.NhanhProductId))  
        .Where(id => id != null)  
        .Distinct()  
        .ToList();  
//lúc này ta có được một list nhanhProdcutId từ file excel mà ta đã nhờ miniExcel đọc qua  
    var existingMap = idsInFile.Count > 0  
        ? await _dbContext.Products  
            .Where(p => p.NhanhProductId != null && idsInFile.Contains(p.NhanhProductId) && !p.IsDeleted)  
            .ToDictionaryAsync(p => p.NhanhProductId!)  
        : new Dictionary<string, ProductEntity>();  
 //lúc này ta có một dictionary chứ với key là: nhanhProductId và value là entity tương ứng của nó 
    var batch = new ImportBatch  
    {  
        ToInsert = new List<ProductEntity>(),  
        ToUpdate = new List<ProductEntity>()  
    };  
    
    //lặp từng rows trong file excel đã được đọc
    foreach (var row in rows)  
    {        
    //product lúc này có 2 batch: một là product đã tồn tại trong db, một là phải tạo mới nó lần lượt với isUpdate: là true
	    //lúc nào có được product rồi
	    var product = ResolveProduct(row, existingMap, out var isUpdate);  
        //tiến hành update nó
        ApplyRowToProduct(row, product);  
        product.UpdatedAt = DateTimeOffset.UtcNow;  
		//add nó vào batch tương ứng
        if (isUpdate)  
            batch.ToUpdate.Add(product);  
        else  
            batch.ToInsert.Add(product);  
    }  
    return batch;  
}
```

```CSharp
private static ProductEntity ResolveProduct(  
    Request.ExcelRow row,  
    Dictionary<string, ProductEntity> existingMap,  
    out bool isUpdate)  
{  
    var cleanId = NullIfEmpty(row.NhanhProductId);  
  
    if (cleanId != null && existingMap.TryGetValue(cleanId, out var existing))  
    {        isUpdate = true;  
        return existing;  
    }  
    isUpdate = false;  
    return new ProductEntity  
    {  
        Id = Guid.NewGuid(),  
        CreatedAt = DateTimeOffset.UtcNow  
    };  
}
```

**ApplyRowToProduct**: "chuyển nhà dữ liệu":
Copy giá trị từ file excel sang cấu trúc entity của db. Trong lúc cope, sử dụng các hàm **Parse helpers** ở cuối code để chuẩn hóa dữ liệu (ví dụ: đổi chuỗi chữ "100.5" thành Decimal, đổi ngày tháng dạng chuỗi thành kiểu dữ liệu **DataTimeOffset**)
```CSharp
private static void ApplyRowToProduct(  
    Request.ExcelRow row,  
    ProductEntity product)  
{  
    product.SourceRowNumber = row.SourceRowNumber;  
    product.NhanhProductId = NullIfEmpty(row.NhanhProductId);  
    product.ParentProductId = NullIfEmpty(row.ParentProductId);  
    product.Barcode = NullIfEmpty(row.Barcode);  
    product.ProductType = NullIfEmpty(row.ProductType);  
    product.ProductCode = NullIfEmpty(row.ProductCode);  
    product.ProductName = NullIfEmpty(row.ProductName);  
    product.Unit = NullIfEmpty(row.Unit);  
    product.CoverImage = NullIfEmpty(row.CoverImage);  
  
    product.WarrantyMonths = ParseInt(row.WarrantyMonthsRaw);  
    product.Weight = ParseDecimal(row.WeightRaw);  
  
    product.ImportPrice = ParseDecimal(row.ImportPriceRaw);  
    product.ImportVat = ParseDecimal(row.ImportVatRaw);  
    product.ImportPriceWithVat = ParseDecimal(row.ImportPriceWithVatRaw);  
    product.ImportVatMode = NullIfEmpty(row.ImportVatMode);  
  
    product.Price = ParseDecimal(row.PriceRaw);  
    product.PriceWithVat = ParseDecimal(row.PriceWithVatRaw);  
    product.BranchPrice = ParseDecimal(row.BranchPriceRaw);  
    product.BranchPriceWithVat = ParseDecimal(row.BranchPriceWithVatRaw);  
    product.SaleVatMode = NullIfEmpty(row.SaleVatMode);  
    product.OldPrice = ParseDecimal(row.OldPriceRaw);  
    product.Vat = ParseDecimal(row.VatRaw);  
  
    product.ProfitPercentage = ParseDecimal(row.ProfitPercentageRaw);  
    product.CostPrice = ParseDecimal(row.CostPriceRaw);  
    product.WholesalePrice = ParseDecimal(row.WholesalePriceRaw);  
    product.WholesalePriceWithVat = ParseDecimal(row.WholesalePriceWithVatRaw);  
    product.WholesaleBranchPrice = ParseDecimal(row.WholesaleBranchPriceRaw);  
    product.WholesaleBranchPriceWithVat = ParseDecimal(row.WholesaleBranchPriceWithVatRaw);  
  
    product.TotalStock = ParseDecimal(row.TotalStockRaw);  
    product.StockTransferring = ParseDecimal(row.StockTransferringRaw);  
    product.Stock = ParseDecimal(row.StockRaw);  
    product.DeliveringStock = ParseDecimal(row.DeliveringStockRaw);  
    product.WarehouseStock = ParseDecimal(row.WarehouseStockRaw);  
    product.StockError = ParseDecimal(row.StockErrorRaw);  
    product.Held = ParseDecimal(row.HeldRaw);  
    product.AvailableForSale = ParseDecimal(row.AvailableForSaleRaw);  
    product.PendingImportStock = ParseDecimal(row.PendingImportStockRaw);  
    product.PreOrder = ParseDecimal(row.PreOrderRaw);  
  
    product.Status = NullIfEmpty(row.Status);  
    product.CategoryCode = NullIfEmpty(row.CategoryCode);  
    product.CategoryName = NullIfEmpty(row.CategoryName);  
    product.Brand = NullIfEmpty(row.Brand);  
    product.Supplier = NullIfEmpty(row.Supplier);  
    product.Slug = NullIfEmpty(row.Slug);  
    product.WebOrder = ParseInt(row.WebOrderRaw);  
    product.NhanhCreatedAt = ParseDateTimeOffset(row.NhanhCreatedAtRaw);  
  
    product.BranchPriceDuplicate = ParseDecimal(row.BranchPriceDuplicateRaw);  
    product.WholesaleBranchPriceDuplicate = ParseDecimal(row.WholesaleBranchPriceDuplicateRaw);  
    product.PriceLabel = NullIfEmpty(row.PriceLabel);  
    product.Size = NullIfEmpty(row.Size);  
    product.Color = NullIfEmpty(row.Color);  
}
```



**PersistBatchAsync**: "cú rollback an toàn"
Mở một transaction, lưu các các mặt hàng ở cả 2 types: update và insert, nếu có lỗi trong quá trình lưu thì lập tấp rollback về ban đầu để tránh mất đồng nhất dữ liệu.

```CSharp
private async Task PersistBatchAsync(ImportBatch batch)  
{  
    await using var tx = await _dbContext.Database.BeginTransactionAsync();  
    try  
    {  
        if (batch.ToInsert.Count > 0)  
            await _dbContext.Products.AddRangeAsync(batch.ToInsert);  
  
        if (batch.ToUpdate.Count > 0)  
            _dbContext.Products.UpdateRange(batch.ToUpdate);  
  
        await _dbContext.SaveChangesAsync();  
        await tx.CommitAsync();  
    }    catch  
    {  
        await tx.RollbackAsync();  
        throw;  
    }}
```


- Những hàm helper:
```CSharp
private static string? NullIfEmpty(string? val)  
{  
    if (string.IsNullOrWhiteSpace(val)) return null;  
    var trimmed = val.Trim();  
    return trimmed.Equals("null", StringComparison.OrdinalIgnoreCase) ? null : trimmed;  
}  
  
/** Parse số thập phân từ Excel (hỗ trợ cả dấu phẩy và chấm). */  
private static decimal? ParseDecimal(string? val)  
{  
    if (string.IsNullOrWhiteSpace(val)) return null;  
    val = val.Trim();  
  
    if (decimal.TryParse(val, NumberStyles.Any, CultureInfo.InvariantCulture, out var dec))  
        return dec;  
  
    if (decimal.TryParse(val.Replace(',', '.'), NumberStyles.Any, CultureInfo.InvariantCulture, out dec))  
        return dec;  
  
    return null;  
}  
  
/** Parse số nguyên từ Excel, làm tròn nếu là số thập phân. */  
private static int? ParseInt(string? val)  
{  
    if (string.IsNullOrWhiteSpace(val)) return null;  
    val = val.Trim();  
  
    if (double.TryParse(val, NumberStyles.Any, CultureInfo.InvariantCulture, out var dbl))  
        return (int)Math.Round(dbl);  
  
    return null;  
}  
  
/** Parse ngày giờ dạng "yyyy-MM-dd HH:mm:ss" từ Excel. */  
private static DateTimeOffset? ParseDateTimeOffset(string? val)  
{  
    if (string.IsNullOrWhiteSpace(val)) return null;  
    val = val.Trim();  
  
    if (DateTimeOffset.TryParseExact(val, "yyyy-MM-dd HH:mm:ss", CultureInfo.InvariantCulture, DateTimeStyles.AssumeLocal, out var dto))  
        return dto.ToUniversalTime();  
  
    if (DateTimeOffset.TryParse(val, CultureInfo.InvariantCulture, DateTimeStyles.None, out dto))  
        return dto.ToUniversalTime();  
  
    return null;  
}
```
