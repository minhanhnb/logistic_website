# 📊 QUY TRÌNH PHÂN TÍCH SO SÁNH HÓA ĐƠN (MBL vs HBL)

## **BƯỚC 1: KHỞI TẠO VÀ CẤU HÌNH API KEY**

### Cơ chế: 
```
Khi tải trang lần đầu tiên:
1. Ứng dụng kiểm tra xem có biến môi trường GEMINI_API_KEY không
   - Nếu có → Sử dụng ngầm (không cần nhập)
   - Nếu không có → Kiểm tra localStorage (nơi lưu khóa cũ)
   - Nếu cũng không có → Hiện dialog nhập API Key

2. API Key được lưu vào localStorage để dùng lần sau
3. Hiển thị trạng thái: ✅ API đã cấu hình hoặc ❌ Chưa cấu hình
```

---

## **BƯỚC 2: UPLOAD FILE PDF (MBL & HBL)**

### Cách nhận file:
- **Kéo thả (Drag & Drop)**: Người dùng kéo file vào vùng upload
- **Nhấp chọn**: Nhấp vào vùng để mở dialog chọn file
- **Tải và kiểm tra**: 
  - Kiểm tra file có phải PDF không
  - Đọc file vào bộ nhớ (ArrayBuffer)
  - Hiển thị: `✓ File đã tải: [tên file]`
  - Kích hoạt nút "Phân tích & So sánh" khi cả 2 file đã tải

---

## **BƯỚC 3: TRÍCH XUẤT VĂN BẢN TỪ PDF** 

### Quy trình 2 bước (nâng cao):

#### **Bước 3a: Trích xuất Native (Phương pháp cơ bản)**
```
1. Mở PDF bằng PDF.js
2. Duyệt từng trang (page 1, 2, 3, ...)
3. Sử dụng getTextContent() để trích xuất text
   - Nếu text đủ dài (>50 ký tự) → Dùng text này
   - Nếu text quá ít → Bước tiếp theo dùng OCR
```

#### **Bước 3b: OCR bằng Gemini Vision (Phương pháp nâng cao)**
```
Nếu trang PDF chứa hình ảnh hoặc native text quá ít:
1. Render trang PDF thành ảnh PNG (Canvas)
   - Scale 2x để chất lượng tốt hơn
   
2. Chuyển ảnh sang Base64 (định dạng gửi API)

3. Gửi ảnh tới Gemini Vision API:
   - Công nghệ: Google Gemini Flash (nhanh & chính xác)
   - Yêu cầu: "Extract ALL text from this document image"
   - Nhận lại: Toàn bộ text được trích xuất

4. Kết hợp cả native text + OCR text
   - MBL text = [native text] + [OCR text nếu cần]
   - HBL text = [native text] + [OCR text nếu cần]
```

**Ví dụ:**
- MBL có 100 trang: 50 trang là native (text), 50 trang là scan (ảnh)
  → 50 trang native dùng text trực tiếp
  → 50 trang scan gửi qua Vision OCR
  - Kết quả: 100% text được trích xuất

---

## **BƯỚC 4: TRÍCH XUẤT CÁC TRƯỜNG DỮ LIỆU (Field Extraction)**

### Sử dụng AI - Gemini Flash Model

#### **Quy trình:**
```
1. Đóng gói prompt (hướng dẫn rõ ràng):
   - "Trích xuất các thông tin sau từ bill"
   - Danh sách 18 trường cần trích xuất
   - Format: JSON
   - Ví dụ các trường:
     • Shipper (Người gửi)
     • Consignee (Người nhận)
     • Vessel (Tên tàu)
     • Port of Loading / Discharge (Cảng xuất / nhập)
     • Container No (Số cont)
     • Seal No (Số seal)
     • Gross Weight (Cân nặng)
     • Measurement (Đo lường - CBM)
     ... và 10 trường khác

2. Gửi request tới Gemini API:
   - URL: https://generativelanguage.googleapis.com/v1beta/models/gemini-flash-latest:generateContent
   - Payload: JSON chứa prompt + text MBL/HBL
   - Timeout: Bao gồm thời gian xử lý

3. Nhận kết quả:
   - API trả lại JSON với các trường
   - Ví dụ response:
     {
       "Shipper": "ABC Company Ltd",
       "Container No": "TCLU1234567",
       "Port of Loading": "Shanghai",
       "Gross weight": "20000 kg",
       ...
     }

4. Fallback (Dự phòng):
   - Nếu API lỗi hoặc API Key không có
   - Dùng Regular Expression (Regex) để tìm trường
   - Độ chính xác thấp hơn nhưng vẫn hoạt động
```

