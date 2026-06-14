Chào bạn, câu hỏi rất hay! Mình sẽ giải thích đoạn này một cách dễ hiểu.

Câu đầy đủ: *"Containers don’t usually ship with configuration files baked in though. This is because the contents of a container are ‘stateless’ or ‘immutable’."*

Tạm dịch: *"Tuy nhiên, container thường không được đóng gói sẵn với các file cấu hình bên trong. Điều này là do nội dung của một container là ‘stateless’ (không lưu trạng thái) hoặc ‘immutable’ (bất biến)."*

Vậy **"stateless"** và **"immutable"** nghĩa là gì?

### 1. Immutable (Bất biến)
Nghĩa là **không thể thay đổi** sau khi container được tạo ra.
- Ví dụ: Bạn tạo một container từ image (bản mẫu) của Ubuntu + Java 17. Lúc này, bên trong container có file hệ thống, thư viện Java 17,... Bạn không thể sửa file Java 17 đó thành Java 11 được. Container coi phần lõi đó là "bất biến" – giống như một cái đĩa CD đã ghi xong, không thể ghi đè lên.

### 2. Stateless (Không lưu trạng thái)
Nghĩa là container **không tự nhớ dữ liệu** sau khi nó chạy xong hoặc khởi động lại.
- Ví dụ: Nếu container chạy ứng dụng web, sau khi khởi động lại, nó sẽ **quên** tất cả dữ liệu được tạo ra trong lần chạy trước (trừ khi bạn cố tình gắn thêm ổ đĩa bên ngoài vào). Nó giống như một chiếc máy tính mà mỗi lần khởi động lại là quay về trạng thái ban đầu, mất hết file vừa tạo.

### 3. Tại sao các file cấu hình không được "bake in" (nướng sẵn) vào container?

Vì:
- Nếu bạn **"nướng sẵn"** file cấu hình (ví dụ: `config.properties` với địa chỉ database là `localhost`) vào bên trong container, thì container đó sẽ bất biến → không thể sửa `localhost` thành địa chỉ thật khi đưa lên máy chủ khác.
- Bạn sẽ phải tạo lại container mỗi khi cần đổi cấu hình – rất bất tiện.

**Giải pháp:** Cấu hình được cung cấp từ bên ngoài vào lúc chạy container, ví dụ:
- Biến môi trường (`DATABASE_URL=...`)
- Gắn file cấu hình từ máy chủ vào container (gọi là volume hoặc bind mount)

Như vậy, container vẫn giữ được tính **immutable** (phần lõi không đổi) và **stateless** (không lưu cấu hình bên trong), nhưng ứng dụng vẫn chạy với đúng cấu hình cần thiết.

### Tóm lại
Bạn hãy hình dung container như một chiếc **hộp đựng đồ ăn mang đi**:
- **Immutable**: Hộp đó khi đã đúc xong thì bạn không thể thay đổi chất liệu hay hình dạng hộp.
- **Stateless**: Bạn không nên nấu luôn nước sốt hoặc gia vị vào trong hộp từ trước, vì khi đến nơi, mỗi người ăn lại muốn gia vị khác nhau. Thay vào đó, bạn mang gia vị riêng (cấu hình bên ngoài) rắc vào khi ăn.

Hy vọng giúp bạn hiểu rõ hơn! Bạn có muốn mình lấy một ví dụ cụ thể về việc chạy container với cấu hình từ bên ngoài không?
