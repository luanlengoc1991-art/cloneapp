# Báo cáo phân tích PIX VIP 1.2.3

- **Ngày phân tích:** 16/10/2026
- **Đối tượng:** bản đã cài tại `C:\Program Files\pixvip\` (`pixvip.exe` 1.2.3). Không tìm thấy file `pixvip-1.2.3.exe` trong thư mục dự án, nên mình phân tích bản đã cài.
- **Cách phân tích:** chỉ đọc file (phân tích tĩnh). **Không chạy file EXE**, không build, không sửa gì. REA MCP **chưa được kết nối** trong phiên này, nên mình dùng một đoạn Node.js nhỏ để đọc danh sách file và nội dung bên trong `app.asar` mà không giải nén ra đĩa.

---

## 0. Kết luận nhanh

| Câu hỏi | Trả lời |
|---|---|
| Ứng dụng là gì? | Phần mềm xử lý ảnh bằng AI trên máy tính: phóng to ảnh, chuyển ảnh sang vector, tách nền, cùng nhiều công cụ "AI" chạy qua ChatGPT. |
| Làm bằng gì? | **Electron + Next.js (React)**. Phần phóng to ảnh dùng `upscayl-bin` (Real-ESRGAN, chạy GPU qua Vulkan). |
| Nguồn gốc | **Bản sửa đổi (fork) của [Upscayl](https://github.com/upscayl/upscayl)**, phần mềm mã nguồn mở theo giấy phép **AGPL-3.0**. Tên tác giả trong file vẫn là Nayam Amarshe (tác giả Upscayl). Bản fork do `luonghieuphi` phát hành, lấy bản cập nhật từ repo GitHub `luonghieuphi/kichnet`. |
| Có tái tạo được không? | **Được, khá dễ.** Khoảng 60% là Upscayl gốc (đã có mã nguồn công khai). Phần còn lại là các tính năng thêm vào, làm lại được bằng thư viện mã nguồn mở hoặc API chính thức. |

---

## 1. Công nghệ và cấu trúc ứng dụng

### 1.1 Cấu trúc thư mục cài đặt
```
pixvip/
├─ pixvip.exe              (189 MB) Electron runtime (Chromium + Node)
├─ *.dll, *.pak, icudtl.dat Thành phần Chromium tiêu chuẩn
├─ sharp-runtime/          Thư viện xử lý ảnh sharp (libvips), để riêng ngoài asar
└─ resources/
   ├─ app.asar             (174 MB) Toàn bộ code ứng dụng
   ├─ app.asar.unpacked/   exiftool-vendored (đọc/ghi metadata ảnh)
   ├─ bin/
   │  ├─ upscayl-bin.exe   Công cụ phóng to ảnh AI (Real-ESRGAN ncnn-vulkan)
   │  ├─ vtracer.exe       Chuyển ảnh màu sang vector
   │  └─ potrace.exe       Chuyển ảnh đen trắng sang vector
   ├─ models/              7 model phóng to 4x (.param + .bin, định dạng ncnn)
   ├─ elevate.exe          Xin quyền admin khi cập nhật
   └─ app-update.yml       Tự cập nhật từ GitHub: luonghieuphi/kichnet
```

### 1.2 Thành phần bên trong `app.asar`
```
export/
├─ common/        Hằng số dùng chung: danh sách lệnh IPC, danh sách model, định dạng ảnh
└─ electron/      Tiến trình chính (main process), viết bằng TypeScript rồi biên dịch sang JS
   ├─ index.js    (45 KB) Đăng ký IPC, vector hoá, ChatGPT, GPU, file tạm
   ├─ main-window.js, preload.js
   ├─ commands/   image / batch / double upscale, chọn file/thư mục, dán ảnh, dừng, cập nhật
   └─ utils/      tham số upscayl-bin, GPU Vulkan, gói model AI, ComfyUI, cài đặt