---

## **BƯỚC 5: SO SÁNH CÁC TRƯỜNG**

### Logic so sánh thông minh:

```
Duyệt tất cả các trường từ cả 2 bill:

Với mỗi trường (Shipper, Container No, ...):
  1. Chuẩn hóa giá trị:
     - Chuyển thành chữ thường
     - Bỏ khoảng trắng thừa
     - VD: "CONTAINER NO: 123" → "container no: 123"

  2. So sánh cấp độ 1: Khớp chính xác?
     MBL: "Shanghai Port" 
     HBL: "Shanghai Port"
     → ✓ KHỚP

  3. So sánh cấp độ 2: So sánh từng part?
     MBL: "John Smith Company"
     HBL: "John Smith"
     - Tách thành: ["john", "smith", "company"] vs ["john", "smith"]
     - Kiểm tra: "john smith" có nằm trong "john smith company" không?
     → ✓ CÓ THỂ CHẤP NHẬN (kiểu chứa)

  4. Ghi nhận kết quả:
     - match = true/false
     - note = "✓ Khớp" hoặc "⚠️ Không khớp"
```

---

## **BƯỚC 6: TÍNH TOÁN THỐNG KÊ**

### Công thức:

```
Tổng trường = Tất cả trường được kiểm tra
Trường khớp = Số trường match = true
Trường khác = Số trường match = false

Độ chính xác (Accuracy) = (Trường khớp / Tổng trường) × 100%

Ví dụ:
- Tổng: 18 trường
- Khớp: 16 trường
- Khác: 2 trường
- Accuracy: (16 / 18) × 100 = 88.9%
```

### Hiển thị:
```
┌─────────────┬─────────────┬─────────────┐
│ Khớp: 16    │ Khác: 2     │ Chính xác: 88%  │
└─────────────┴─────────────┴─────────────┘
```

---

## **BƯỚC 7: HIỂN THỊ BẢNG SO SÁNH CHI TIẾT**

### Định dạng bảng:

```
┌──────────────────┬─────────────┬─────────────┬──────────┬──────────┐
│ Trường dữ liệu   │ MBL Value   │ HBL Value   │ Trạng thái   │ Ghi chú  │
├──────────────────┼─────────────┼─────────────┼──────────┼──────────┤
│ Shipper          │ ABC Co Ltd  │ ABC Co Ltd  │ ✓ Khớp   │ ✓ Khớp   │
│ Container No     │ TCLU123     │ TCLU124     │ ✗ Khác   │ ⚠️ Sai  │
│ Port of Loading  │ Shanghai    │ Shanghai    │ ✓ Khớp   │ ✓ Khớp   │
│ Gross Weight     │ 20000 kg    │ 25000 kg    │ ✗ Khác   │ ⚠️ Sai  │
└──────────────────┴─────────────┴─────────────┴──────────┴──────────┘

Màu sắc:
- Hàng xanh nhạt = Trường khớp
- Hàng đỏ nhạt = Trường khác nhau
```

---

## **BƯỚC 8: PHÂN TÍCH AI VÀ KHUYẾN NGHỊ**

### Process:

