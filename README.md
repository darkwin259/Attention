# THUYẾT MINH MÔ TẢ THUẬT TOÁN PHẦN MỀM
## Phần mềm Hiển thị Công suất Phát và Điều chỉnh Suy hao Khối Thu Phát Vệ Tinh K31-TTVT

---

## 1. Tổng quan

Phần mềm K31-TTVT được xây dựng trên nền tảng Python sử dụng thư viện đồ họa PyQt5, thực hiện hai chức năng chính: **giám sát các thông số thiết bị** (nhiệt độ, công suất phát) và **điều khiển mức suy hao** của khối thu phát vệ tinh thông qua giao tiếp nối tiếp UART với vi điều khiển STM32F407. Thuật toán hoạt động của phần mềm được chia thành hai nhánh song song: **nhánh giám sát** (nhận và hiển thị dữ liệu từ STM32) và **nhánh điều khiển** (gửi lệnh điều chỉnh suy hao xuống STM32).

Lưu đồ thuật toán mô tả toàn bộ quy trình hoạt động của phần mềm từ khi khởi động đến khi kết thúc phiên làm việc, bao gồm các bước xác thực người dùng, thiết lập kết nối, giám sát dữ liệu thời gian thực và điều khiển thiết bị từ xa.

---

## 2. Thuyết minh chi tiết thuật toán

### 2.1. Khởi động ứng dụng và xác thực người dùng

Khi ứng dụng được khởi động, hệ thống hiển thị cửa sổ **đăng nhập người dùng** yêu cầu nhập thông tin tài khoản (tên đăng nhập và mật khẩu). Đây là bước bảo mật nhằm phân quyền sử dụng phần mềm. Hệ thống hỗ trợ hai cấp quyền: tài khoản **quản trị** (admin) có toàn quyền điều khiển, và tài khoản **khách** (guest) chỉ có quyền quan sát.

Sau khi người dùng nhập thông tin, thuật toán thực hiện kiểm tra: **thông tin đăng nhập có hợp lệ hay không?**

- Nếu **không hợp lệ**: hệ thống hiển thị **thông báo lỗi** cho người dùng biết thông tin đăng nhập sai, sau đó quay trở lại bước đăng nhập để người dùng nhập lại.

- Nếu **hợp lệ**: hệ thống xác nhận quyền truy cập và chuyển sang bước tiếp theo — hiển thị giao diện chính của phần mềm.

### 2.2. Hiển thị giao diện và thiết lập cấu hình kết nối

Sau khi xác thực thành công, phần mềm **hiển thị giao diện** chính (Trang chủ). Tại đây, người dùng thực hiện bước cấu hình kết nối bằng cách **chọn cổng COM và tốc độ truyền** (baudrate) phù hợp với thiết bị phần cứng. Hệ thống tự động quét và liệt kê các cổng COM đang có sẵn trên máy tính, cập nhật định kỳ mỗi 2 giây để phát hiện khi thiết bị được cắm vào hoặc rút ra.

### 2.3. Thiết lập kết nối với thiết bị

Khi người dùng nhấn nút "Bắt đầu chương trình", thuật toán tiến hành kiểm tra: **kết nối cuộc với thiết bị có thành công hay không?**

- Nếu **không thành công** (cổng COM không tồn tại, đang bị chiếm, hoặc thiết bị chưa được kết nối): hệ thống hiển thị **thông báo kiểm tra kết nối**, yêu cầu người dùng kiểm tra lại cáp kết nối, chọn đúng cổng COM, sau đó quay lại bước chọn cổng COM.

- Nếu **thành công**: phần mềm khởi tạo luồng đọc dữ liệu nối tiếp (Serial Reader Thread), dừng bộ đếm quét cổng COM, và chuyển giao diện sang **chế độ điều khiển và giám sát**. Tại chế độ này, hai nhánh xử lý song song được kích hoạt.

### 2.4. Nhánh giám sát — Nhận và hiển thị dữ liệu (nhánh trái)

