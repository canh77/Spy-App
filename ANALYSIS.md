# Teardown: com.vidix.aivideogenerator.aiphotogenerator

> Phân tích tĩnh phục vụ nghiên cứu cạnh tranh. Nguồn: APK set đã giải nén tại
> `resources/` + `sources/` (Java đã decompile) + native libs, trên máy phân tích.
> Mỗi kết luận tách rõ **Quan sát** (thấy trong artifact) / **Suy luận** / **Chưa biết**.
> Không có giá trị bí mật (API key, token) nào được chép vào tài liệu này.

## 1. Danh tính

| Mục | Giá trị | Nguồn |
|---|---|---|
| Package | `com.vidix.aivideogenerator.aiphotogenerator` | AndroidManifest |
| versionName / versionCode | **1.3.3 / 114** | AndroidManifest |
| compileSdk | 36 (codename 16) | AndroidManifest |
| Phân phối | **Split APK set (AAB)** — base + `config.<lang>` + `config.arm64_v8a` + `config.xxhdpi` | resources/ |
| AdMob App ID | `ca-app-pub-4584260126367940~…` (debugger meta-data) | AndroidManifest |

**Quan sát:** chỉ có ABI `arm64-v8a` trong set này → bản tải về cho máy 64-bit; không thấy `armeabi-v7a`.

## 2. Ngăn xếp kiếm tiền (monetization)

**Quan sát** — các package trong `sources/com`:

| Nhóm | SDK thấy được |
|---|---|
| Mediation | **AppLovin MAX** + **AdMob** |
| Adapter | AdColony (`jirbo`), Vungle, **Mintegral** (`mbridge`), **Pangle/ByteDance** (`bykv`, `pgl`, `tiktok`), **Meta Audience Network** (`facebook`), Vungle |
| Wrapper | **`apero`** — SDK quảng cáo của một studio VN, bọc trên AdMob/AppLovin |
| Billing | Google Play Billing (`com.android.vending.BILLING`, `CHECK_LICENSE`) |

**Suy luận:** đây là mô hình ad-mediation đầy đủ (giống chính sản phẩm của bạn), full-waterfall nhiều network. Sự có mặt của `apero` cho thấy app nhiều khả năng cùng hệ sinh thái studio VN.

## 3. Attribution & analytics

**Quan sát:** Adjust · AppsFlyer (`com.appsflyer.FacebookApplicationId`) · Facebook SDK (ApplicationId + ClientToken) · Firebase (Analytics, Crashlytics, Performance, Remote Config, Messaging, **App Check + Play Integrity**) · ByteDance **APM** (`apm`, `libapminsighta/b.so`).

**Suy luận:** đo lường rất dày — attribution kép (Adjust + AppsFlyer), và **App Check + Play Integrity** nghĩa là họ quan tâm chống lạm dụng API phía backend.

## 4. AI / xử lý ảnh trên thiết bị

**Quan sát** — native libs + assets:

| Thành phần | Bằng chứng |
|---|---|
| TensorFlow Lite + GPU delegate | `libtensorflowlite_jni.so` (4.5MB), `libtensorflowlite_gpu_jni.so` (2.6MB) |
| **MediaPipe Tasks Vision** | `libmediapipe_tasks_vision_jni.so` (**10.3MB**) |
| Model bundle | `assets/models` (**38MB**), `assets/shaders` |
| Hậu xử lý detection | `libnms.so` (NMS), `libyuv-decoder.so` |

**Suy luận:** một phần pipeline ảnh/khuôn mặt chạy **on-device** (MediaPipe + TFLite), không thuần server. 38MB model đi kèm APK là khoản nặng đáng chú ý về size.
**Chưa biết:** phần sinh video/ảnh AI chính chạy local hay gọi API cloud — cần xem lớp network trong `sources` để xác định.

## 5. Chống giả mạo / ký request

**Quan sát (REA):**
- `libsigner.so` — ELF arm64, 1.37MB, SHA-256 `55353b75…`, 74 dynsym / 2 export. (evidence_id `ev_bc76ab…`)
- `libpglarmor.so` (68KB) — Pangle "armor" (chống tamper).

**Suy luận:** request tới dịch vụ của họ nhiều khả năng **được ký ở tầng native** (libsigner) để chống gọi API trực tiếp từ ngoài. Đây chính là lý do kỹ thuật vì sao "rút data từ backend" là việc không làm được chỉ bằng phân tích tĩnh — và không nằm trong phạm vi hợp lệ.

## 6. Quyền đáng chú ý

**Quan sát:** `SYSTEM_ALERT_WINDOW` (overlay) · `ACCESS_ADSERVICES_*` (Privacy Sandbox) · `AD_ID` · `CAMERA` · `READ_MEDIA_IMAGES/VIDEO/AUDIO` + `READ_MEDIA_VISUAL_USER_SELECTED` · `POST_NOTIFICATIONS` · `RECEIVE_BOOT_COMPLETED` · `USE_FULL_SCREEN_INTENT` · `FOREGROUND_SERVICE`.

**Suy luận:** `RECEIVE_BOOT_COMPLETED` + `FULL_SCREEN_INTENT` + `SYSTEM_ALERT_WINDOW` là bộ quyền hay dùng cho **thông báo re-engagement / màn full-screen kéo người dùng lại** — đáng soi nếu bạn so sánh chiến lược retention.

## 7. Nền tảng kỹ thuật

**Quan sát:** Dagger (DI) · Glide (`bumptech`) · Lottie (`airbnb`) · OkHttp · Room · WorkManager · Compose/emoji2 · `canhub` (image cropper) · `makeramen` (rounded image) · `lexiai`, `vidix` (package riêng của app — nhiều khả năng lớp AI/domain của họ).

## 8. REA dùng được gì ở đây

| Việc | Trạng thái |
|---|---|
| `inspect_android_package/class/method` | ❌ cần **file .apk standalone**; thư mục đã giải nén không nhận |
| `inspect_native_exports` / `open_binary` trên `.so` | ✅ chạy tốt (đã dùng cho libsigner.so) |
| Đọc manifest + `sources/` Java | đọc trực tiếp (đã decompile sẵn) |

→ Muốn dùng trọn bộ tool Android của REA thì cần **APK gốc** (một file). `pull_adb_package` kéo từ thiết bị đã cài sẽ cho file hợp lệ.

## 9. Khoảng trống cần đào tiếp

- [ ] Lớp network trong `sources` — host/endpoint app gọi, để biết AI chạy local hay cloud (§4).
- [ ] Cách `libsigner` ký request (cần `open_binary` + pseudocode native).
- [ ] Remote Config keys — chiến lược bật/tắt tính năng & quảng cáo.
- [ ] So sánh version nếu có bản APK khác (REA `compare`).