```
Đầu vào:
- MBL text (1500 ký tự đầu tiên)
- HBL text (1500 ký tự đầu tiên)
- Danh sách 18 trường so sánh (match/khác)

Prompt gửi tới Gemini:
"Bạn là chuyên gia logistics. Hãy phân tích:
1. Tại sao các trường không khớp?
2. Phát hiện lỗi tinh tế (typo, dấu, format)?
3. Đánh giá mức độ nghiêm trọng?
4. Hành động cần làm?
5. Danh sách ưu tiên (từ cao đến thấp)?"

AI trả lại:
- Phân tích chi tiết 2-3 đoạn
- Danh sách từng vấn đề
- Khuyến nghị cụ thể

Ví dụ output:
"
## 🔴 PHÁT HIỆN VẤN ĐỀ:

### 1. Sai số Container (CAO)
- MBL: TCLU1234567
- HBL: TCLU1234568 (chữ số cuối khác 1)
- Nguyên nhân: Lỗi nhập liệu
- Hành động: PHẢI SỬA NGAY

### 2. Trọng lượng khác nhau (CAO)
- MBL: 20,000 kg
- HBL: 25,000 kg
- Nguyên nhân: Có thể thiếu hàng
- Hành động: Kiểm tra kho ngay lập tức

### 3. Port nhập khác chút (THẤP)
- MBL: Shanghai Port
- HBL: Shanghai
- Nguyên nhân: Cách viết khác, nhưng cùng cảng
- Hành động: Có thể bỏ qua
"
```

---

## **BƯỚC 9: HIỂN THỊ DEBUG LOGS**

### Console logs (cho kỹ thuật viên):

```
🚀 START ANALYSIS - 14:30:45
MBL Text Length: 5432
HBL Text Length: 4891

📖 Extracting text from MBL.pdf (245.32 KB)
   📄 PDF has 2 page(s)
   Page 1/2 - Native text: Shipper: ABC Company Ltd...
   ✅ Text extraction complete: 5432 characters total

📖 Extracting text from HBL.pdf (198.15 KB)
   📄 PDF has 1 page(s)
   Page 1/1 - Native text: Consignee: XYZ Corp...
   ⚠️ Page 1 has little native text (30 chars), running OCR...
   📸 Page 1 rendered to image (1652x2400px)
   🤖 Sending to Gemini Vision for text extraction...
   ✅ Vision OCR complete: 4891 characters

🤖 [MBL] Gọi Gemini API...
   📝 Text length: 5432 characters
   🤖 JSON parsed successfully - 18 fields

🤖 [HBL] Gọi Gemini API...
   📝 Text length: 4891 characters
   🤖 JSON parsed successfully - 18 fields

📊 Comparisons: 18 fields, 16 khớp, 2 khác
   - Accuracy: 88.9%

🤖 [ANALYSIS] Gọi AI để phân tích chi tiết...
   ✅ Analysis complete (1245 characters)

✅ ANALYSIS COMPLETE
💡 Open browser DevTools (F12) to see detailed logs
```

---

## **TÓMLƯỢC QUY TRÌNH (FLOW CHART)**

