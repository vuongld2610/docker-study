Giải pháp 2 hoàn toàn có thể đóng gói gọn gàng thành một Docker image và đây cũng là cách làm chuẩn nhất. Tuy nhiên, vì đây là một kịch bản tùy biến riêng cho FPT Play, hiện **không có image chính thức (official)** có sẵn trên Docker Hub mà bạn chỉ cần kéo về là chạy ngay.

Thay vào đó, bạn sẽ dùng một **Base Image có sẵn Python + Playwright** (do Microsoft cung cấp), sau đó thêm file script Python của bạn vào.

Dưới đây là cách triển khai chi tiết bằng Docker trên Ubuntu Server để hệ thống tự động chạy:

### Bước 1: Chuẩn bị file Script Python (`fetch_token.py`)

Tạo một thư mục trên Ubuntu (ví dụ: `/opt/fpt-scraper`), tạo file `fetch_token.py` có nội dung tự động mở Chrome, load cookie, bắt link `.m3u8` và ghi đè vào file playlist:

```python
import asyncio
import json
import os
import re
from playwright.async_api import async_playwright

# Đường dẫn file playlist và cookie bên trong container
PLAYLIST_PATH = "/output/fptplay.m3u"
COOKIE_PATH = "/config/cookies.json"
TARGET_URL = "https://fptplay.vn/xem-truyen-hinh/vtv3-hd" # Bạn có thể đổi sang VTV6 tùy gói cước

async def main():
async with async_playwright() as p:
# Khởi chạy trình duyệt ẩn danh
browser = await p.chromium.launch(headless=True)
context = await browser.new_context()

# Nạp cookie nếu có để bỏ qua bước đăng nhập thủ công
if os.path.exists(COOKIE_PATH):
with open(COOKIE_PATH, "r") as f:
cookies = json.load(f)
await context.add_cookies(cookies)

page = await context.new_page()
m3u8_url = None

# Bắt các request mạng ngầm
async def handle_request(request):
nonlocal m3u8_url
url = request.url
if "vips-livecdn.fptplay.net" in url and "index.m3u8?token=" in url:
m3u8_url = url

page.on("request", lambda req: asyncio.create_task(handle_request(req)))

# Truy cập trang phát video
try:
print(" đang truy cập FPT Play để lấy token...")
await page.goto(TARGET_URL, timeout=60000)
# Chờ 10 giây để trình duyệt load luồng video và sinh token
await page.wait_for_timeout(10000)
except Exception as e:
print(f"Lỗi truy cập: {e}")

await browser.close()

# Nếu tóm được link, tiến hành ghi đè file .m3u
if m3u8_url:
print(f" Tìm thấy URL mới: {m3u8_url}")
playlist_content = f"#EXTM3U\n#EXTINF:-1 tvg-id=\"VTV3\" tvg-name=\"VTV3 HD\",VTV3 HD\n{m3u8_url}\n"

os.makedirs(os.path.dirname(PLAYLIST_PATH), exist_ok=True)
with open(PLAYLIST_PATH, "w") as f:
f.write(playlist_content)
print(" Đã cập nhật file playlist thành công!")
else:
print(" Không tìm thấy link m3u8 kèm token. Hãy kiểm tra lại Cookie.")

if __name__ == "__main__":
asyncio.run(main())

```

### Bước 2: Tạo `Dockerfile` để tự build Image

Nằm cùng thư mục với file Python trên, bạn tạo một file tên là `Dockerfile`:

```dockerfile
# Sử dụng bản image chuẩn tích hợp sẵn Python và các thư viện cần thiết của Playwright
FROM mcr.microsoft.com/playwright/python:v1.44.0-jammy

WORKDIR /app

# Copy script vào container
COPY fetch_token.py .

# Cài đặt thư viện playwright cho python
RUN pip install playwright

# Chạy script khi container khởi động
CMD ["python", "fetch_token.py"]

```

**Cách build thành image cục bộ:**

```bash
docker build -t fpt-token-scraper:latest .

```

### Bước 3: Cách vận hành 24/7 bằng Docker Compose kết hợp Cronjob

Để TVHeadend đọc được file, bạn cần tạo trước một file `cookies.json` (bằng cách dùng extension như *EditThisCookie* trên Chrome máy tính, export dạng JSON rồi lưu vào `/opt/fpt-scraper/config/cookies.json`).

Sau đó, cấu hình file `docker-compose.yml` chung cho cả TVHeadend và con Bot cào này:

```yaml
version: '3'

services:
tvheadend:
image: lscr.io/linuxserver/tvheadend:latest
container_name: tvheadend
restart: unless-stopped
environment:
- PUID=1000
- PGID=1000
- TZ=Asia/Ho_Chi_Minh
ports:
- 9981:9981
- 9982:9982
volumes:
- /var/lib/tvheadend/config:/config
# Mount chung thư mục chứa file playlist m3u với con bot
- /opt/fpt-scraper/output:/playlist

fpt-scraper:
image: fpt-token-scraper:latest
container_name: fpt-scraper
profiles: ["manual"] # Để container này không tự chạy liên tục mà chỉ chạy khi gọi
volumes:
- /opt/fpt-scraper/config:/config
- /opt/fpt-scraper/output:/output

```

#### Tự động hóa gia hạn token (Đặt Cronjob trên Ubuntu Host):

Vì link có thời hạn khoảng 2 ngày, bạn chỉ cần đặt một lệnh `cron` trên Ubuntu để **cứ mỗi đêm, Docker lại dựng con Bot lên cào link mới, ghi đè vào file chung rồi tự tắt đi**:

Mở cronjob bằng lệnh: `crontab -e` và thêm dòng này vào:

```bash
0 3 * * * docker compose -f /opt/fpt-scraper/docker-compose.yml run --rm fpt-scraper

```

*(Cứ 3 giờ sáng mỗi ngày, lệnh này sẽ đánh thức container `fpt-scraper` dậy, cào token mới đè vào `/opt/fpt-scraper/output/fptplay.m3u` rồi tự xóa container đó đi để giải phóng RAM).*

Trong TVHeadend, bạn chỉ cần tạo một `IPTV Automatic Network` và trỏ đường dẫn file đầu vào là `/playlist/fptplay.m3u` là xong. Mọi thứ chạy ngầm và tự động gia hạn hoàn toàn.
