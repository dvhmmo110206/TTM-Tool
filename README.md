# TTM-Tool — Hệ thống Tự động hóa Đa Nền tảng

Ứng dụng Desktop Windows (PySide6 + Asyncio + Multiprocessing + Patchright/Playwright) tích hợp hạ tầng dùng chung cho nhiều nền tảng (**MiraiMind**, **Mochi Chat**,...) và đồng bộ dữ liệu đám mây qua **Cloudflare Worker & D1 Database**.

- **Quy chuẩn kiến trúc & Phát triển (AI Agents & Developers):** [.agent/AGENTS.md](.agent/AGENTS.md) và thư mục [.agent/references/](.agent/references/)
- **Hướng dẫn build và phát hành tự động cập nhật:** [UPDATE_SETUP_VI.md](UPDATE_SETUP_VI.md)
- **Phân tích kỹ thuật Mochi Chat:** [MOCHICHAT_ANALYSIS.md](MOCHICHAT_ANALYSIS.md)

---

## 1. Khởi chạy & Kiểm thử

### Chạy giao diện Desktop
Giao diện sử dụng `PySide6` và chạy trực tiếp như một ứng dụng Windows độc lập:

```powershell
# Sử dụng uv (khuyến nghị)
uv run python app.py

# Hoặc sử dụng pip truyền thống
python -m pip install -r requirements-gui.txt
python app.py
```

Trước khi cửa sổ chính mở, ứng dụng tự động thực hiện:
1. Nâng cấp schema cấu hình (`core/config/runtime_settings_migration.py`).
2. Kiểm tra bản cập nhật mới và xác thực license qua `/api/auth/activate`. Chỉ license key hợp lệ được lưu cục bộ trong `QSettings`.

### Chạy bộ kiểm thử (Unit & Integration Tests)
```powershell
uv run pytest
# Hoặc chạy rút gọn
uv run pytest -q
```

---

## 2. Kiến trúc Hệ thống & Cấu trúc Thư mục

Dự án tuân thủ nguyên tắc **Hạ tầng độc lập 100% (Platform-Agnostic)**: các gói hạ tầng dùng chung tuyệt đối không chứa logic riêng của từng nền tảng, mọi phân tách nghiệp vụ được định danh tường minh qua `platform_id` và `owner_id`.

```text
TTM-Tool/
├── app.py                          # Entry point khởi chạy ứng dụng Desktop
├── core/                           # Tầng hạ tầng dùng chung & Nghiệp vụ nền tảng
│   ├── auth/                       # Xác thực SSO dùng chung (Google OAuth: google.py)
│   ├── browser/                    # CentralBrowserManager (1 tiến trình Chromium duy nhất)
│   ├── captcha/                    # CaptchaBroker & CaptchaClient giải Geetest/Captcha
│   ├── config/                     # Cấu hình định kiểu mạnh (RuntimeConfig, Constants, Migration)
│   ├── mail/                       # Các provider email tạm & dịch vụ mail (tempmail_sdk, mail_tm, catchmail, gmailvip_vn)
│   ├── proxy/                      # ProxyPoolBroker & ProxyPoolClient điều phối Proxy tĩnh/xoay
│   ├── platforms/                  # Các nền tảng nghiệp vụ độc lập
│   │   ├── miraimind/              # Nền tảng MiraiMind (fetcher, photon_chat, 5 job runners)
│   │   └── mochichat/              # Nền tảng Mochi Chat (api, account_loader, invite_referral runner)
│   ├── job_config.py               # Cấu hình khởi chạy job process
│   └── job_worker.py               # Worker thực thi job trong tiến trình con
├── gui/                            # Giao diện người dùng PySide6
│   ├── controllers/                # JobController điều phối tiến trình worker và IPC queue
│   ├── pages/                      # Các trang chức năng (Dashboard, MiraiMind, Mochi Chat, Proxy, Settings,...)
│   ├── widgets/                    # Bộ UI Component chuẩn hóa (LineEdit, ToggleSwitch, DataTable, Buttons,...)
│   ├── job_lifecycle_service.py    # JobLifecycleManager — Quản lý tập trung vòng đời Job toàn hệ thống
│   ├── main_window.py              # Cửa sổ chính và điều hướng AppWorkspaceStack
│   └── theme.py                    # Hệ màu THEME_PALETTE và QSS toàn cục
├── packages/proxy_core/            # Thư viện lõi chuẩn hóa ProxyModel và xoay IP
├── cloudflare-worker/              # Backend từ xa trên Cloudflare Worker & D1 Database
└── dev_tests/                      # Bộ kiểm thử tự động (pytest)
```

