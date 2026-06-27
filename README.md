# YouTube Subtitle Generator Extension

Hệ thống tạo phụ đề tự động và thời gian thực cho YouTube, sử dụng Manifest V3 Chrome Extension kết hợp với Python WebSocket Backend để nhận dạng giọng nói và dịch máy.

## Tính Năng

- Giao diện hiện đại và mượt mà
- Cấu hình URL WebSocket linh hoạt (hỗ trợ cả localhost và ngrok)
- Chụp âm thanh trực tiếp từ tab trình duyệt thông qua Chrome Offscreen API mà không cần mic ngoài
- Phân đoạn giọng nói thời gian thực sử dụng Silero VAD (Voice Activity Detection)
- Nhận dạng giọng nói (ASR) đa ngôn ngữ bằng cách fine-tune Whisper (Tiny) sử dụng LoRA và Wav2Vec 2.0
- Dịch thuật tự động (NMT) sang tiếng Việt bằng cách fine-tune MarianMT (với phân cụm KMeans conditioning) và mBART-50
- Hiển thị phụ đề overlay trực tiếp trên video YouTube với hiệu ứng gõ chữ (typing effect)
- Đồng bộ hóa phụ đề khi tua video thông qua local storage cache (Replay Mode) và tự động chuyển về Live Mode khi tua tới phần chưa capture

## Cài Đặt

### 1. Cài đặt Extension

1. Mở Google Chrome và truy cập đường dẫn `chrome://extensions/`
2. Bật chế độ nhà phát triển (Developer mode) ở góc trên bên phải
3. Click chọn "Load unpacked" (Tải tiện ích đã giải nén)
4. Chọn thư mục `extension` của dự án này

### 2. Chạy Backend Server

Backend chạy trên môi trường Jupyter Notebook:

1. Mở file `server/backend_latest.ipynb` bằng Google Colab hoặc Jupyter Lab cục bộ.
2. Cài đặt các thư viện phụ thuộc cần thiết trong các ô đầu của Notebook (huggingface_hub, pyngrok, openai-whisper, nest_asyncio, transformers, sentence-transformers, websockets).
3. Cấu hình ngrok authtoken nếu cần dùng public tunnel (Notebook hỗ trợ sẵn pyngrok).
4. Chạy toàn bộ các ô để khởi tạo các mô hình học máy (VAD, Whisper, Wav2Vec 2.0, MarianMT, mBART-50) và bật WebSocket Server.
5. Server mặc định sẽ lắng nghe trên cổng `5001`. Lấy địa chỉ URL WebSocket được in ra ở ô cuối (ví dụ: `ws://127.0.0.1:5001` hoặc `wss://xxxx.ngrok-free.app`).

## Cách Sử Dụng

1. Click vào biểu tượng Extension trên thanh công cụ Chrome.
2. Nhập URL WebSocket backend.
3. Chọn mô hình nhận dạng giọng nói (Wav2Vec2 hoặc Whisper Tiny) và mô hình dịch thuật (MarianMT hoặc mBART-50).
4. Lựa chọn ngôn ngữ nguồn (Original Language) và ngôn ngữ dịch (Target Language).
5. Bấm nút kết nối để thiết lập session.
6. Mở một video bất kỳ trên YouTube để xem phụ đề overlay thời gian thực hiển thị trên trình phát video.
7. Khi tua ngược video về trước (seek backward), hệ thống tự động chuyển sang Replay Mode và load lại phụ đề từ cache; khi tua đến các phần mới, hệ thống tự động tiếp tục stream âm thanh và phát phụ đề trực tiếp.

## Kiến Trúc và Luồng Thông Tin

Hệ thống sử dụng mô hình kiến trúc phân tán phục vụ real-time stream:

1. **Content Script (`content.js`):** Theo dõi trạng thái trình phát video YouTube (seek, playback rate, timeupdate) và vẽ overlay phụ đề trực tiếp lên phần tử `<video>`.
2. **Offscreen Document (`offscreen.js`):** Chụp nguồn stream audio từ tab thông qua `chrome.tabCapture` để có chất lượng âm thanh cao nhất (giảm thiểu tiếng ồn ngoại cảnh), chuyển đổi sang PCM 16kHz float32 và truyền vào kết nối WebSocket.
3. **Background Script (`background.js`):** Làm cầu nối giữa Content Script và Offscreen Document.
4. **WebSocket Backend (`backend_latest.ipynb`):**
   - Nhận dữ liệu âm thanh binary từ WebSocket.
   - Áp dụng Silero VAD để nhận biết khoảng lặng và phân đoạn câu thoại thời gian thực.
   - Đưa các câu thoại âm thanh qua pipeline ASR và NMT để lấy văn bản và bản dịch.
   - Trả kết quả JSON trở lại cho Extension hiển thị.

## Giao Thức WebSocket

Hệ thống giao tiếp giữa Extension và Backend hoàn toàn thông qua WebSocket.

### 1. Control Messages (JSON)

- **Gửi từ Extension đến Backend để thiết lập cấu hình:**
  ```json
  {
    "type": "config",
    "asrModel": "whisper_finetuned | wav2vec",
    "translationModel": "marian | mbart"
  }
  ```
- **Gửi từ Extension để đồng bộ thời gian thực của video:**
  ```json
  {
    "type": "time_sync",
    "timestamp": 12.34
  }
  ```
- **Gửi từ Extension để đồng bộ tốc độ phát:**
  ```json
  {
    "type": "playback_rate",
    "rate": 1.25
  }
  ```

### 2. Audio Stream Messages (Binary)

Dữ liệu âm thanh được truyền dưới dạng binary arraybuffer có cấu trúc header như sau:
- `Byte 0`: Mã ngôn ngữ nguồn (0 cho tiếng Anh, 1 cho tiếng Việt)
- `Byte 1`: Mã ngôn ngữ dịch (0 cho tiếng Anh, 1 cho tiếng Việt)
- `Byte 2-9`: Timestamp thời gian thực khi bắt đầu ghi âm (Uint64)
- `Byte 10-11`: Độ dài của chuỗi Video ID (Uint16)
- `Byte 12 -> 12 + VideoIDLength`: Chuỗi Video ID (UTF-8)
- `Các byte tiếp theo`: Dữ liệu âm thanh PCM float32 được ghi nhận từ AudioWorklet

### 3. Response Messages (JSON)

- **Trả về từ Backend sau khi hoàn thành nhận dạng và dịch thuật:**
  ```json
  {
    "type": "transcription",
    "text": "Bản dịch hoặc văn bản nhận dạng tương ứng",
    "start": 5.28,
    "end": 9.28,
    "startClock": 1719472300000
  }
  ```

## Lộ trình Phát Triển (Roadmap)

- Xóa màn hình cấu hình URL khi phát hành chính thức.
- Tối ưu hóa giao diện người dùng, bổ sung các tùy chọn tùy chỉnh style phụ đề (cỡ chữ, màu sắc, vị trí).
- Bổ sung tính năng xuất file phụ đề SRT từ lịch sử đã capture.