Đây là nhánh xử lý chính chạy liên tục trong suốt phiên làm việc, thực hiện vòng lặp sau:

**Bước 1 — Nhận dữ liệu từ STM32 qua UART**: Luồng Serial Reader liên tục lắng nghe dữ liệu từ vi điều khiển STM32 gửi lên qua giao tiếp UART. Vi điều khiển STM32 định kỳ mỗi 500ms gửi một gói dữ liệu chứa giá trị công suất phát và nhiệt độ thiết bị, theo định dạng chuỗi ký tự phân cách bởi dấu phẩy (ví dụ: "140,330" tương ứng 14.0W và 33.0°C).

**Bước 2 — Tách dữ liệu nhiệt độ và công suất**: Sau khi nhận được chuỗi dữ liệu, phần mềm thực hiện phân tích cú pháp (parsing) để tách riêng hai thành phần: giá trị công suất phát (đơn vị Watt) và giá trị nhiệt độ (đơn vị °C). Các giá trị thô được chia cho 10 để chuyển về đơn vị thực tế. Đồng thời, thuật toán kiểm tra tính hợp lệ của dữ liệu: nếu nhiệt độ vượt quá 100°C hoặc công suất vượt ngưỡng cho phép, giá trị sẽ được đặt về 0 và ghi nhận cảnh báo, tránh hiển thị sai lệch trên giao diện.

**Bước 3 — Hiển thị dữ liệu và ghi log**: Các giá trị sau khi xử lý được cập nhật lên giao diện đồ họa thông qua các widget dạng đồng hồ đo (Gauge Widget), hiển thị trực quan nhiệt độ và công suất phát hiện tại. Song song đó, dữ liệu được ghi lại vào file nhật ký CSV (bao gồm mốc thời gian, nhiệt độ, công suất và giá trị suy hao hiện tại) phục vụ cho việc truy xuất và xuất báo cáo sau này.

Sau khi hoàn thành một chu kỳ xử lý, thuật toán kiểm tra: **người dùng có kết thúc phiên làm việc không?**

- Nếu **không**: vòng lặp quay lại Bước 1, tiếp tục nhận và xử lý dữ liệu mới từ STM32.

- Nếu **có** (người dùng nhấn nút kết thúc hoặc đóng ứng dụng): phần mềm thực hiện **lưu cài đặt** (cổng COM đã chọn, giá trị suy hao cuối cùng, giao diện sáng/tối), **đóng kết nối** serial, đóng file nhật ký và **thoát** chương trình.

### 2.5. Nhánh điều khiển — Điều chỉnh suy hao (nhánh phải)

Song song với nhánh giám sát, nhánh điều khiển cho phép người dùng tương tác chủ động với thiết bị để thay đổi mức suy hao tín hiệu. Quy trình gồm các bước sau:

**Bước 1 — Người dùng điều chỉnh mức suy hao**: Trên giao diện điều khiển, người dùng sử dụng thanh trượt (slider) để chọn mức suy hao mong muốn. Thanh trượt có 64 bước (từ 0 đến 63), mỗi bước tương ứng 0.5 dB, cho phép điều chỉnh suy hao trong dải từ 0.0 dB đến 31.5 dB. Giá trị suy hao hiện tại được hiển thị trực quan trên giao diện theo thời gian thực khi người dùng kéo thanh trượt.

**Bước 2 — Quy đổi mức suy hao thành giá trị điều khiển**: Khi người dùng xác nhận giá trị suy hao (nhấn nút OK hoặc phím Enter), phần mềm quy đổi giá trị dB thành giá trị điều khiển số nguyên theo công thức: *giá trị gửi = mức suy hao (dB) × 10*. Ví dụ: mức suy hao 10.5 dB được quy đổi thành giá trị 105.