```
┌─────────────────────────────────────────────────────────────┐
│ 1. KHỞI ĐỘNG: Kiểm tra API Key (env / localStorage / input) │
└────────────────────┬────────────────────────────────────────┘
                     │
┌────────────────────▼────────────────────────────────────────┐
│ 2. UPLOAD: Người dùng kéo thả 2 file MBL & HBL             │
└────────────────────┬────────────────────────────────────────┘
                     │
┌────────────────────▼────────────────────────────────────────┐
│ 3. TRÍCH XUẤT TEXT:                                         │
│    - PDF.js native text extraction                          │
│    - OCR bằng Gemini Vision (nếu PDF là ảnh)              │
│    Result: MBL full text, HBL full text                    │
└────────────────────┬────────────────────────────────────────┘
                     │
┌────────────────────▼────────────────────────────────────────┐
│ 4. TRÍCH XUẤT TRƯỜNG (AI - Gemini):                        │
│    - MBL: 18 trường → JSON object                          │
│    - HBL: 18 trường → JSON object                          │
└────────────────────┬────────────────────────────────────────┘
                     │
┌────────────────────▼────────────────────────────────────────┐
│ 5. SO SÁNH:                                                 │
│    - Lặp từng trường                                        │
│    - Chuẩn hóa & so sánh thông minh                        │
│    - Result: 18 comparisons (match/khác)                   │
└────────────────────┬────────────────────────────────────────┘
                     │
┌────────────────────▼────────────────────────────────────────┐
│ 6. TÍNH TOÁN STATS:                                        │
│    - Tổng trường, Khớp, Khác, Accuracy %                  │
└────────────────────┬────────────────────────────────────────┘
                     │
┌────────────────────▼────────────────────────────────────────┐
│ 7. HIỂN THỊ BẢNG và AI ANALYSIS:                           │
│    - Bảng so sánh (màu xanh/đỏ)                           │
│    - Khuyến nghị từ AI (Gemini)                           │
│    - Debug logs                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## **ĐIỂM MỨC CỤ THỂ:**

| Thành phần | Công nghệ | Mục đích |
|-----------|-----------|---------|
| **API Key** | localStorage / process.env | Lưu & quản lý khóa |
| **PDF Reading** | PDF.js | Đọc file PDF |
| **Text OCR** | Gemini Vision API | Trích text từ ảnh |
| **Field Extraction** | Gemini Flash Model | Trích xuất 18 trường |
| **Comparison Logic** | Regex + String Matching | So sánh thông minh |
| **Analysis** | Gemini Flash Model | Phân tích & khuyến nghị |
| **UI** | HTML/CSS/JavaScript | Hiển thị kết quả |
| **Debug Console** | JavaScript console capture | Ghi log chi tiết |

---

## **DANH SÁCH 18 TRƯỜNG ĐƯỢC TRÍCH XUẤT:**

1. **Shipper** - Người gửi hàng
2. **Consignee** - Người nhận hàng
3. **Notify Party** - Bên thông báo
4. **Pre-carriage by** - Vận chuyển tiền vận
5. **Place of Receipt** - Nơi nhận hàng
6. **Vessel** - Tên tàu
7. **Voy. No** - Số chuyến
8. **Port of Loading** - Cảng xuất
9. **Port of Discharge** - Cảng nhập
10. **Place of Delivery** - Nơi giao hàng
11. **Container No** - Số container
12. **Seal No** - Số seal
13. **Marks & Numbers** - Dấu hiệu & số hiệu
14. **Number of containers or other Pkgs** - Số kiện/container
15. **Description of Packages and goods** - Mô tả hàng hóa
16. **Gross weight** - Cân nặng tổng cộng
17. **Measurement** - Đo lường (CBM)
18. **Total number of container or packages** - Tổng số kiện/container

---

## **LỖI PHỔ BIẾN PHÁT HIỆN:**

| Loại Lỗi | Ví dụ | Mức Độ | Hành Động |
|----------|-------|--------|----------|
| Typo Container | TCLU123456**7** vs TCLU123456**8** | 🔴 CAO | Sửa ngay |
| Trọng lượng khác | 20,000 kg vs 25,000 kg | 🔴 CAO | Kiểm tra kho |
| Port viết tắt | "Shanghai port" vs "Shanghai" | 🟡 TRUNG | Có thể chấp nhận |
| Ngôn ngữ khác | "John Smith" vs "Giơng Smit" | 🔴 CAO | Xác nhận tên |
| Format tiền | "$100" vs "100 USD" | 🟡 TRUNG | Chuẩn hóa |
| Dấu chấm phẩy | "A, B, C" vs "A; B; C" | 🟢 THẤP | Bỏ qua |

---

**Đó là toàn bộ quy trình! 🎯 Từ upload PDF → text extraction → field parsing → comparison → AI analysis → kết quả chi tiết**

Tài liệu được tạo: **QUY_TRINH_PHAN_TICH_BILL.md**
