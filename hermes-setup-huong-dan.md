# Hướng dẫn setup Hermes-Agent: Codex + Gemini (không API key) & Discord (kèm Telegram)

> File này để **bạn tự làm theo từng bước**. Mọi lệnh chạy trong **PowerShell**.
> Brain chính = **Codex** (đăng nhập ChatGPT). Backend phụ "kiểu agy" = **Gemini qua đăng nhập Google**. Bật **Discord** song song **Telegram**.
> **Không cần sửa code** — tất cả tính năng đã có sẵn trong repo, đây chỉ là cấu hình.

---

## 0. Tình trạng hiện tại (ĐÃ XONG — không cần làm lại)

- ✅ `uv` đã cài.
- ✅ venv Python **3.11.15** tại `C:\Users\kle23\OneDrive\Documents\hermes-agent\venv`.
- ✅ Fork đã cài vào venv kèm `discord.py` + `python-telegram-bot`.
- ✅ Node 22 portable đã tự cài vào `C:\Users\kle23\AppData\Local\hermes\node\`.
- ✅ `HERMES_HOME` = **`C:\Users\kle23\AppData\Local\hermes`** (đã có `.env`).

**Còn lại (file này hướng dẫn):** đăng nhập Codex → đặt Codex làm model → đăng nhập Gemini → (tùy chọn) fallback → tạo Discord bot → chạy gateway → kiểm tra.

---

## ⭐ Mỗi lần mở terminal mới, làm 2 dòng này TRƯỚC

```powershell
cd "C:\Users\kle23\OneDrive\Documents\hermes-agent"
.\venv\Scripts\Activate.ps1
```

Sau khi activate, dấu nhắc sẽ có tiền tố `(venv)`. Từ đó gõ `hermes ...` là chạy đúng bản trong venv.

> Nếu dòng `Activate.ps1` báo lỗi "running scripts is disabled", chạy 1 lần:
> ```powershell
> Set-ExecutionPolicy -Scope CurrentUser -ExecutionPolicy RemoteSigned
> ```
> rồi activate lại. Hoặc bỏ qua activate và gọi thẳng `.\venv\Scripts\hermes.exe ...` ở mọi lệnh.

---

## 1. Đăng nhập Codex (brain chính, KHÔNG cần API key)

### 1.1 Chạy device-code login
```powershell
hermes auth add openai-codex
```
- Màn hình sẽ in **một mã (code) + một URL** (hoặc tự mở trình duyệt).
- Mở URL, đăng nhập tài khoản **ChatGPT/Codex** của bạn, nhập mã, bấm xác nhận.
- **Kết quả mong đợi:** dòng cuối in `Added openai-codex OAuth credential #1: "..."`.

### 1.2 Kiểm tra đã lưu credential
```powershell
hermes auth list
hermes auth status openai-codex
```
- **Mong đợi:** `auth list` có khối `openai-codex (1 credentials)`; `auth status` in `openai-codex: logged in`.

### 1.3 Đặt Codex làm model mặc định
**Cách A (khuyến nghị, có menu chọn model bạn được phép dùng):**
```powershell
hermes model
```
→ chọn provider **`openai-codex`** → chọn model Codex (vd `gpt-5.x-codex`).

**Cách B (gõ thẳng, không cần menu):**
```powershell
hermes config set model.provider openai-codex
hermes config set model.default gpt-5.3-codex
```
*(Đổi `gpt-5.3-codex` thành model bạn thực sự có; nếu không chắc, dùng Cách A để xem danh sách.)*

### 1.4 Thử nhanh
```powershell
hermes
```
Trong phiên chat gõ: `bạn đang chạy model nào?` → Enter. Xong gõ `/exit`.
- **Mong đợi:** có câu trả lời, không báo lỗi auth → Codex đang chạy.

---

## 2. Đăng nhập Google/Gemini (backend phụ "kiểu agy", KHÔNG cần API key)

> `agy` (Antigravity CLI) **không có API mạng** để làm backend. Cách đúng để "đăng nhập 1 lần rồi dùng như backend" chính là provider `google-gemini-cli` — cùng tài khoản Google, cùng dòng model Gemini mà agy dùng.

### 2.1 Login
```powershell
hermes auth add google-gemini-cli
```
- Mở trình duyệt, đăng nhập **Google**.
- **Mong đợi:** `Added google-gemini-cli OAuth credential #1: "<email-của-bạn>"`.

### 2.2 Kiểm tra
```powershell
hermes auth list
hermes auth status google-gemini-cli
```
- **Mong đợi:** có khối `google-gemini-cli (1 credentials)`; status `logged in`.