renderer/out/     Giao diện Next.js đã xuất tĩnh (static export)
node_modules/     ~18.700 file thư viện
```

### 1.3 Thư viện chính (theo `package.json`)
- **Giao diện:** React, Next.js 14, Tailwind CSS, DaisyUI/theme-change (đổi giao diện), Radix UI, cmdk, lucide-react, react-compare-slider (kéo so sánh trước/sau), react-resizable-panels, jotai (quản lý trạng thái)
- **Electron:** electron-next, electron-updater, electron-settings, electron-log
- **Xử lý ảnh:** sharp, exiftool-vendored, upscayl-bin, vtracer, potrace
- **Dịch vụ ngoài:** **Firebase** (lưu dữ liệu bản quyền), **PostHog** (thống kê người dùng), ChatGPT (qua cửa sổ web nhúng), ComfyUI (tuỳ chọn, chạy trên máy)
- **Giấy phép khai báo:** `AGPL-3.0`

---

## 2. Chức năng chính

### 2.1 Kế thừa từ Upscayl
| Chức năng | Module |
|---|---|
| Phóng to 1 ảnh | `commands/image-upscayl.js` |
| Phóng to cả thư mục | `commands/batch-upscayl.js` |
| Phóng to 2 lần liên tiếp (16x) | `commands/double-upscayl.js` |
| Dùng model tự thêm | `commands/custom-models-select.js` |
| Dán ảnh từ clipboard | `commands/paste-image.js` |
| Chép metadata EXIF sang ảnh kết quả | `utils/copy-metadata.js` |
| Chọn GPU, tile size, định dạng xuất, tỉ lệ | `utils/get-arguments.js`, `get-preferred-vulkan-gpu.js` |
| Tự cập nhật | `commands/auto-update.js` |

**7 model có sẵn:** upscayl-standard-4x, upscayl-lite-4x, high-fidelity-4x, remacri-4x, ultramix-balanced-4x, ultrasharp-4x, digital-art-4x.

### 2.2 Tính năng PIX VIP thêm vào
| Nhóm | Chức năng | Cách thực hiện |
|---|---|---|
| **Chuyển vector** | Ảnh sang SVG/PDF; chế độ màu hoặc đen trắng; các cấu hình fast/balanced/detailed/portrait/artwork; tạo mặt nạ logo | `vtracer.exe` / `potrace.exe` + sharp. PDF được tạo bằng cửa sổ Electron ẩn rồi `printToPDF` |
| **Tách nền** | Nhiều chế độ tách, nền trong suốt hoặc nền trắng | Model ONNX (BiRefNet general/portrait, ISNet) trong một "AI Model Pack" tải riêng, người dùng tự chọn thư mục |
| **Theo dõi GPU** | Mức dùng GPU, VRAM, nhiệt độ | Gọi `nvidia-smi` |
| **Trợ lý AI qua ChatGPT** | Các công cụ bên dưới | Mở cửa sổ `chatgpt.com/?q=<prompt>`, tự dán ảnh vào, chờ ChatGPT tạo ảnh rồi lấy ảnh về (`persist:anhpro_chatgpt`) |
| **ComfyUI** | Kết nối ComfyUI chạy trên máy (`127.0.0.1:8188`) | Mới chỉ kiểm tra kết nối và lấy danh sách checkpoint |

Các công cụ "AI" trong giao diện (đa phần là **mẫu prompt gửi sang ChatGPT**):
- Phục hồi ảnh cũ (phục hồi nét mặt, khử nhiễu)
- Phục hồi thiết kế biển bảng và ấn phẩm; nắn phẳng, khử méo ảnh chụp chéo
- Render 2D sang 3D; phối cảnh 3D biển hiệu gắn lên mặt tiền
- Retoucher sản phẩm / chuẩn ảnh E-commerce Studio
- Fake Model AI (ghép khuôn mặt và trang phục từ ảnh này sang dáng người ở ảnh khác)
- Cắt và chỉnh ảnh thẻ (làm mịn da)
- Bế file in ấn (tạo đường cắt cho decal/sticker, xem trước khổ A4)
- Lịch sử thiết kế, so sánh trước/sau, ghép 3 ảnh cạnh nhau

### 2.3 Bản quyền và kích hoạt
- Ứng dụng có **hệ thống kích hoạt bằng license key gắn với mã máy (HWID)**, lưu trên Firebase. Có các trạng thái VIP / Premium. Việc kiểm tra key diễn ra **ngay trên giao diện**, kết quả lưu trong `localStorage`.
- Có dấu vết đổi tên thương hiệu nhiều lần: Sharpix, rồi Pixel UP, rồi PIX VIP (`sharpix_license_key`, `pixelup_license_key`).
- Có ảnh quảng cáo popup (`popup-ad.png`) và link liên hệ Messenger.
- *Báo cáo chỉ ghi nhận ở mức tổng quan, không đi sâu vào cách vượt qua cơ chế này.*

---

## 3. Giao diện và quy trình hoạt động

### 3.1 Kiến trúc tiến trình
```
┌──────────── Renderer (Next.js/React) ────────────┐
│ Thanh bên: chọn công cụ, model, cài đặt           │
│ Vùng chính: kéo thả / Ctrl+V, so sánh trước/sau   │
│ Tab ChatGPT (webview cô lập)                      │
└──────────────┬───────────────────────────────────┘
               │ IPC (preload.js → window.electron)