---

## 3. Các Phân hệ Hạ tầng Trọng tâm

### 3.1. Quản lý Cấu hình Tập trung (`core/config/runtime_config.py`)
Toàn bộ cài đặt được gom nhóm vào các `@dataclass` định kiểu mạnh và đồng bộ hai chiều với `QSettings`:
- **`CommonConfig`**: Cấu hình hạ tầng chung (`headless`, `google_login_via_proxy`, `auto_check_update`, `enable_cloud_lifecycle_refresh`, API key và thông số xoay proxy).
- **`MiraimindConfig`**: Cấu hình riêng của MiraiMind (luồng tài khoản/job/proxy, số lần thất bại liên tiếp tối đa, rate limit 424, mail provider, captcha provider, số bình luận/acc).
- **`MochiChatConfig`**: Cấu hình riêng của Mochi Chat (luồng tài khoản/job/proxy, mail provider & API key).
- **`RuntimeConfig`**: Container gốc tổng hợp 3 cấu hình trên thông qua `RuntimeConfig.from_settings(settings)` và `save_to_settings(settings)`.

### 3.2. Quản lý Vòng đời Job & Đồng bộ Dashboard (`JobLifecycleManager`)
Mọi nền tảng (`miraimind`, `mochichat`,...) đều đi qua duy nhất [`gui/job_lifecycle_service.py`](gui/job_lifecycle_service.py):
- **Tầng 1 — Cập nhật In-Memory tức thì (0ms):** Độc quyền gọi `DashboardPage.upsert_job`, cập nhật tiến độ (`record_progress`), trạng thái (`record_status`) và tính toán lại ngay các thẻ KPI trên giao diện mà không gây trễ UI.
- **Đồng bộ Cloud DB bất đồng bộ:** Tự động bắn luồng nền gọi `start_job_on_cloud_best_effort` và `finish_job_on_cloud_best_effort` lên Cloudflare Worker.
- **Bảo vệ hạn ngạch Cloudflare Free (`enable_cloud_refresh = False`):** Mặc định tắt việc tự động tải lại toàn bộ Dashboard từ Cloud sau mỗi sự kiện job để tránh làm cạn kiệt giới hạn request/đọc D1 khi chạy nhiều luồng liên tục. Người dùng có thể bấm nút **"Làm mới"** trên Dashboard bất cứ lúc nào cần đồng bộ từ server.

### 3.3. Quản lý Trình duyệt Tập trung (`CentralBrowserManager`)
- Nằm tại [`core/browser/manager.py`](core/browser/manager.py), duy trì **duy nhất 1 tiến trình Chromium** cho toàn ứng dụng (tiết kiệm 70–80% RAM).
- Cung cấp `acquire_context(owner_id=..., proxy=...)` cho các tác vụ cần cô lập phiên (như Google OAuth SSO) và `get_shared_context(owner_id=...)` cho Captcha Solver.
- Tự động thu hồi toàn bộ tab/context theo chủ sở hữu qua `release_by_owner(owner_id)` khi dừng hoặc hủy job.

### 3.4. Điều phối Proxy Đa Nền tảng (`ProxyPoolBroker`)
- Nằm tại [`core/proxy/broker.py`](core/proxy/broker.py), điều phối độc quyền toàn bộ Proxy Pool thông qua 4 phương thức chuẩn: `lease`, `release`, `report_failure`, `release_job` kèm tham số bắt buộc `platform_id`.
- Hỗ trợ cấp phát song song đa nền tảng, tự động xoay IP cho proxy xoay và thu hồi sạch các `_pending_leases` qua `unregister_job` để chống deadlock.

