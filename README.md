# ChatApp — Ứng dụng nhắn tin nội bộ (C# / .NET)

Khung dự án khởi tạo cho hệ thống nhắn tin đa nền tảng, lưu trữ trên server nội bộ (local).

## Cấu trúc solution

```
ChatApp.sln
src/
  ChatApp.Shared/   Các DTO dùng chung giữa Server và Client
  ChatApp.Server/   ASP.NET Core Web API + SignalR + EF Core (backend, chạy trên server local)
  ChatApp.Client/   .NET MAUI — 1 codebase C# chạy Windows, iOS, Android, macOS
```

## Công nghệ

| Thành phần | Công nghệ |
|---|---|
| Backend API | ASP.NET Core 8 Web API |
| Real-time | SignalR (WebSocket, tự reconnect) |
| Database | SQL Server (đổi sang PostgreSQL/SQLite dễ dàng qua EF Core) |
| Lưu file | Ổ đĩa local server, phân thư mục theo năm/tháng (`FileStorageService`) |
| Auth | JWT Bearer Token + BCrypt hash mật khẩu |
| Client | .NET MAUI (MVVM, CommunityToolkit.Mvvm) |

## Các tính năng đã dựng khung theo yêu cầu

- **Đa nền tảng**: 1 project MAUI build ra Windows / iOS / Android / macOS.
- **Cá nhân hóa**: hồ sơ người dùng (avatar, status message), token lưu theo thiết bị (SecureStorage).
- **Nhóm chat**: tạo / sửa / xóa (soft-delete), loại `Direct` (1-1) / `Private` / `Public`, quản lý thành viên & vai trò (Owner/Admin/Member) — `GroupsController`.
- **Nhắn tin**: gửi tin realtime qua SignalR, xem lịch sử phân trang (cuộn ngược tải thêm), gửi file/ảnh đính kèm — `MessagesController`, `FilesController`.
- **Lưu trữ dài hạn**: tin nhắn lưu SQL Server, file lưu ổ đĩa local, index theo `GroupId + SentAtUtc` để truy vấn lịch sử nhanh dù dữ liệu lớn.
- **Đồng bộ đa thiết bị**: mọi client cùng tài khoản join chung SignalR group, nhận `ReceiveMessage` như nhau — mở app trên điện thoại và máy tính cùng lúc vẫn đồng bộ.
- **Chiến dịch (Campaign)**: tạo & gửi thông báo hàng loạt tới nhiều nhóm/người dùng, trạng thái Draft/Scheduled/Running/Completed — `CampaignsController`.

## Chạy thử (development)

### 1. Backend
```bash
cd src/ChatApp.Server
# Cài EF Core CLI nếu chưa có
dotnet tool install --global dotnet-ef

# Sửa ConnectionStrings:Default trong appsettings.json trỏ tới SQL Server local
dotnet ef migrations add InitialCreate
dotnet run
```
API mặc định chạy ở `https://localhost:5001` hoặc `http://localhost:5000` (xem `launchSettings.json` do Visual Studio sinh khi mở project lần đầu).

### 2. Client (MAUI)
Mở `ChatApp.sln` bằng Visual Studio 2022 (cài workload **.NET Multi-platform App UI development**), chọn project `ChatApp.Client`, chọn target (Windows Machine / Android Emulator / iOS Simulator — cần Mac để build iOS) rồi Run.

Ở màn hình đăng nhập, nhập địa chỉ IP nội bộ của máy chạy server, ví dụ `http://192.168.1.10:5000`.

## Lịch sử sửa lỗi build

Phiên bản đầu chỉ có mã nguồn C#/XAML mà thiếu các file scaffolding bắt buộc của .NET MAUI, gây lỗi khi build:

| Lỗi | Nguyên nhân | Đã sửa |
|---|---|---|
| `XA1018 AndroidManifest file does not exist` | Thiếu `Platforms/Android/AndroidManifest.xml` | Đã thêm `AndroidManifest.xml`, `MainActivity.cs`, `MainApplication.cs` |
| `MSB3954 Failed to compute hash for splash.svg` | Thiếu `Resources/Splash/splash.svg` và `Resources/AppIcon/*.svg` | Đã thêm icon/splash SVG mẫu (màu tím `#512BD4`, có thể thay bằng logo thật) |
| `Improper project configuration: no AppxManifest, WindowsPackageType not MSIX` | Thiếu cấu hình đóng gói Windows | Đã set `<WindowsPackageType>None</WindowsPackageType>` (build Windows dạng unpackaged, không cần Store) + thêm `Platforms/Windows/App.xaml`, `App.xaml.cs`, `app.manifest` |
| (tiềm ẩn) thiếu font `OpenSans-Regular.ttf` / `OpenSans-Semibold.ttf` được khai báo trong `MauiProgram.cs` | Chưa có file font thật | Đã tải font Open Sans gốc vào `Resources/Fonts/` |
| Thiếu scaffolding iOS/macOS | Chưa có `AppDelegate.cs`, `Info.plist` | Đã thêm cho cả iOS và MacCatalyst |

Nếu sau này muốn đóng gói Windows lên Microsoft Store (MSIX), hãy xoá dòng `<WindowsPackageType>None</WindowsPackageType>` và thêm lại file `Package.appxmanifest` cùng bộ ảnh Store (`Square150x150Logo.png`, `Square44x44Logo.png`, `Wide310x150Logo.png`, `StoreLogo.png`, `SplashScreen.png`) trong `Platforms/Windows/Images/`.

### Lần sửa 2

| Lỗi | Nguyên nhân | Đã sửa |
|---|---|---|
| `CS1061 'ILoggingBuilder' does not contain 'AddDebug'` | Thiếu package `Microsoft.Extensions.Logging.Debug` mà `MauiProgram.cs` gọi `builder.Logging.AddDebug()` | Đã thêm `PackageReference` vào `ChatApp.Client.csproj` |
| Hàng loạt `CS0103 'InitializeComponent' does not exist` ở mọi trang XAML | **Hệ quả dây chuyền** của lỗi CS1061 ở trên — khi có lỗi biên dịch C#, bộ sinh mã (source generator) cho XAML không chạy nên các `partial class` không có phần `InitializeComponent` được sinh ra | Tự hết sau khi sửa lỗi `AddDebug` ở trên — không phải lỗi XAML thật sự |
| `IDE0130 Namespace "ChatApp.Shared.DTOs" does not match folder structure` | File `DTOs.cs` khai báo `namespace ChatApp.Shared.DTOs` nhưng lại nằm trực tiếp ở thư mục gốc project `ChatApp.Shared` | Đã chuyển file vào `ChatApp.Shared/DTOs/DTOs.cs` cho khớp namespace |

**Lưu ý chung**: khi Error List hiện rất nhiều lỗi `InitializeComponent`/`InitializeComponent does not exist` cùng lúc ở nhiều trang khác nhau, gần như luôn là *hệ quả* của 1 lỗi biên dịch C# thật ở nơi khác (thường nằm ở dòng lỗi có mã như CS10xx/CS0246 chứ không phải CS0103) — nên ưu tiên sửa lỗi đó trước rồi build lại, thay vì sửa từng file XAML.



1. **Migration & seed data**: chạy `dotnet ef migrations add` sau khi chỉnh sửa model.
2. **HTTPS nội bộ**: cấp chứng chỉ nội bộ (internal CA) cho server local, hoặc dùng reverse proxy (Nginx/IIS) có TLS.
3. **Background job cho Campaign hẹn giờ**: dùng `IHostedService` / Hangfire để quét các Campaign có `ScheduledAtUtc` đã tới hạn và gọi `RunCampaignAsync`.
4. **Nén/giới hạn dung lượng file, quét virus** trước khi lưu vào `FileStorageService`.
5. **Backup định kỳ** cho DB và thư mục lưu file (lưu trữ dài hạn).
6. **Đẩy thông báo (push notification)** khi app chạy nền trên mobile — dùng Firebase Cloud Messaging / Apple Push Notification qua MAUI Essentials hoặc Plugin.Firebase.
7. **Refresh token** thay vì JWT sống 7 ngày cố định, để tăng bảo mật.
8. **Mã hóa nội dung tin nhắn tại chỗ (at rest)** nếu yêu cầu bảo mật cao hơn.
