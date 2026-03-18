---
description: Quy trình tạo landing page webinar, tích hợp Google Sheet, deploy lên Vercel qua GitHub
---

# 🚀 Deploy Landing Page Webinar lên Vercel

## Tổng quan

Quy trình đầy đủ từ tạo landing page, tích hợp form đăng ký → Google Sheet, popup sau đăng ký, đến deploy lên Vercel tự động qua GitHub.

---

## 1. Cấu trúc dự án

```
📁 Landing Page/
├── index.html        # Trang chính
├── style.css         # CSS styling
├── og-image.png      # Ảnh thumbnail khi share link
├── ảnh 4.png         # Ảnh minh họa
├── ảnh 5.png
├── ảnh 6.png
└── content.txt       # Nội dung gốc
```

---

## 2. Tích hợp Form → Google Sheet

### Bước 1: Tạo Google Apps Script

Mở Google Sheet → **Extensions → Apps Script** → dán code:

```javascript
function doPost(e) {
  var sheet = SpreadsheetApp.openById('SHEET_ID').getSheetByName('Sheet1');
  sheet.appendRow([
    e.parameter.timestamp,
    e.parameter.name,
    e.parameter.email,
    e.parameter.phone
  ]);
  return ContentService.createTextOutput("OK");
}
```

> ⚠️ Thay `SHEET_ID` bằng ID thật (lấy từ URL sheet). Nếu Apps Script đã có code cũ (`doGet`), thêm `doPost` ở cuối file.

### Bước 2: Deploy Apps Script

- **Deploy → New deployment → Web app**
- Execute as: **Me**
- Who has access: **Anyone**
- Copy URL exec

### Bước 3: Code gửi dữ liệu (trong index.html)

Dùng **hidden iframe** để bypass CORS 100%:

```javascript
const GOOGLE_SCRIPT_URL = 'https://script.google.com/macros/s/YOUR_URL/exec';

function sendToGoogleSheet(data) {
    return new Promise((resolve) => {
        const iframeName = 'hidden-iframe-' + Date.now();
        const iframe = document.createElement('iframe');
        iframe.name = iframeName;
        iframe.style.display = 'none';
        document.body.appendChild(iframe);

        const hiddenForm = document.createElement('form');
        hiddenForm.method = 'POST';
        hiddenForm.action = GOOGLE_SCRIPT_URL;
        hiddenForm.target = iframeName;
        hiddenForm.style.display = 'none';

        for (const [key, value] of Object.entries(data)) {
            const input = document.createElement('input');
            input.type = 'hidden';
            input.name = key;
            input.value = value;
            hiddenForm.appendChild(input);
        }

        document.body.appendChild(hiddenForm);
        hiddenForm.submit();

        setTimeout(() => {
            iframe.remove();
            hiddenForm.remove();
            resolve();
        }, 3000);
    });
}
```

> ⚠️ KHÔNG dùng `fetch` với `mode: 'no-cors'` + `Content-Type: application/json` → sẽ bị chặn.
> ⚠️ KHÔNG dùng `fetch` với `FormData` trực tiếp → Google redirect gây lỗi CORS.
> ✅ Dùng **hidden iframe + form thật** → bypass CORS hoàn toàn.

---

## 3. Popup sau đăng ký

Popup hiện sau khi submit form, chứa:
- Thanh sọc đỏ-trắng trên cùng
- Tiêu đề "CÒN 1 BƯỚC NỮA..."
- Nút vào nhóm Zalo
- Đồng hồ đếm ngược

```javascript
// Hiện popup
popupOverlay.classList.add('active');
document.body.style.overflow = 'hidden';
startCountdown();
```

---

## 4. OG Meta Tags (Thumbnail khi share link)

Thêm vào `<head>`:

```html
<meta property="og:type" content="website">
<meta property="og:title" content="Tiêu đề trang">
<meta property="og:description" content="Mô tả">
<meta property="og:image" content="https://YOUR_DOMAIN/og-image.png">
<meta property="og:image:width" content="1200">
<meta property="og:image:height" content="630">
```

> ⚠️ `og:image` **BẮT BUỘC** dùng URL tuyệt đối (https://...), không dùng relative path.

---

## 5. Push code lên GitHub

### Lần đầu (init repo):

```powershell
# Git được cài ở D:\Git (tùy máy)
D:\Git\cmd\git.exe init
D:\Git\cmd\git.exe config user.email "your@email.com"
D:\Git\cmd\git.exe config user.name "your-username"
D:\Git\cmd\git.exe add .
D:\Git\cmd\git.exe commit -m "Landing page Webinar AI"
D:\Git\cmd\git.exe branch -M main
D:\Git\cmd\git.exe remote add origin https://github.com/USERNAME/REPO.git
D:\Git\cmd\git.exe push -u origin main
```

### Các lần sau (cập nhật code):

// turbo-all

```powershell
D:\Git\cmd\git.exe add .
D:\Git\cmd\git.exe commit -m "Cập nhật landing page"
D:\Git\cmd\git.exe push
```

> Sau khi push, Vercel sẽ **tự động deploy lại** (nếu đã kết nối).

---

## 6. Deploy lên Vercel

1. Vào **[vercel.com/new](https://vercel.com/new)** → đăng nhập GitHub
2. **Import** → chọn repo
3. Framework Preset: **Other**
4. Bấm **Deploy**
5. Sau deploy: cập nhật `og:image` URL thành domain Vercel

---

## 7. Lưu ý quan trọng

| Vấn đề | Giải pháp |
|--------|-----------|
| Git chưa nhận trong VS Code | Đóng VS Code, mở lại. Hoặc dùng full path `D:\Git\cmd\git.exe` |
| Form không gửi được data | Dùng hidden iframe, KHÔNG dùng fetch + no-cors |
| OG image không hiện | Phải dùng URL tuyệt đối (https://...) |
| Apps Script đã có code cũ | Thêm `doPost` ở cuối file, KHÔNG tạo hàm doPost thứ 2 |
| Nhiều form trên 1 trang | Dùng `querySelectorAll('.webinar-form').forEach(...)` |