---

## 4. Chi tiết Kỹ thuật Nền tảng MiraiMind (`com.immomo.miraimind`)

Các kết luận đã xác minh khi đối chiếu [`core/platforms/miraimind/fetcher.py`](core/platforms/miraimind/fetcher.py) với `MyApp.xapk` (version `1.1.95`):

### `DeviceProfile` hiện tại
- `mmuid`: tạo một `shared_device_seed` 128 ký tự từ các trường thiết bị, sau đó tính `SHA-1(shared_device_seed)` (khớp với output native command `100` trong capture).
- `mmuidv3`: mặc định rỗng.
- `uid`: mặc định rỗng khi chưa đăng nhập; sau khi đăng nhập truyền UID tài khoản thực do server cấp.
- `device_id`: random 8 bytes (`secrets.token_hex(8)`), loại trừ các Android ID không hợp lệ.
- `android_version` & `model`: chọn ngẫu nhiên theo thế hệ Samsung tương ứng:

  | Android | Models |
  | --- | --- |
  | 12 | `SM-S901B`, `SM-S906B`, `SM-S908B`, `SM-A536B` |
  | 13 | `SM-S911B`, `SM-S916B`, `SM-S918B`, `SM-A546B` |
  | 14 | `SM-S921B`, `SM-S926B`, `SM-S928B`, `SM-A556B` |
  | 15 | `SM-S931B`, `SM-S936B`, `SM-S938B`, `SM-A566B` |

- Các trường fingerprint bổ sung gồm `screen`, `serial_no`, `imei`, `mac`, `oaid`, `cid`, `sdcard_path`, `sdcard_perm`, `wifi_state`, `drm_uid` và `shared_device_seed`.
- `MiraimindFetcher` giữ nguyên `mmuid`, `mmuidv3`, `uid` và `device_id` trong cả query params lẫn JSON body khi gửi tới gateway `https://melon-gateway-os.immomo.com/miraimind-server/api/...`.

### Các Job Runners của MiraiMind (`core/platforms/miraimind/runners/`)
1. `invitation_job_runner.py`: Chạy mã mời khách hàng (`customer_invite`).
2. `album_topup_job_runner.py`: Nạp kẹo/điểm vào Album (`album_topup`).
3. `bot_engagement_job_runner.py`: Tăng tương tác Bot chat qua Photon WebSocket (`bot_engagement`).
4. `social_job_runner.py`: Tăng theo dõi Creator & thả tim bài viết (`creator_follow`, `feed_like`).
5. `stock_account_job_runner.py`: Tạo và nuôi tài khoản kho dự trữ (`stock_account`).

---

## 5. Chi tiết Kỹ thuật Nền tảng Mochi Chat (`com.yuedong.mochi`)

Chi tiết phân tích XAPK và giao thức mạng xem tại [MOCHICHAT_ANALYSIS.md](MOCHICHAT_ANALYSIS.md).

- **Kiến trúc ứng dụng gốc:** React Native + Hermes Bytecode v96 (Expo SDK 55.0.0).
- **Backend Host:** `https://mochi.lightchaser.xyz` (REST JSON API qua HTTPS chuẩn).
- **Luồng nghiệp vụ chính:**
  - Đăng nhập Google OAuth SSO qua [`core/auth/google.py`](core/auth/google.py) và `CentralBrowserManager` (hoặc lấy tài khoản từ nhà cung cấp mail).
  - Giao tiếp API qua [`core/platforms/mochichat/api.py`](core/platforms/mochichat/api.py): xác thực `/auth/google`, quản lý hồ sơ `/user/me`, lấy mã mời `/invites/my-code`, nhập mã mời `/invites/redeem`, và điểm danh `/tasks/daily/claim`.
  - Thực thi đa luồng bất đồng bộ qua [`core/platforms/mochichat/runners/invite_referral.py`](core/platforms/mochichat/runners/invite_referral.py) (`MochiInviteRunner`).
