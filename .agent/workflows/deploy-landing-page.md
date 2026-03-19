---
description: Quy trình tạo landing page webinar, tích hợp Google Sheet, deploy lên Vercel qua GitHub
---

# 🚀 Deploy Landing Page Webinar lên Vercel

## Tổng quan

Quy trình đầy đủ từ tạo landing page, tích hợp form đăng ký → Google Sheet, popup sau đăng ký, OG thumbnail, đến deploy lên Vercel tự động qua GitHub + custom domain.

---

## 1. Cấu trúc dự án

```
📁 Landing Page/
├── index.html        # Trang chính
├── style.css         # CSS styling
├── og-image.png      # Ảnh thumbnail khi share link (1200x630)
├── ảnh *.png         # Ảnh minh họa
└── content.txt       # Nội dung gốc
```

---

## 2. Tích hợp Form → Google Sheet

### Bước 1: Tạo Google Apps Script

Mở Google Sheet → **Extensions → Apps Script** → thêm hàm `doPost`:

```javascript
function doPost(e) {
  var sheet = SpreadsheetApp.openById('SHEET_ID').getSheetByName('Sheet1');
  sheet.appendRow([
    e.parameter.timestamp,
    e.parameter.name,
    e.parameter.email,
    e.parameter.phone,
    e.parameter.link_utm
  ]);
  return ContentService.createTextOutput("OK");
}
```

> ⚠️ Thay `SHEET_ID` bằng ID thật. Nếu đã có `doGet`, thêm `doPost` ở cuối file (KHÔNG tạo 2 hàm `doPost`).

### Bước 2: Deploy Apps Script

**Deploy → New deployment → Web app** → Execute as: Me → Who has access: Anyone → Copy URL exec.

### Bước 3: Code gửi dữ liệu (hidden iframe bypass CORS)

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
        setTimeout(() => { iframe.remove(); hiddenForm.remove(); resolve(); }, 3000);
    });
}
```

> ⚠️ KHÔNG dùng `fetch` + `no-cors` + `application/json` → bị chặn. Dùng **hidden iframe** là chuẩn.

### Bước 4: Gửi kèm UTM tracking

```javascript
const linkUtm = window.location.href;
await sendToGoogleSheet({ name, email, phone, timestamp: new Date().toLocaleString('vi-VN'), link_utm: linkUtm });
```

### Nhiều form trên 1 trang

```javascript
document.querySelectorAll('.webinar-form').forEach(form => {
    form.addEventListener('submit', async function(e) { ... });
});
```

---

## 3. OG Thumbnail (Facebook / Zalo)

### Meta tags (BẮT BUỘC dùng URL tuyệt đối):

```html
<meta property="og:type" content="website">
<meta property="og:url" content="https://YOUR_DOMAIN/">
<meta property="og:title" content="Tiêu đề">
<meta property="og:description" content="Mô tả">
<meta property="og:image" content="https://YOUR_DOMAIN/og-image.png">
<meta property="og:image:width" content="1200">
<meta property="og:image:height" content="630">
<meta property="og:locale" content="vi_VN">
```

### Kích thước ảnh tối ưu:
- **Facebook**: 1200x630 px (tỷ lệ 1.91:1)
- **Zalo**: Tương tự Facebook

### Buộc cập nhật thumbnail:
- **Facebook**: Vào [developers.facebook.com/tools/debug](https://developers.facebook.com/tools/debug) → dán link → bấm **Scrape Again**
- **Zalo**: Đợi vài giờ hoặc gửi link mới
---

## 4. Countdown Timer (Tạo urgency)

### HTML (đặt ngay sau `</form>` trong form-wrapper):

```html
<div class="form-countdown">
    <p class="countdown-label">⚡ Ưu đãi kết thúc sau:</p>
    <div class="countdown-timer" id="countdown-top">
        <div class="countdown-box"><span class="countdown-num" id="ct-min-top">10</span><span class="countdown-unit">Phút</span></div>
        <div class="countdown-sep">:</div>
        <div class="countdown-box"><span class="countdown-num" id="ct-sec-top">00</span><span class="countdown-unit">Giây</span></div>
    </div>
</div>
```

### JavaScript (dùng localStorage để giữ countdown khi reload):

```javascript
(function() {
    const COUNTDOWN_MINUTES = 10;
    const storageKey = 'webinar_countdown_end';
    let endTime = localStorage.getItem(storageKey);
    if (!endTime || Number(endTime) < Date.now()) {
        endTime = Date.now() + COUNTDOWN_MINUTES * 60 * 1000;
        localStorage.setItem(storageKey, endTime);
    }
    endTime = Number(endTime);

    function updateCountdown() {
        const diff = Math.max(0, endTime - Date.now());
        const mins = Math.floor(diff / 60000);
        const secs = Math.floor((diff % 60000) / 1000);
        // Cập nhật cả 2 timer (top + bottom)
        ['top', 'bottom'].forEach(pos => {
            const minEl = document.getElementById('ct-min-' + pos);
            const secEl = document.getElementById('ct-sec-' + pos);
            if (minEl) minEl.textContent = String(mins).padStart(2, '0');
            if (secEl) secEl.textContent = String(secs).padStart(2, '0');
        });
        if (diff > 0) requestAnimationFrame(() => setTimeout(updateCountdown, 1000));
    }
    updateCountdown();
})();
```

> Dưới 2 phút → chuyển đỏ + nhấp nháy (class `countdown-urgent`).

---

## 5. Cài Git trên máy

```powershell
# Cài qua winget (tự động):
winget install --id Git.Git -e --source winget --accept-package-agreements --accept-source-agreements

