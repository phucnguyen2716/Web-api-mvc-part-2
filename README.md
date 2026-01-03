# Web API + MVC – Part 2

## 1. Product API

Tài liệu này mô tả **chuẩn thiết kế và triển khai API Product** trong dự án **ASP.NET Core Web API + MVC**, tập trung vào việc **tách bạch rõ ràng trách nhiệm của từng API**.

---

## 📌 Quy ước chung

* **Create sản phẩm (có upload ảnh)** → `POST`
* **Update sản phẩm (KHÔNG đổi ảnh)** → `PUT`
* **Ảnh chỉ được xử lý tại API Create**

### ❗ Nguyên tắc bắt buộc

* ❌ **Không dùng PUT để upload ảnh**
* ❌ **Không dùng POST để update dữ liệu đơn thuần**
* ✅ **Mỗi API chỉ làm đúng 1 nhiệm vụ**
* ✅ **Đúng Content-Type để tránh lỗi 415 / 500**

---

## 1.1 Update Product (KHÔNG upload ảnh)

### 🔹 Mục đích

* Cập nhật **Name / Price / Category**
* **Không xử lý hình ảnh**

---

### 🔹 Endpoint

| Thuộc tính   | Giá trị              |
| ------------ | -------------------- |
| Method       | `PUT`                |
| URL          | `/api/Products/{id}` |
| Content-Type | `application/json`   |

---

### 🔹 Request Body (JSON)

```json
{
  "name": "Sản phẩm cập nhật",
  "price": 150000,
  "categoryId": 2
}
```

---

### 🔹 Controller Code

```csharp
[HttpPut("{id}")]
public async Task<IActionResult> Update(
    int id,
    [FromBody] ProductDto dto)
{
    try
    {
        var product = await _context.Products.FindAsync(id);
        if (product == null)
            return NotFound(new { message = "Không tìm thấy sản phẩm" });

        var categoryExists = await _context.Categories
            .AnyAsync(c => c.Id == dto.CategoryId);

        if (!categoryExists)
            return BadRequest(new { message = "Danh mục không tồn tại" });

        product.Name = dto.Name;
        product.Price = dto.Price;
        product.CategoryId = dto.CategoryId;

        await _context.SaveChangesAsync();

        return NoContent(); // 204
    }
    catch (Exception ex)
    {
        return StatusCode(500, new
        {
            message = "Lỗi hệ thống",
            detail = ex.InnerException?.Message ?? ex.Message
        });
    }
}
```

---

### 🔹 Response

| Status | Ý nghĩa                 |
| ------ | ----------------------- |
| 204    | Cập nhật thành công     |
| 404    | Không tìm thấy sản phẩm |
| 400    | Category không tồn tại  |
| 500    | Lỗi hệ thống            |

---

## 1.2 Create Product (CÓ upload ảnh)

### 🔹 Mục đích

* Tạo mới sản phẩm
* Upload & lưu ảnh vào `wwwroot/uploads`

---

### 🔹 Endpoint

| Thuộc tính   | Giá trị               |
| ------------ | --------------------- |
| Method       | `POST`                |
| URL          | `/api/Products`       |
| Content-Type | `multipart/form-data` |

---

### 🔹 Form Data

| Key        | Type           | Required |
| ---------- | -------------- | -------- |
| name       | string         | ✅        |
| price      | number         | ✅        |
| categoryId | number         | ✅        |
| image      | file (jpg/png) | ✅        |

---

### 🔹 Controller Code

```csharp
[HttpPost]
public async Task<IActionResult> Create([FromForm] ProductDto dto)
{
    try
    {
        var allowedExtensions = new[] { ".jpg", ".jpeg", ".png" };
        var extension = Path.GetExtension(dto.Image.FileName).ToLower();

        if (!allowedExtensions.Contains(extension))
        {
            return BadRequest(new
            {
                errorCode = "INVALID_FILE_TYPE",
                message = "Chỉ chấp nhận ảnh JPG hoặc PNG"
            });
        }

        var fileName = $"{Guid.NewGuid()}{extension}";
        var uploadFolder = Path.Combine("wwwroot", "uploads");

        if (!Directory.Exists(uploadFolder))
            Directory.CreateDirectory(uploadFolder);

        var filePath = Path.Combine(uploadFolder, fileName);

        using (var stream = new FileStream(filePath, FileMode.Create))
        {
            await dto.Image.CopyToAsync(stream);
        }

        var product = new Product
        {
            Name = dto.Name,
            Price = dto.Price,
            ImageUrl = $"/uploads/{fileName}",
            CategoryId = dto.CategoryId
        };

        _context.Products.Add(product);
        await _context.SaveChangesAsync();

        return CreatedAtAction(nameof(GetById),
            new { id = product.Id }, product);
    }
    catch (Exception ex)
    {
        return StatusCode(500, new
        {
            errorCode = "SYSTEM_ERROR",
            message = "Lỗi hệ thống khi tạo sản phẩm",
            detail = ex.Message
        });
    }
}
```

---

### 🔹 Response

| Status | Ý nghĩa                 |
| ------ | ----------------------- |
| 201    | Tạo sản phẩm thành công |
| 400    | File ảnh không hợp lệ   |
| 500    | Lỗi hệ thống            |

---

## 1.3 Tổng kết chuẩn thiết kế API

### ✅ Đúng chuẩn REST

* `POST` → Create (có side-effect: upload file)
* `PUT` → Update (idempotent, không upload file)

### ✅ Dễ dùng cho MVC / Frontend

* MVC Form Create → `multipart/form-data`
* MVC Edit Form → `application/json`

## 2. Cấu hình CORS (Cross-Origin Resource Sharing)

### 2.1 Đăng ký dịch vụ CORS (Program.cs)

```csharp
builder.Services.AddCors(options =>
{
    options.AddPolicy("AllowSpecificOrigin", policy =>
    {
        policy.WithOrigins("https://localhost:44390") // Chỉ cho phép domain này
              .AllowAnyMethod()   // Cho phép tất cả các HTTP method (GET, POST, PUT, DELETE...)
              .AllowAnyHeader()   // Cho phép tất cả các header
              .AllowCredentials(); // Cho phép gửi cookies hoặc authorization header
    });
});

app.UseCors("AllowSpecificOrigin");
app.UseStaticFiles();
