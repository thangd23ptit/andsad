# STM32F103C8T6 — Cảnh báo lửa và MQ-2 với FreeRTOS

Firmware C cho Blue Pill STM32F103C8T6, Flash 64 KiB và RAM 20 KiB.
Build bằng Arm GNU Toolchain và GNU Make. Thư mục Lib đã chứa mã cần để build offline.

## Cấu trúc

| Vị trí | Nội dung |
|---|---|
| User/src/main.c | Tạo 4 queue, tạo 5 task và gọi vTaskStartScheduler |
| User/src/task_sensors.c | Lấy ADC mỗi 50 ms, đọc DO lửa và gửi mẫu mỗi 10 ms |
| User/src/task_buttons.c | Chống dội 35 ms và gửi sự kiện nhấn nút |
| User/src/task_control.c | Xử lý lệnh và ngưỡng, điều khiển LED/còi/relay |
| User/src/task_oled.c | Hiển thị lên SSD1306 128×64 |
| User/src/task_uart.c | Nhận lệnh từ ESP32 và báo trạng thái mỗi giây |
| User/inc | Header của từng task, app.h và FreeRTOSConfig.h |
| MyLib | Driver GPIO, ADC, USART1 và SSD1306 |
| Lib | FreeRTOS Kernel, CMSIS Core, CMSIS Device STM32F1 |
| Startup, Linker | Mã khởi động và phân vùng Flash/RAM |
| Output | Firmware ELF, HEX, BIN |
| Makefile | Lệnh build và nạp qua ST-LINK |

## Build và nạp trên Windows

Cài Arm GNU Toolchain bản `arm-none-eabi`, GNU Make và STM32CubeProgrammer.
Thêm các thư mục chứa chương trình vào PATH. Mở **Git Bash** trong thư mục chứa Makefile
để có các lệnh `mkdir` và `rm` dùng trong Makefile. Từ Git CMD có thể gõ
`bash` để vào Bash, sau đó chạy:

```bash
arm-none-eabi-gcc --version
make --version
make -j4
make flash
```

`make flash` dùng `STM32_Programmer_CLI` qua ST-LINK/SWD.
Đấu PA13 ↔ SWDIO, PA14 ↔ SWCLK, GND ↔ GND, và nối VTref tới mức 3,3 V của bo
nếu đầu nạp có chân VTref. Cấp nguồn cho bo và để BOOT0 ở 0.
Có thể mở STM32CubeProgrammer và chọn file `Output/STM32_CanhBao.hex` để nạp trực tiếp.
Nếu nạp file BIN, đặt địa chỉ đầu Flash `0x08000000`.
Nếu muốn build lại từ đầu, chạy `make clean` rồi `make -j4`.

## Chân nối

| Chân STM32 | Chức năng |
|---|---|
| PA0 | MQ-2 AO qua chia áp 12 kΩ phía AO và 18 kΩ xuống GND |
| PA1 | Flame DO mức logic 3,3 V, tác động LOW |
| PB6 / PB7 | OLED SCL / SDA, I²C mềm, địa chỉ 0x3C |
| PB0 | LED đỏ qua 330 Ω |
| PB1 | IN/S của module buzzer active có driver tích hợp và tương thích logic 3,3 V |
| PB10 | Tín hiệu điều khiển transistor kéo IN của relay module xuống LOW |
| PB12 / PB13 | Nút cảnh báo / nút quạt, nhấn nối chân xuống GND |
| PA9 / PA10 | USART1 TX → GPIO16 ESP32, RX ← GPIO17 ESP32 |

Nối GND STM32, ESP32 và phần tín hiệu các module. MQ-2 module cấp 5 V.
OLED cấp 3,3 V, SDA và SCL cần pull-up lên 3,3 V. Nếu module chưa tích hợp,
có thể dùng 4,7 kΩ mỗi đường. Tránh đưa AO hoặc DO 5 V trực tiếp vào PA0/PA1.
Dùng relay và quạt DC cho bài thực hành. PB10 điều khiển transistor của
relay như trong bảng.