**Bước 3 — Gửi lệnh suy hao từ App đến STM32**: Giá trị điều khiển được đóng gói thành chuỗi ký tự kèm theo ký tự phân cách 'd' (ví dụ: "105d") và gửi xuống vi điều khiển STM32 thông qua kết nối UART đang hoạt động.

**Bước 4 — STM32 điều khiển IC suy hao PE4302**: Phía vi điều khiển STM32, khi nhận được lệnh qua UART, bộ xử lý ngắt UART sẽ tích lũy các byte nhận được vào bộ đệm cho đến khi gặp ký tự phân cách 'd'. Sau đó, chuỗi số trong bộ đệm được chuyển đổi thành giá trị nguyên, quy đổi thành số bước suy hao (mỗi bước = 0.5 dB, tức chia cho 5), và gửi qua giao tiếp SPI đến IC suy hao số PE4302. IC PE4302 là bộ suy hao 6-bit, nhận dữ liệu SPI và tự động thiết lập mức suy hao RF tương ứng. Tín hiệu Latch Enable (LE) được kích xung dương để chốt giá trị suy hao mới vào IC.

**Bước 5 — Cập nhật trạng thái và lưu bản ghi điều chỉnh**: Sau khi gửi lệnh thành công, phần mềm cập nhật trạng thái trên giao diện và ghi lại bản ghi điều chỉnh vào nhật ký phiên (session log), bao gồm: mốc thời gian, giá trị nhiệt độ và công suất tại thời điểm điều chỉnh, cùng mức suy hao đã thiết lập. Bản ghi này phục vụ cho việc xuất báo cáo dưới dạng file Excel hoặc PDF.

---

## 3. Tóm tắt luồng hoạt động

Toàn bộ thuật toán của phần mềm K31-TTVT có thể được tóm tắt qua các giai đoạn chính sau:

| Giai đoạn | Mô tả | Kết quả |
|-----------|-------|---------|
| **Khởi động** | Ứng dụng khởi chạy, hiển thị cửa sổ đăng nhập | Xác thực người dùng |
| **Xác thực** | Kiểm tra tên đăng nhập và mật khẩu | Phân quyền admin/guest |
| **Cấu hình** | Chọn cổng COM và tốc độ truyền | Thiết lập tham số kết nối |
| **Kết nối** | Mở cổng serial, khởi tạo luồng đọc dữ liệu | Sẵn sàng truyền/nhận dữ liệu |
| **Giám sát** | Vòng lặp nhận dữ liệu → phân tích → hiển thị → ghi log | Hiển thị thời gian thực trên Gauge |
| **Điều khiển** | Chọn suy hao → quy đổi → gửi UART → SPI → PE4302 | Thiết lập suy hao RF trên phần cứng |
| **Kết thúc** | Lưu cài đặt, đóng kết nối, đóng file log | Thoát ứng dụng an toàn |

---

## 4. Đặc điểm kỹ thuật nổi bật

**a) Xử lý đa luồng (Multi-threading):** Phần mềm sử dụng luồng riêng (QThread) cho việc đọc dữ liệu serial, tách biệt khỏi luồng giao diện chính. Điều này đảm bảo giao diện người dùng không bị "đơ" (freeze) trong quá trình chờ dữ liệu từ thiết bị, duy trì trải nghiệm tương tác mượt mà.

**b) Cơ chế cờ hiệu (Flag mechanism):** Thuật toán sử dụng biến cờ `data_ready` để đồng bộ giữa luồng đọc serial và luồng xử lý dữ liệu. Khi một gói dữ liệu được nhận, cờ được đặt lên (set), ngăn việc đọc chồng chéo. Sau khi dữ liệu được xử lý xong, cờ được hạ xuống (reset) để sẵn sàng nhận gói dữ liệu tiếp theo.

**c) Kiểm tra tính hợp lệ dữ liệu:** Trước khi hiển thị, dữ liệu nhận từ STM32 được kiểm tra ngưỡng (nhiệt độ ≤ 100°C, công suất ≤ 300W). Các giá trị vượt ngưỡng được xử lý bằng cách đặt về 0 và ghi nhận cảnh báo, tránh hiển thị sai lệch trên giao diện.