### 2.3 Thử chuyển model trong CLI
```powershell
hermes
```
Trong phiên: gõ `/model` → chọn `google-gemini-cli` + 1 model Gemini → hỏi `test gemini` → có trả lời → gõ `/model` chọn lại `openai-codex` → `/exit`.
- **Mong đợi:** cả hai provider đều trả lời được.

### 2.4 (Tùy chọn) Đặt Gemini làm fallback tự động cho Codex
Khi Codex hết hạn mức (429/usage-limit), Hermes tự chuyển sang Gemini trong lượt đó.

**Cách A:**
```powershell
hermes fallback add
```
→ chọn `google-gemini-cli` + model Gemini.

**Cách B (sửa file):** mở `C:\Users\kle23\AppData\Local\hermes\config.yaml`, thêm vào cuối (khóa top-level):
```yaml
fallback_providers:
  - provider: google-gemini-cli
    model: gemini-2.5-pro
```
*(Đổi `gemini-2.5-pro` thành model bạn có.)*

Kiểm tra:
```powershell
hermes fallback list
```

---

## 3. (Tùy chọn) Dùng `agy` như một CÔNG CỤ coding-agent

> Hermes vẫn dùng Codex làm brain, nhưng có thể **điều khiển** `agy` để chạy tác vụ code.

### 3.1 Cài + đăng nhập agy
```powershell
irm https://antigravity.google/cli/install.ps1 | iex
agy --version
agy --print "say hi"
```
- Lần đầu `agy --print` sẽ kích hoạt đăng nhập Google (đây là "agy auth" bạn muốn).

### 3.2 Đưa skill vào Hermes (nếu cần)
```powershell
Copy-Item -Recurse "optional-skills\autonomous-ai-agents\antigravity-cli" "C:\Users\kle23\AppData\Local\hermes\skills\antigravity-cli"
```
Sau đó mở `hermes`, gõ `/skills` và kiểm tra có `antigravity-cli`.

### 3.3 Thử
Trong `hermes` gõ: `dùng terminal chạy: agy --print "tóm tắt repo trong 3 gạch đầu dòng"`.

---

## 4. Bật Discord bot (song song Telegram)

### 4.1 Tạo bot trên Discord
1. Vào https://discord.com/developers/applications → **New Application** (đặt tên bất kỳ).
2. Tab **Bot** → bấm **Reset Token** → **Copy** token (chuỗi dài). Giữ bí mật.
3. Vẫn tab **Bot** → bật **MESSAGE CONTENT INTENT** (bắt buộc, để bot đọc được nội dung tin nhắn).
4. Tab **OAuth2 → URL Generator**:
   - **Scopes:** tích `bot` và `applications.commands`.
   - **Bot Permissions:** Send Messages, Read Message History, Create Public Threads, Add Reactions (thêm Connect/Speak nếu muốn voice).
   - Copy URL ở dưới → mở trong trình duyệt → chọn server của bạn → **Authorize** (mời bot vào server).
5. Lấy **User ID của bạn:** trong Discord, vào Settings → Advanced → bật **Developer Mode**. Rồi chuột phải vào tên bạn → **Copy User ID**.

### 4.2 Thêm token vào `.env`
Mở file `C:\Users\kle23\AppData\Local\hermes\.env` bằng Notepad:
```powershell
notepad "C:\Users\kle23\AppData\Local\hermes\.env"
```
Thêm các dòng (thay giá trị trong `<...>`):
```
DISCORD_BOT_TOKEN=<token-bạn-copy-ở-4.1>
DISCORD_ALLOWED_USERS=<user-id-của-bạn>
```
Tùy chọn:
```
DISCORD_HOME_CHANNEL=<channel-id>      # kênh mặc định cho cron/thông báo
```
Lưu file.

> ⚠️ **Bảo mật:** chỉ để `DISCORD_ALLOWED_USERS` là chính bạn. **Đừng** đặt `DISCORD_ALLOW_ALL_USERS=true` (ai cũng điều khiển được agent, kèm quyền chạy terminal).

### 4.3 (Tùy chọn) Giữ Telegram chạy cùng
Nếu muốn dùng cả Telegram, thêm vào cùng file `.env`:
```
TELEGRAM_BOT_TOKEN=<token-từ-@BotFather>
TELEGRAM_ALLOWED_USERS=<telegram-user-id-của-bạn>
```
*(Bỏ qua nếu chưa cần — Discord vẫn chạy độc lập.)*