### Module buzzer active 3 chân

- **IN/S → PB1**, nếu đầu vào tương thích GPIO 3,3 V và không kéo PB1 lên 5 V.
- **GND → GND chung** với STM32.
- **VCC → nguồn đúng theo thông số module**. PB1 chỉ nối chân tín hiệu.
- Module có driver tích hợp phù hợp có thể đấu trực tiếp như trên.
- Mặc định `BUZZER_ACTIVE_HIGH = 1` trong `MyLib/board.h`. PB1 HIGH thì
  còi kêu. Nếu module kích LOW, đặt thành `0` rồi build và nạp lại.
  Cấu hình này áp dụng cả khi khởi tạo GPIO và khi chương trình báo lỗi.
- Nếu cần giữ còi im khi STM32 còn reset, dùng module có sẵn điện trở định mức
  tắt hoặc thêm điện trở kéo phù hợp với mạch IN (kéo xuống cho kích HIGH,
  kéo lên 3,3 V cho kích LOW nếu mạch IN hỗ trợ).

Cấu hình gốc dùng HSI 8 MHz. Khi điều chỉnh mạch:
- Đổi mức phát hiện lửa trong `board_flame()` tại `MyLib/board.c`.
- Đổi địa chỉ OLED bằng macro `OLED_ADDRESS` trong `MyLib/ssd1306.c`.

## Logic và UART

Ngưỡng gốc 1800 ADC, trễ 3 giây. ADC phải duy trì tại hoặc trên ngưỡng
trong thời gian đếm. Sau khi đã kích hoạt, cảnh báo MQ-2 tắt khi ADC xuống
đến ngưỡng hồi (thấp hơn ngưỡng đặt 80 ADC, tối thiểu 0).
Lửa bật còi/LED và khóa quạt. Bật thủ công là yêu cầu bổ sung.
Tắt yêu cầu thủ công đưa đầu ra về logic tự động.

UART 115200, 8N1. ESP32 có thể dùng firmware Blynk đã có.
Lệnh ASCII kết thúc bằng LF hoặc CRLF:
- TH=1800: ngưỡng từ 1 đến 4095.
- DE=3: thời gian trễ từ 0 đến 60 giây.
- AL=1 và FA=1: bật yêu cầu cảnh báo và quạt thủ công. Giá trị 0 tắt yêu cầu.

Dòng phản hồi:
`S,adc,lua,canh_bao,quat,canh_bao_thu_cong,quat_thu_cong,nguong,tre`.

## Kiểm tra mạch

1. Ở trạng thái bình thường, bấm lần lượt hai nút để thử các đầu ra.
2. Đặt ngưỡng thấp hơn ADC hiện tại để thử bộ đếm trễ và relay.
3. Với cấu hình flame kích LOW, tháo dây DO rồi nối PA1 xuống GND để giả lập lửa.
   LED/còi phải bật và quạt phải tắt. Gỡ dây thử rồi nối lại DO.
4. Thử Blynk và kiểm tra V0, V1, V2, V3, V8. Ngắt Wi-Fi để kiểm tra hoạt động tại chỗ.

Bản này đã biên dịch và liên kết cho Cortex-M3. Các bước nạp và chạy
với linh kiện thực tế cần thực hiện trên bo của bạn.

## Nguồn thư viện

- FreeRTOS-Kernel V11.2.0, commit 0adc196d4bd52a2d91102b525b0aafc1e14a2386.
- ST cmsis_device_f1, commit c8e9a4a4f16b6d2cb2a2083cbe5161025280fb22.
  Tên thư mục `stm32f10x_lib` theo bố cục mẫu. Nội dung là CMSIS Device F1.
- Arm CMSIS_5 5.9.0, commit 2b7495b8535bdcb306dac29b9ded4cfb679d7e5c.

Các file giấy phép nằm trong từng thư mục thư viện. Startup lấy từ
gói CMSIS Device F1 của ST. Tham chiếu thanh ghi STM32F1 theo ST RM0008.
