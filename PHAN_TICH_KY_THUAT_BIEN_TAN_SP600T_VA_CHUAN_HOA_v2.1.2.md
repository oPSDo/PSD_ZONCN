# TÀI LIỆU PHÂN TÍCH KỸ THUẬT BIẾN TẦN ZONCN SP600T & QUY CHUẨN HOÁ BẢN v2.1.2

---

## 1. TỔNG QUAN VÀ BỐI CẢNH SỰ CỐ
- **Đối tượng:** Biến tần ZONCN SP600T chuyên dụng cho máy nén khí trục vít (sử dụng động cơ đồng bộ nam châm vĩnh cửu PM Motor / động cơ không đồng bộ Asynchronous).
- **Hiện tượng thực tế:** 
  1. Khi máy nén khí đang chạy nạp/xả một lúc thì đột ngột dừng và biến tần hiện đèn vàng **FAULT**.
  2. Khi công nhân tắt aptomat đột ngột hoặc bấm nút Dừng khẩn cấp (Emergency Stop), biến tần rơi vào trạng thái khoá lỗi (đèn vàng FAULT). Người dùng phải ngắt điện chờ xả tụ lâu mới bật lại được, hoặc trước đây trên màn hình HMI 600T bấm vào nút **Study (Học tập)** thì đèn vàng FAULT mới tắt và máy hoạt động bình thường trở lại.
  3. Màn hình HMI phần cứng đã bị hỏng, toàn bộ việc giám sát và điều khiển chuyển qua phần mềm máy tính (PC App). Do đó, PC App cần có đầy đủ tính năng như HMI 600T (đặc biệt là màn hình **Motor Debug Function** với nút **Study**).

---

## 2. PHÂN TÍCH KỸ THUẬT TỪ TÀI LIỆU `TaiLieu.pdf` & DỮ LIỆU ĐỌC MODBUS THỰC TẾ

### 2.1. Cơ chế đo lường và bảo vệ điện áp / dòng điện
1. **Thanh ghi Bus DC (`0x2002`):**
   - Biến tần nắn dòng điện xoay chiều 3 pha (380V AC) qua cầu diode/chỉnh lưu để tạo ra điện áp Bus một chiều (DC Bus Voltage).
   - Công thức chuẩn lý thuyết và thực nghiệm:
     $$V_{\text{DC}} = \sqrt{2} \times V_{\text{AC}} \approx 1.414 \times V_{\text{AC}}$$
     $$\Rightarrow V_{\text{AC}} = \frac{V_{\text{DC}}}{\sqrt{2}} \approx \frac{V_{\text{DC}}}{1.414}$$
   - Ở lưới điện 380V AC bình thường, $V_{\text{DC}}$ dao động từ 530V đến 560V. Khi đó điện áp pha-pha tính toán là $\approx 375\text{V} - 395\text{V}$ AC.
2. **Vấn đề thanh ghi `0x2008`:**
   - Trong firmware ZONCN SP600T, thanh ghi `0x2008` trên thực tế đọc được giá trị raw là `1141`.
   - Nếu phần mềm chia scale 10 sẽ ra `114.1 V`. Ở các phiên bản trước, code kiểm tra:
     ```python
     if v_grid > 50.0 and v_grid < v_low_setting (325V):
         threading.Thread(target=self.cmd_stop).start()
     ```
   - **Hậu quả nghiêm trọng:** Phần mềm PC ngộ nhận điện áp lưới bị tụt dưới 325V và liên tục gửi lệnh `cmd_stop()` dừng cưỡng bức máy khi động cơ đang quay tải nặng nạp/xả!
   - Việc ngắt động cơ đột ngột giữa chu trình nạp xả tạo ra xung dòng điện cảm ứng và áp ngược cực lớn, khiến biến tần tự chốt cờ lỗi phần cứng và bật **đèn vàng FAULT**.
3. **Quy chuẩn bảo vệ:**
   - Biến tần SP600T là thiết bị công nghiệp cao cấp đã được tích hợp sẵn bảo vệ phần cứng siêu tốc (phản hồi trong micro-giây) với các mã lỗi:
     - `LU` (Low Voltage - Điện áp thấp)
     - `oU` (Over Voltage - Quá áp)
     - `oC` (Over Current - Quá dòng)
     - `OH` (Over Heat - Quá nhiệt IGBT)
   - **Kết luận kiến trúc:** Phần mềm máy tính (PC App) **KHÔNG ĐƯỢC TỰ Ý GỬI LỆNH DỪNG MÁY `cmd_stop()`** dựa trên suy diễn điện áp tính toán từ xa. Mọi hành vi bảo vệ ngắt máy phải để phần cứng biến tần tự thực thi. PC App chỉ có nhiệm vụ đọc trạng thái, cảnh báo trực quan cho người vận hành.

---

## 3. GIẢI MÃ TÍNH NĂNG "MOTOR DEBUG FUNCTION" (MÀN HÌNH HỌC TẬP & CÀI ĐẶT ĐỘNG CƠ)

Từ ảnh chụp thực tế màn hình simulator HMI 600T (`media_1790755290125.png`) và đối chiếu với bảng tham số nhóm P1 trong `TaiLieu.pdf`, ta có quy chuẩn 100% chính xác:

### 3.1. Danh mục 15 tham số cài đặt động cơ (Nhóm P1)
| STT | Tên trên màn hình HMI | Mã tham số | Địa chỉ Modbus (Hex) | Scale | Đơn vị | Ý nghĩa kỹ thuật |
|:---:|:---|:---:|:---:|:---:|:---:|:---|
| 1 | **Max Freq** | P1.19 | `0x0077` | 0.01 (x100) | Hz | Tần số tối đa của động cơ |
| 2 | **Up Limit Freq** | P1.05 | `0x0069` | 0.01 (x100) | Hz | Giới hạn tần số trên |
| 3 | **LowLimit Freq** | P1.06 | `0x006A` | 0.01 (x100) | Hz | Giới hạn tần số dưới |
| 4 | **Motor Type** | P1.20 | `0x0078` | 1 | - | Loại động cơ (0: KĐB, 2: PM Đồng bộ) |
| 5 | **Rated Power** | P1.21 | `0x0079` | 0.1 (x10) | kW | Công suất định mức |
| 6 | **Rated V** | P1.22 | `0x007A` | 1 | V | Điện áp định mức |
| 7 | **Rated Current** | P1.23 | `0x007B` | 0.01 (x100) | A | Dòng điện định mức |
| 8 | **Rated Freq** | P1.24 | `0x007C` | 0.01 (x100) | Hz | Tần số định mức |
| 9 | **Rated Speed** | P1.25 | `0x007D` | 1 | rpm | Tốc độ quay định mức (vòng/phút) |
| 10 | **Back EMF** | P1.26 | `0x007E` | 1 | V | Sức điện động cảm ứng ngược |
| 11 | **Acc Time** | P1.07 | `0x006B` | 0.01 (x100) | S | Thời gian tăng tốc |
| 12 | **Dec Time** | P1.08 | `0x006C` | 0.01 (x100) | S | Thời gian giảm tốc |
| 13 | **Rs** | P1.31 | `0x0083` | 0.001 hoặc 1 | mΩ | Điện trở stator động cơ |
| 14 | **Ld** | P1.32 | `0x0084` | 0.01 hoặc 1 | mH | Điện cảm trục d |
| 15 | **Lq** | P1.33 | `0x0085` | 0.01 hoặc 1 | mH | Điện cảm trục q |

### 3.2. Chức năng 5 nút điều khiển trên màn hình Motor Debug
1. **Nút `[Study]` (Tự học tham số động cơ & Xoá lỗi đèn vàng FAULT):**
   - **Cơ chế:** Ghi giá trị `1` vào thanh ghi tham số **P1.30** (Địa chỉ Modbus `0x0082`).
   - `P1.30 = 1`: Tự học tĩnh (Static Auto-tune). Biến tần bơm dòng xung tần số cao để đo điện trở và điện cảm cuộn dây mà **không làm quay trục vít**, đồng thời hiệu chuẩn lại góc từ thông của nam châm vĩnh cửu.
   - **Tác dụng cốt lõi:** Reset toàn bộ chốt lỗi cảnh báo xung đột góc pha, dập tắt ngay lập tức đèn vàng **FAULT**, đưa biến tần về trạng thái sẵn sàng hoạt động (Ready).
2. **Nút `[Jog On]` & `[Jog Off]`:**
   - Lệnh chạy thử nhấp (JOG) theo tần số Jog (P1.09 / F1.17) và lệnh dừng Jog.
3. **Nút `[Fan On]` & `[Fan Off]`:**
   - Điều khiển quạt làm mát của biến tần/máy nén khí qua tham số F2.30 hoặc relay điều khiển quạt.

---

## 4. QUY TRÌNH THỰC HIỆN BẢN v2.1.2 TỪ GỐC v2.1.1
1. **Sao chép:** Copy trọn vẹn `Source_v2.1.1` $\rightarrow$ `Source_v2.1.2`.
2. **Sửa lỗi tính áp & dừng máy:**
   - Trong `zoncn_protocol.py`: Loại bỏ hoàn toàn `cmd_stop()` tự động khi kiểm tra điện áp lưới.
   - Tính điện áp lưới AC hiển thị chuẩn: $V_{\text{AC}} = V_{\text{bus}} / 1.414$.
3. **Tích hợp màn hình Motor Debug Function:**
   - Tạo cửa sổ / tab "Motor Debug Function" theo đúng layout của ảnh HMI: 15 ô nhập/hiển thị thông số + 5 nút chức năng (`Study`, `Jog On`, `Jog Off`, `Fan On`, `Fan Off`) + nút `Return`.
   - Kết nối trực tiếp hàm Modbus đọc và ghi các thanh ghi nhóm P1 tương ứng.
4. **Bảo toàn cơ chế Fmax, Fmin:**
   - Giữ nguyên cấu trúc gán và đọc $F_{\max}, F_{\min}$ từ cài đặt của bản v2.1.1.
5. **Biên dịch PyInstaller:**
   - Tạo file spec và script build xuất ra file thực thi duy nhất `PSD_ZONCN_v2.1.2.exe`.