### 4.4 (Tùy chọn) Tinh chỉnh hành vi Discord
Mở `C:\Users\kle23\AppData\Local\hermes\config.yaml`, thêm khối top-level:
```yaml
discord:
  require_mention: true     # trong kênh server phải @mention bot
  auto_thread: true         # tự mở thread khi được mention
  reactions: true
  history_backfill: true
```

---

## 5. Chạy gateway & kiểm tra

### 5.1 Khởi động gateway
```powershell
hermes gateway
```
- **Mong đợi:** log liệt kê **discord** (và **telegram** nếu đã set token) kết nối thành công, vd `Discord connected as ...`.
- Cửa sổ này phải **để mở** thì bot mới chạy. Dừng bằng `Ctrl+C`.

### 5.2 Test trên Discord
Trong server đã mời bot: **@mention** bot (hoặc nhắn DM) câu `bạn đang chạy model nào?`.
- **Mong đợi:** bot trả lời (do Codex). Nếu bật `auto_thread`, nó trả lời trong thread.

### 5.3 Test trên Telegram (nếu đã set)
Nhắn cho bot Telegram của bạn → **mong đợi:** cùng gateway trả lời.

### 5.4 (Tùy chọn) Chạy nền như dịch vụ
Khi đã ổn, thay vì giữ cửa sổ mở:
```powershell
hermes gateway install     # cài chạy nền
hermes gateway status      # xem trạng thái
hermes gateway stop        # dừng
```

---

## 6. Checklist hoàn tất (chạy lần lượt, xác nhận từng dòng)

```powershell
hermes auth list                      # thấy CẢ openai-codex VÀ google-gemini-cli
hermes auth status openai-codex       # logged in
hermes auth status google-gemini-cli  # logged in
hermes fallback list                  # có google-gemini-cli (nếu làm 2.4)
hermes doctor                         # không còn lỗi auth cho Codex/Gemini
```
- [ ] CLI `hermes` trả lời được; `/model` chuyển Codex ↔ Gemini OK.
- [ ] `hermes gateway` → log có discord (+ telegram nếu set).
- [ ] @mention/DM Discord bot → có phản hồi.

---

## 7. Khắc phục sự cố nhanh

| Triệu chứng | Cách xử lý |
|---|---|
| `hermes` không nhận diện | Chưa activate venv. Chạy lại 2 dòng ở mục ⭐, hoặc gọi `.\venv\Scripts\hermes.exe ...`. |
| `Activate.ps1` bị chặn | `Set-ExecutionPolicy -Scope CurrentUser RemoteSigned` rồi activate lại. |
| Bot Discord không đọc tin nhắn | Chưa bật **MESSAGE CONTENT INTENT** (mục 4.1 bước 3). |
| Gateway không thấy Discord | `DISCORD_BOT_TOKEN` sai/thiếu trong `.env`; kiểm tra `notepad "C:\Users\kle23\AppData\Local\hermes\.env"`. |
| `auth list` báo "re-auth required" | Token hết hạn: chạy lại `hermes auth add openai-codex` (hoặc `google-gemini-cli`). |
| Codex báo hết hạn mức (429) | Làm mục 2.4 để Gemini làm fallback, hoặc `/model` chuyển sang Gemini thủ công. |
| Muốn thêm STT giọng nói | `python -m uv pip install -e ".[voice]" --python ".\venv\Scripts\python.exe"` (trong thư mục repo). |

---

## 8. Ghi nhớ vận hành

- **Cập nhật fork về sau:** `git pull` trong repo, rồi cài lại deps nếu có thay đổi:
  ```powershell
  cd "C:\Users\kle23\OneDrive\Documents\hermes-agent"
  git pull
  python -m uv pip install -e ".[all,messaging]" --python ".\venv\Scripts\python.exe"
  ```
- **File quan trọng:**
  - Secrets/token: `C:\Users\kle23\AppData\Local\hermes\.env`
  - Cấu hình: `C:\Users\kle23\AppData\Local\hermes\config.yaml`
  - Credential OAuth: tự lưu trong `C:\Users\kle23\AppData\Local\hermes\` (Hermes quản lý).
- **`agy` không phải backend LLM** — đừng cố trỏ provider vào `agy agentapi`. Backend "đăng nhập Google" đúng nghĩa là `google-gemini-cli`.

---

*Hết. Làm theo mục 1 → 2 → (3) → 4 → 5 → 6. Gặp lỗi ở bước nào cứ báo, kèm dòng log lỗi.*
