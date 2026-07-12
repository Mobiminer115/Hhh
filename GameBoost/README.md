# GameBoost

GameBoost là Theos tweak dành cho **thiết bị iOS đã jailbreak**, chỉ nên inject
vào game/app mà bạn sở hữu hoặc được phép kiểm thử. Project tạo cả file
`GameBoost.dylib` và gói cài đặt `.deb` bằng GitHub Actions.

## Chức năng

- Nút `GB` nổi, kéo được; chạm để bật/tắt menu.
- Cửa sổ trong suốt chỉ nhận touch tại nút và panel, nên phần còn lại không chặn
  thao tác của game.
- `Performance QoS`: nâng QoS của thread gọi `CAMetalLayer.nextDrawable` lên
  `USER_INTERACTIVE`. Chế độ này tự tắt khi iOS báo nhiệt độ `serious` hoặc
  `critical`.
- `Metal render scale`: đổi `CAMetalLayer.drawableSize` từ 50% đến 150%, bước
  5%; thiết lập được lưu riêng trong game.

## Giới hạn kỹ thuật quan trọng

- iOS không cung cấp API user-space để ghim một game vào toàn bộ lõi CPU. Chế
  độ hiệu năng ở đây là **scheduler/QoS hint**, không tắt quản lý nhiệt và không
  cam kết mọi lõi đều chạy.
- Render scale chỉ tác động pipeline dùng `CAMetalLayer`. Game OpenGL hoặc game
  có hệ thống dynamic resolution riêng có thể bỏ qua/ghi đè kết quả và cần hook
  riêng cho Unity/Unreal.
- Upscale trên 100% tăng tải GPU, nhiệt và pin; không tự tạo thêm chi tiết hình
  ảnh như một thuật toán MetalFX/FSR.
- Đây không phải dylib dùng cho iOS nguyên bản/App Store. Nó cần Substrate,
  Substitute hoặc lớp tương thích tương đương trên jailbreak.

## Tương thích build mặc định

| Artifact | iOS | Kiểu jailbreak | Slice |
| --- | --- | --- | --- |
| `gameboost-rootful-ios12-13-arm64` | 12–13.7 | rootful | `arm64` |
| `gameboost-rootful-ios14-plus` | 14–18 | rootful | `arm64`, `arm64e` |
| `gameboost-rootless-ios15-plus` | 15–18 | rootless | `arm64`, `arm64e` |

App Store game thông thường chạy slice `arm64`, kể cả trên thiết bị A12 trở lên;
slice `arm64e` chủ yếu cần khi inject vào binary hệ thống. Hai artifact hiện đại
vẫn kiểm tra bắt buộc cả hai slice theo yêu cầu của project. Artifact iOS 12–13
dùng `arm64` vì GitHub runner/Xcode hiện tại không tạo được arm64e ABI cũ.

## Cấu hình Bundle ID

Không nên inject tweak vào mọi tiến trình. Hãy thay `com.example.game` trong
`GameBoost.plist` bằng Bundle ID chính xác của game, ví dụ:

```xml
<array>
    <string>com.yourstudio.yourgame</string>
</array>
```

Bạn cũng có thể nhập Bundle ID khi bấm **Run workflow**, hoặc tạo Repository
Variable tên `GAME_BUNDLE_ID`. Workflow sẽ kiểm tra định dạng trước khi build.

## Build trên GitHub Actions

1. Tạo repository mới và đưa toàn bộ nội dung thư mục này lên nhánh `main`.
2. Mở tab **Actions** → **Build GameBoost** → **Run workflow**.
3. Nhập Bundle ID của game.
4. Khi job xong, tải một trong ba artifact phù hợp jailbreak. Mỗi artifact có
   `.deb`, `GameBoost.dylib` và `architectures.txt`.

Workflow chạy trên macOS/Xcode, cài Theos cùng `ldid`, build ba package và dùng
`lipo` để xác nhận đúng các slice kiến trúc của từng artifact.

## Build cục bộ trên macOS

Yêu cầu Xcode đầy đủ, Theos, `ldid` và `xz`.

Rootful:

```bash
export THEOS="$HOME/theos"
make clean package FINALPACKAGE=1
```

Rootless:

```bash
export THEOS="$HOME/theos"
make clean package FINALPACKAGE=1 THEOS_PACKAGE_SCHEME=rootless
```

Rootful iOS 12–13 (game/app `arm64`):

```bash
export THEOS="$HOME/theos"
make clean package FINALPACKAGE=1 \
  ARCHS=arm64 TARGET=iphone:clang:latest:12.0
```

Các file đầu ra nằm trong `packages/`; dylib đã stage nằm dưới `.theos/_/`.

## iOS 12–13 và arm64e ABI cũ

Workflow đã build riêng `arm64` cho game/app trên iOS 12–13.7. Muốn inject vào
binary hệ thống `arm64e` trên các bản iOS này cần thêm một job dùng toolchain
Xcode cũ phù hợp; không thể trộn ABI arm64e cũ và mới thành một binary duy nhất
bằng Xcode hiện tại. Vì vậy artifact legacy không được gắn nhãn hỗ trợ arm64e.