┌──────────────▼──── Main process (Node) ──────────┐
│ spawn upscayl-bin / vtracer / potrace / nvidia-smi│
│ sharp: đổi định dạng, đổi kích thước, tạo mặt nạ  │
│ exiftool: chép metadata                           │
│ BrowserWindow ChatGPT (session riêng, lưu đăng nhập)│
└──────────────────────────────────────────────────┘
```

### 3.2 Quy trình phóng to ảnh
1. Người dùng kéo thả, chọn file hoặc dán ảnh. Renderer gửi lệnh `UPSCAYL` kèm đường dẫn, model, tỉ lệ, GPU và định dạng.
2. Main process ghép tham số rồi chạy `upscayl-bin.exe -i … -o … -m models -n <model> -s 4 …`.
3. Đọc phần trăm tiến độ từ stderr, gửi về giao diện qua `UPSCAYL_PROGRESS`.
4. Nếu tỉ lệ khác 4x thì dùng sharp đổi kích thước hoặc định dạng ("Adding some finishing touches").
5. Chép metadata, rồi gửi `UPSCAYL_DONE` để giao diện hiện thanh kéo so sánh trước/sau.

### 3.3 Quy trình công cụ dùng ChatGPT
1. Người dùng chọn công cụ và các tuỳ chọn. Giao diện ghép thành prompt tiếng Việt.
2. Lệnh `open-ai-assistant` mở cửa sổ ChatGPT với prompt có sẵn và tự dán ảnh vào.
3. Ứng dụng liên tục kiểm tra trang cho đến khi thấy ảnh kết quả, lấy ảnh qua session, lưu vào thư mục tạm, rồi báo về giao diện (`AI_IMAGE_RECEIVED`).
4. Có thể phóng to tiếp kết quả bằng upscayl-bin.

---

## 4. Khả năng tái tạo bằng mã nguồn mới

### 4.1 Đánh giá theo từng phần
| Phần | Độ khó | Cách làm lại |
|---|---|---|
| Khung Electron + Next.js + phóng to ảnh | Thấp | Lấy trực tiếp mã nguồn **Upscayl** (AGPL-3.0) |
| Vector hoá | Thấp | vtracer (MIT), potrace (GPL), sharp |
| Tách nền | Trung bình | Model ONNX BiRefNet/ISNet chạy bằng `onnxruntime-node`, hoặc dùng thư viện `rembg` |
| Công cụ AI (phục hồi ảnh, render 3D, ghép người…) | Trung bình | **Nên dùng API chính thức** (OpenAI Images API, Gemini, hoặc ComfyUI/Stable Diffusion chạy trên máy) thay vì tự động điều khiển trang web ChatGPT |
| Theo dõi GPU, ảnh thẻ, bế file in | Thấp | nvidia-smi, sharp, canvas |
| Giao diện | Trung bình | Tự thiết kế lại; không sao chép logo hay thương hiệu PIX VIP |

**Ước tính:** một người quen Electron/React mất khoảng **3–6 tuần** để có bản tương đương, vì đã có sẵn Upscayl làm nền.

### 4.2 Lưu ý về pháp lý và rủi ro
1. **AGPL-3.0:** Upscayl và bản fork này đều theo AGPL. Bản làm lại dựa trên Upscayl **cũng phải theo AGPL** và phải công khai mã nguồn cho người dùng. Ngược lại, AGPL cũng có nghĩa người dùng PIX VIP có quyền yêu cầu mã nguồn của nó.
2. **Giấy phép model:** một số model cộng đồng (ví dụ UltraSharp, Remacri) có thể chỉ cho phép **dùng phi thương mại** (CC BY-NC-SA). Cần kiểm tra từng model trước khi bán sản phẩm.
3. **Tự động điều khiển ChatGPT qua trình duyệt nhúng** có thể vi phạm điều khoản sử dụng của OpenAI và dễ hỏng mỗi khi giao diện ChatGPT thay đổi. Nên dùng API chính thức.
4. **Thương hiệu:** không dùng tên, logo hay hệ thống key của PIX VIP/Sharpix. Nên tự viết hệ thống bản quyền riêng (nếu cần) và kiểm tra key ở phía máy chủ.

### 4.3 Điểm yếu kỹ thuật của bản gốc (nên làm tốt hơn khi viết lại)
- `nodeIntegration: true` và `webSecurity: false` là cấu hình rủi ro. Nên bật `contextIsolation`, tắt nodeIntegration và chỉ mở các hàm cần thiết qua preload.
- Kiểm tra bản quyền ở giao diện và lưu kết quả trong `localStorage`, nên rất dễ bị làm giả.
- Tính năng vector hoá chỉ hỗ trợ Windows. Thư mục `scripts/` mà code gọi tới không có trong bản cài, nên một số tính năng có thể đang lỗi.
- Có gửi dữ liệu thống kê lên PostHog (cần thông báo quyền riêng tư cho người dùng).

---

## 5. Đề xuất bước tiếp theo
1. Kết nối REA MCP nếu muốn phân tích sâu hơn (ví dụ các file nhị phân `upscayl-bin`, `vtracer`).
2. Tải mã nguồn Upscayl, chạy được bản gốc trước.
3. Thêm lần lượt từng phần: vector hoá, tách nền (ONNX), rồi công cụ AI qua API chính thức.
4. Thiết kế giao diện và thương hiệu riêng, công bố mã nguồn theo AGPL.

---

## 6. Cập nhật: bản pixAI 1.2.6 (16/10/2026)

- **Vị trí cài đặt:** `C:\Program Files\pixvip\pixAI\pixAI.exe`. Bản 1.2.3 đã bị gỡ.
- **Đổi tên:** PIX VIP → **pixAI** (logo mới "PIX AI"). Vẫn cập nhật từ repo `luonghieuphi/kichnet`.
- **Không đổi:** phần xử lý chính (`export/electron/index.js` giống hệt từng byte), các file `upscayl-bin`, `vtracer`, `potrace`, 7 model, danh sách 13 công cụ.
- **Thay đổi nằm ở giao diện:** file giao diện chính tăng từ 618 KB lên 693 KB, khoảng 170 nhãn mới.

| Công cụ | Thay đổi trong 1.2.6 |
|---|---|
| Tách nền | Đổi tên thành **"Xóa Nền PNG"** |
| Tách lớp | Đổi thành **"Bóc Tách Layer"**: người dùng tự khoanh vùng từng đối tượng (kéo khung, đặt tên). AI trả về N ảnh PNG không nền và 1 ảnh nền đã xoá đối tượng. Có thể "Tách tiếp" nhiều lượt. |
| Chỉnh sửa ảnh thẻ | Mở rộng nhiều nhất. **Nhóm tuổi** (Trẻ em, Thanh niên/SV, Đi làm, Lão niên) thay cho giới tính. **Trang phục:** sơ mi trắng/xanh, vest đen, áo dài, polo, hoặc tải ảnh mẫu đồ. **7 kiểu tóc.** **Phông nền:** xanh CCCD/hộ chiếu, xanh đậm/nhạt, trắng Visa, xám. Có mịn da và chỉnh sáng. **Chế độ hàng loạt** (tự gửi 10 ảnh mỗi lượt sang ChatGPT). |
| Ảnh thẻ: **Cắt ảnh** (mới, chạy trên máy) | Khung cắt kéo được 4 góc, kích thước 2x3, 3x4, 4x6, Visa 3.5x4.5, Visa Mỹ 5x5, hoặc tự do |
| Ảnh thẻ: **Xếp in ảnh** (mới, chạy trên máy) | Khổ giấy 10x15, A4, A5, dọc/ngang. Có tự động lấp đầy, xếp thủ công, combo 3x4 + 4x6. Chỉnh khe cắt, lề an toàn, kẻ viền cắt (độ dày, màu). Xuất 300/600 DPI, PNG/JPG, in trực tiếp (Ctrl+P). |
| Thiết kế AI | Đổi thành **"Thiết kế quảng cáo"**. Backdrop sự kiện có thêm vị trí chữ nổi (giữa vách, đỉnh vòm, lệch trái/phải, phân tầng, tự nhập) và kiểu standee nhân vật (giữ nguyên hoặc **Chibi 3D**). Các mục được chia thành phần đánh số, gập/mở được. |
| Phục hồi ấn phẩm | Mô tả đổi thành "chuẩn in 4K" (chỉ đổi chữ) |