**d) Lưu trữ và xuất báo cáo:** Toàn bộ dữ liệu giám sát được lưu liên tục vào file CSV. Người dùng có thể xuất báo cáo dưới dạng file Excel (kèm biểu đồ nhiệt độ và công suất theo thời gian) hoặc file PDF (kèm bảng dữ liệu và logo đơn vị) phục vụ công tác quản lý và báo cáo.

**e) Ghi nhớ cài đặt:** Phần mềm lưu lại các tham số cài đặt của phiên làm việc trước (cổng COM, mức suy hao, giao diện sáng/tối) và tự động khôi phục khi khởi động lại, giúp giảm thao tác cấu hình lặp lại cho người dùng.

---

## 5. Sơ đồ tương tác hệ thống

Mối quan hệ giữa các thành phần trong hệ thống được tóm tắt như sau:

```
┌─────────────────────────────────────────────────────────────────────┐
│                        PHẦN MỀM (PC)                               │
│                                                                     │
│   ┌──────────┐    ┌────────────────┐    ┌────────────────────────┐  │
│   │  Đăng    │───▶│  Giao diện     │───▶│  Chế độ Điều khiển    │  │
│   │  nhập    │    │  chính (Home)  │    │  & Giám sát           │  │
│   └──────────┘    └────────────────┘    │                        │  │
│                                         │  ┌──────┐  ┌────────┐ │  │
│                                         │  │Gauge │  │Slider  │ │  │
│                                         │  │Widget│  │Suy hao │ │  │
│                                         │  └──┬───┘  └───┬────┘ │  │
│                                         └─────┼──────────┼──────┘  │
│                                               │          │         │
│                              ┌────────────────┘          │         │
│                              │ Hiển thị                  │ Gửi    │
│                              ▼                           ▼         │
│                    ┌──────────────────────────────────────────┐     │
│                    │      Serial Reader Thread (UART)         │     │
│                    └───────────────────┬──────────────────────┘     │
└────────────────────────────────────────┼────────────────────────────┘
                                         │
                              USB-UART (FT232, 115200 baud)
                                         │
┌────────────────────────────────────────┼────────────────────────────┐
│                        PHẦN CỨNG (STM32F407)                       │
│                                         │                           │
│                    ┌───────────────────┴──────────────────────┐     │
│                    │         USART1 (Interrupt-driven)         │     │
│                    └──┬────────────────────────────────────┬──┘     │
│                       │                                    │        │
│            Gửi dữ liệu lên                     Nhận lệnh suy hao  │
│            (mỗi 500ms)                          (ký tự kết thúc'd')│
│                       │                                    │        │
│              ┌────────┴────────┐              ┌────────────┴──────┐ │
│              │                 │              │                   │ │
│       ┌──────┴──────┐   ┌─────┴─────┐   ┌────┴────────────────┐  │ │
│       │  LM75 (I2C) │   │ ADC3      │   │ PE4302 (SPI1)      │  │ │
│       │  Nhiệt độ   │   │ Công suất │   │ Suy hao 0-31.5 dB  │  │ │
│       └─────────────┘   └───────────┘   └─────────────────────┘  │ │
└──────────────────────────────────────────────────────────────────────┘
```

---

## 6. Kết luận

Thuật toán phần mềm K31-TTVT được thiết kế theo kiến trúc đa luồng, đảm bảo khả năng giám sát thời gian thực các thông số thiết bị đồng thời cho phép điều khiển từ xa mức suy hao tín hiệu. Việc phân chia rõ ràng giữa nhánh giám sát (nhận dữ liệu liên tục) và nhánh điều khiển (gửi lệnh theo yêu cầu) giúp hệ thống hoạt động ổn định, đáp ứng đầy đủ yêu cầu vận hành của khối thu phát vệ tinh trong thực tế.