# Git được cài ở D:\Git (tùy máy). Tìm bằng:
Get-ChildItem -Path "C:\","D:\" -Filter "git.exe" -Recurse -ErrorAction SilentlyContinue -Depth 4 | Select-Object -First 3 -ExpandProperty FullName
```

> ⚠️ Sau khi cài, cần đóng mở lại VS Code hoặc dùng full path `D:\Git\cmd\git.exe`.

---

## 6. Push code lên GitHub

### Lần đầu (init repo):

```powershell
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

> Push xong → Vercel **tự động deploy lại** trong ~30 giây.

---

## 7. Deploy lên Vercel

1. Vào **[vercel.com/new](https://vercel.com/new)** → đăng nhập GitHub
2. **Import** → chọn repo
3. Framework Preset: **Other**
4. Bấm **Deploy**

---

## 8. Custom Domain (Vercel + Hostinger)

### Bước 1: Thêm domain trên Vercel
- Vào project → **Settings → Domains**
- Nhập `subdomain.yourdomain.com` → **Add**
- Chọn **Connect to an environment** → **Production** → **Save**
- Vercel hiện thông tin CNAME record

### Bước 2: Thêm CNAME trên Hostinger
- Vào **DNS / Máy chủ tên miền**
- **Thêm bản ghi**:

| Loại | Tên | Trỏ tới | TTL |
|------|-----|---------|-----|
| CNAME | `subdomain` | `xxx.vercel-dns-xxx.com` (copy từ Vercel) | 14400 |

- Đợi **5–30 phút** → Vercel hiện ✅ Valid Configuration
- Vercel tự cấp SSL (https) miễn phí

> ⚠️ Nếu Hostinger dùng dns-parking: Chỉnh sửa DNS → chọn "Máy chủ tên miền Hostinger" trước.

---

## 9. Meta Pixel (Facebook Ads Tracking)

Dán mã pixel vào `<head>`, trước `</head>`:

```html
<!-- Meta Pixel Code -->
<script>
!function(f,b,e,v,n,t,s)
{if(f.fbq)return;n=f.fbq=function(){n.callMethod?
n.callMethod.apply(n,arguments):n.queue.push(arguments)};
if(!f._fbq)f._fbq=n;n.push=n;n.loaded=!0;n.version='2.0';
n.queue=[];t=b.createElement(e);t.async=!0;
t.src=v;s=b.getElementsByTagName(e)[0];
s.parentNode.insertBefore(t,s)}(window, document,'script',
'https://connect.facebook.net/en_US/fbevents.js');
fbq('init', 'YOUR_PIXEL_ID');
fbq('track', 'PageView');
</script>
<noscript><img height="1" width="1" style="display:none"
src="https://www.facebook.com/tr?id=YOUR_PIXEL_ID&ev=PageView&noscript=1"
/></noscript>
<!-- End Meta Pixel Code -->
```

> Thay `YOUR_PIXEL_ID` bằng ID pixel thật. Kiểm tra bằng extension **Meta Pixel Helper** trên Chrome.

---

## 10. Mẫu bài viết bán hàng

```
🤖 Bạn vẫn đang làm mọi thứ bằng tay?

Báo cáo. Phân tích. Theo dõi doanh thu. Trả lời tin nhắn...
Mỗi ngày mất 3–4 tiếng cho việc lặp đi lặp lại → lãng phí 90 tiếng/tháng.

💡 Chỉ cần 1 Trợ Lý AI – tất cả chạy tự động 24/7 trên Zalo hoặc Telegram.

🎓 WEBINAR MIỄN PHÍ: SIÊU TRỢ LÍ AI CÁ NHÂN

✅ Xây dựng Trợ lý AI từ A–Z (không cần biết code)
✅ Demo trực tiếp: AI tự động báo cáo, phân tích, bắt trend
✅ Tự động hóa 80% công việc chỉ với 1 hệ thống

🕐 19:45 – 22:00 | Ngày 25/03/2026
📍 Online qua Zoom
💰 Hoàn toàn MIỄN PHÍ

👉 ĐĂNG KÝ NGAY: https://sieutroly.trungcaha.click

⚠️ Số lượng có hạn – Đừng bỏ lỡ!
```

> Dùng cho: Facebook Ads, post fanpage, gửi Zalo, Messenger.

---

## 11. Lưu ý quan trọng

| Vấn đề | Giải pháp |
|--------|-----------|
| Git chưa nhận trong VS Code | Đóng VS Code mở lại, hoặc dùng full path `D:\Git\cmd\git.exe` |
| Form không gửi data | Dùng hidden iframe, KHÔNG dùng fetch + no-cors |
| OG image không hiện | Phải dùng URL tuyệt đối `https://...` |
| Apps Script đã có code cũ | Thêm `doPost` cuối file, KHÔNG tạo 2 hàm cùng tên |
| Nhiều form trên 1 trang | Dùng `querySelectorAll().forEach()` |
| FB/Zalo không cập nhật thumbnail | FB: Scrape Again tại debug tool. Zalo: đợi vài giờ |
| DNS chưa nhận | Đợi 5-30 phút, kiểm tra CNAME đúng chưa |
| Meta Pixel không track | Kiểm tra bằng Meta Pixel Helper extension |
