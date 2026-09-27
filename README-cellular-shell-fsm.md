# Shell FSM của cellular modem — OMAN-MT212V2_NLR-app

Tài liệu này theo target [`OMAN-MT212V2_NLR-app.uvprojx`](uvproj/OMAN-MT212V2_NLR-app.uvprojx), với `CONFIG_PORT_GPRS`, `CONFIG_CUSTOMER_OMAN`, `CONFIG_HARDWARE_QUECTEL=1`, `CONFIG_HARDWARE_QUECTEL_EG91=1`, `CONFIG_FEATURE_LTE=1`, `CONFIG_FEATURE_CATM1_NBIOT=0`. Shell chạy trong Application/NLR, không thuộc firmware đo lường MTR. Nguồn chính: [`ShellHost.cpp`](src/Driver/USB_Modem/Src/ShellHost.cpp), [`ShellState.h`](src/Driver/USB_Modem/Inc/ShellState.h), [`ShellState.cpp`](src/Driver/USB_Modem/Src/ShellState.cpp). Các hằng và nhánh thay đổi theo cấu hình sản phẩm.

## Mô hình thực thi

### Shell đảm nhiệm gì và tác động gì xuống modem?

Shell là FSM **quản lý modem và nền mạng**: chuyển cấu hình COSEM/APN/radio thành AT command, theo dõi đăng ký mạng, mở/đóng PDP, đọc thông tin phục vụ diagnostic và thực hiện tác vụ quản trị. Socket push và phiên DLMS được Client/Server vận hành trên nền mạng đó. Shell tạo lệnh trong `ProcessTimer()`, xử lý response trong `ProcessRx()`, và tiếp nhận yêu cầu thao tác trong `ProcessRxCmd()`; các callback cập nhật dữ liệu modem/SIM/mạng về Application.

| State lớn | Nhiệm vụ và thao tác trong code | Thao tác xuống modem |
| --- | --- | --- |
| `INIT` | Đặt cấu hình radio/APN, đăng ký mạng, đọc định danh, giữ substate READY | `ATE0`, `AT+CFUN=0/4/1`, `AT+QICSGP`, `AT+QCFG="nwscanmode"`, `AT+QCFG="band"`, `AT+COPS`, `AT+CMEE`, `AT+CEREG`/`CGREG`, `ATI1`, `AT+CIMI`, `AT+CNUM`, `AT+CGSN`, `AT+CCID?` theo nhánh |
| `GET_MODEM_STATISTICS` | Thu operator, mức sóng, attach, cell lân cận, trạng thái socket; cập nhật diagnostic | `AT+COPS?`, `AT+CSQ`, `AT+CGREG?`/`CREG?`, `AT+CGATT?`, `AT+QENG="servingcell"`/`"neighbourcell"`, `AT+QISTATE`; Sierra có `XMETRIC`/`XCELLINFO` |
| `GET_LTE_MONITORING` | Đọc các chỉ số LTE, cập nhật cấu trúc monitoring | `AT+QCSQ`, `AT+QENG` và các truy vấn LTE tùy modem; các bước `QCFG="celevel"/"emmtimer"/"sibinfo"`, `CEREG=4` thuộc nhánh tính năng tương ứng, không mặc định toàn bộ đều chạy ở EG91 |
| `GET_LTE_CONNECTION_REJECTION` | Đọc và lưu nguyên nhân mạng từ chối | `AT+CEER` hoặc `AT+QCFG="emmcause"`, đổi định dạng/truy vấn operator bằng `AT+COPS` theo nhánh |
| `OPEN_PDP_CONTEXT` | Chuẩn bị APN/profile, activate context, lấy IP, đánh thức Client/Server theo yêu cầu pending | Quectel: `AT+QICSGP`, `AT+QIACT=1`, `AT+QIACT?`; Sierra: `CGDCONT`, `CGACT`, `KCNXCFG`, `KCNXUP`, `KCGPADDR` theo substate |
| `CLOSE_PDP_CONTEXT` | Deactivate context, xóa/cập nhật trạng thái IP và kết nối | Quectel `AT+QIDEACT=1`; nhánh khác `AT+CGACT=0`/`AT+KCNXDOWN` |
| `CLOSE_LTE_TCP_SESSION` | Đóng session ID do Client/Server yêu cầu, xử lý trạng thái kết thúc | Quectel `AT+QICLOSE=<id>`; Sierra `AT+KTCPCLOSE`/`AT+KTCPDEL` |
| `SET_LTE_MUTE` | Quản lý thao tác LTE mute và trả trạng thái mute | `ProcessTimer()` thực tế gửi `AT+KLTEMUTE=1,20,1` mà không có `#if` loại Quectel tại lệnh này. Vì vậy nếu state được yêu cầu ở EG91 thì code vẫn phát lệnh; khả năng modem chấp nhận phải xác minh riêng |
| `SET_PIN` | Đổi PIN từ old/new PIN, trả diagnostic thành công hoặc lỗi | `AT+CPWD="SC","<old>","<new>"` |
| `SET_RETRANS_CFG` | Nạp cấu hình TCP retransmission/minimum RTO | `AT+QICFG="tcp/minrto"`, `AT+QICFG="tcp/retranscfg"` |
| `GET_LTE_NTP_CLOCK` | Đồng bộ thời gian từ NTP, đọc lại clock và cập nhật thời gian ứng dụng | Quectel `AT+QNTP=1,"<server>",<port>`, `AT+CCLK?` |
| `START_FOTA_TRANSFER` | Khởi động tải firmware modem, theo dõi tiến độ và cập nhật trạng thái image transfer | Quectel `AT+QFOTADL="<URL>"`, nhận URC FOTA; Sierra có các bước `WDSI`/`WDSS`/`WDSW`. Lệnh cụ thể phụ thuộc nhánh |
| `ERASE_FPLMN_LIST` | Đọc/ghi SIM để xóa danh sách mạng bị cấm nếu tính năng bật | `AT+CSIM` |
| Các state `GET_NETWORK_OPERATOR_*`, `GET_MAIN_CELL_INFO`, `GET_NEIGHBOUR_CELL_INFO` | Tác vụ bổ sung của EDISTRIBUZIONE | `AT+COPS=?`/`COPS?`, `AT+QENG`; không thuộc nhánh OMAN mặc định |
| `DISABLE_FOTA_TRANSFER` | Dừng/giải phóng FOTA của HL7690 | Các lệnh Sierra `AT+WDSS`; không phải state EG91 |

**GPIO và reset:** các lớp ShellState không trực tiếp ghi GPIO power/reset. Khi init không nhận response, Shell yêu cầu `C_ServerHost::ResetHard()`; tác động GPIO diễn ra sau đó trong ServerHost → ModemHost → driver. `SHELL_HOST_MSG_RESET` chỉ reset phần mềm Shell và timer; bản thân message này không chứng minh modem đã bị cắt nguồn. Chi tiết pin và chuỗi hard reset nằm ở [phần điều khiển phần cứng](README-cellular-client-server-fsm.md#điều-khiển-gpio-powerreset-và-khởi-động-cmux).

`C_ShellHost::MsgProcess()` lấy message từ queue rồi gọi `ProcessRxFromCmux`, `ProcessRx`, `ProcessRxCmd`, `ProcessTimer` hoặc `ProcessTxComplete` trên đối tượng FSM hiện tại. Giá trị `E_SHELL_STATE` trả về thay thế trạng thái hiện tại; khi thay đổi, host gọi `Reset()` trên trạng thái cũ. Bên trong mỗi trạng thái lớn có `m_u08SubState`. `GotoSubState()` xóa số lần thử và cờ chờ phản hồi, dừng timer cũ, chọn độ trễ rồi chạy timer mới. `SendDataWaitResponse()` gửi AT qua `ModemAdaptor` và đánh dấu đang chờ response. UART RX đi qua CMUX → `ModemAdaptor` → `ModemDataParser`, sau đó quay về host bằng message RX/URC. Mỗi timer message mang ID; ID cũ hơn `m_u32CurrentTimerID` bị bỏ qua. Xem [`ModemAdaptor.cpp`](src/Driver/USB_Modem/Src/ModemAdaptor.cpp) và [`Cmux_Enexis.cpp`](src/Driver/Cmux/Src/Cmux_Enexis.cpp).

![State machine Shell](documentation/diagrams/cellular/shell-fsm.svg)

[Bản draw.io có thể chỉnh sửa](documentation/diagrams/cellular/shell-fsm.drawio)

Sơ đồ gộp các thao tác phụ; danh sách trạng thái đầy đủ nằm trong `E_SHELL_STATE` của `ShellState.h`. Các nhánh `GET_NETWORK_OPERATOR_*`, `GET_MAIN_CELL_INFO`, `GET_NEIGHBOUR_CELL_INFO` chỉ biên dịch khi bật `CONFIG_FEATURE_EDISTRIBUZIONE`. `DISABLE_FOTA_TRANSFER` chỉ cho Sierra HL7690.

## Khởi tạo và đăng ký mạng

Các bước dưới đây là **substate của `SHELL_STATE_INIT`**, không phải state ngang cấp. FSM chỉ đi tiếp khi nhận response được chấp nhận và `m_bWaitResponse` đang bật; lệnh timeout sẽ chạy cơ chế retry/reset. Trình tự phụ thuộc phản hồi và cấu hình, nên đây là đường chính cho target OMAN, không phải chuỗi AT cố định cho mọi tình huống.

| Substate / nhóm | AT hoặc tác vụ | Điều kiện chuyển tiếp |
| --- | --- | --- |
| `WAIT_STARTUP` → `LOCAL_ECHO_OFF` | Khởi tạo mute/PDP; gửi `ATE0` | Timer khởi động; `OK` cho `ATE0` → `UNREGISTER_FROM_THE_NETWORK` |
| `UNREGISTER_FROM_THE_NETWORK` | Đưa radio về chế độ cấu hình (`AT+CFUN=0`/`4` theo nhánh) | `OK` → `QICSGP` với Quectel |
| `QICSGP`, `SCAN_MODE`, `BAND_CFG` | Cấu hình APN/profile, scan mode/band của EG91 | `OK` chuyển lần lượt theo nhánh EG91 và `CONFIG_FEATURE_CATM1_NBIOT=0`; bỏ qua bước không áp dụng |
| `CONFIGURE_NETWORK_FORMAT`, `ENABLE_REPORTING_CME_ERRORS` | Định dạng nhà mạng, `AT+CMEE=1` | `OK` → cấu hình notification/radio; EG91 có độ trễ ngẫu nhiên trước `START_FULL_FUNCTIONALITY` |
| `ENABLE_*_NOTIFICATION`, `START_FULL_FUNCTIONALITY` | `AT+CREG`, `AT+CEREG` hoặc `AT+CGREG`, `AT+CGEREP`, `AT+CFUN=1` theo nhánh | `OK` → kiểm tra/đợi đăng ký mạng |
| `CHECK_NETWORK_REGISTRATION_STATUS` | `AT+COPS?` | Nếu trạng thái operator là `2`, gọi đăng ký lại; còn lại → `WAIT_NETWORK_REGISTRATION` |
| `REGISTER_TO_NETWORK` | `AT+COPS=0` | `OK` → `WAIT_NETWORK_REGISTRATION` |
| `WAIT_NETWORK_REGISTRATION` | Poll `AT+CEREG?` khi radio là 4G, `AT+CGREG?` ở mode khác của EG91 | Home/roaming → `ATI1`; đang tìm mạng → đợi 3 giây rồi poll lại; vượt giới hạn → bắt đầu lại từ `UNREGISTER`; bị từ chối → phát `CELL_MODEM_EVENT_NETWORK_REGISTRATION_FAILED` và đọc `CEER`; trạng thái khác poll lại, tối đa khoảng 100 chu kỳ |
| `ATI1`, `CIMI`, `MSISDN`, `IMEI`, `ICCID` | Đọc phiên bản modem và định danh SIM/modem | Response hợp lệ chuyển sang bước kế; cuối chuỗi → `READY`. Một số bước có thể đi tiếp khi thiếu dữ liệu tùy handler |
| `READY` | Shell sẵn sàng nhận yêu cầu ngoài luồng init | Message `SHELL_HOST_MSG_RX_CMD` kích hoạt state thao tác tương ứng |

Chi tiết điều kiện `CEREG`/`CGREG`, giới hạn tìm mạng và lệnh AT nằm trong [`C_ShellStateInit::ProcessRx/ProcessTimer`](src/Driver/USB_Modem/Src/ShellState.cpp). `+CME ERROR` trong bước `CEER` được đưa vào `ManageConnexionRetry()`, rồi luồng trở lại `UNREGISTER_FROM_THE_NETWORK`.

## Sự kiện làm đổi state lớn

| Sự kiện host | Điều kiện ở `INIT/READY` | State đích / kết quả |
| --- | --- | --- |
| `SHELL_HOST_MSG_RX_GET_LTE_MODEM_STATISTICS` | Shell đã READY | `GET_MODEM_STATISTICS`: đọc operator, tín hiệu, attach/cell/socket rồi cập nhật diagnostic |
| `SHELL_HOST_MSG_RX_GET_LTE_MONITORING` | READY, đủ thời gian trước kỳ mute | `GET_LTE_MONITORING`; nếu gần kỳ mute thì hoãn |
| `SHELL_HOST_MSG_RX_GET_LTE_CONNECTION_REJECTION` | READY, đủ thời gian trước mute | `GET_LTE_CONNECTION_REJECTION`: đọc nguyên nhân reject; gần mute thì hoãn |
| `SHELL_HOST_MSG_RX_GET_NTP_CLOCK_SYNC` | READY, NTP bật, đủ thời gian trước mute | `GET_LTE_NTP_CLOCK` |
| `SHELL_HOST_MSG_OPEN_PDP_CONTEXT` / `CLOSE_PDP_CONTEXT` | READY, đủ thời gian trước mute | `OPEN_PDP_CONTEXT` / `CLOSE_PDP_CONTEXT`; kết quả mở/đóng được thông báo lên host/ứng dụng |
| `SHELL_HOST_MSG_RX_SET_LTE_MUTE` | READY | `SET_LTE_MUTE` |
| `SHELL_HOST_MSG_PIN_UPDATE` | READY, quản lý PIN bật | `SET_PIN`; nhận kết quả `OK`/`ERROR`/`CME ERROR` rồi cập nhật diagnostic |
| `SHELL_HOST_MSG_RX_CLOSE_TCP_*_SESSION` | READY, đủ thời gian trước mute | `CLOSE_LTE_TCP_SESSION` |
| `SHELL_HOST_MSG_RX_START_FOTA_TRANSFER` | READY, đủ thời gian trước mute | `START_FOTA_TRANSFER`; chuỗi AT và sự kiện FOTA quyết định thành công/thất bại |
| `SHELL_HOST_MSG_SET_RETRANS_CFG` | READY, đủ thời gian trước mute | `SET_RETRANS_CFG` (Quectel) |
| `SHELL_HOST_MSG_RESET` | Bất kỳ state | Dừng timer, reset state hiện tại, về `INIT`; xử lý lại yêu cầu đang pending theo thứ tự trong `ResetProcess()` |
| `SHELL_HOST_MSG_RESET_IF_INIT_READY` | Chỉ khi state là `INIT` và substate `READY` | Reset tương tự; các substate khác bỏ qua |

**PDP của Quectel:** `OPEN_PDP_CONTEXT` kiểm tra/cấu hình profile (`AT+QICSGP` hoặc `AT+CGDCONT` theo nhánh), kích hoạt và kiểm tra địa chỉ (`AT+QIACT?`); `CLOSE_PDP_CONTEXT` đóng context và cập nhật modem status. `QUEC_PDPDEACT` là URC bất đồng bộ: ngay cả khi đang init, code phát `CELL_MODEM_EVENT_PDP_CONTEXT_DESTROYED` và yêu cầu lấy nguyên nhân reject. Xem các lớp `C_ShellStateOpenPDPContext` và `C_ShellStateClosePDPContext` trong `ShellState.cpp`.

## Timer, retry và điểm cần chú ý

`SHELL_STATE_TIMER_TOUCH_AND_GO=100 ms`; lệnh ngắn/dài, chọn operator, kích hoạt PDP và FOTA có timeout riêng trong `ShellState.h`. Sau khi phát AT, `m_u08NbTries` tăng; ở timer tiếp theo, nếu vượt `SHELL_STATE_TX_MAX_RETRIES` (bằng **0**), `SHELL_STATE_INIT` yêu cầu `C_ServerHost::ResetHard()` thay vì lặp AT vô hạn. Các state thao tác có xử lý lỗi/thoát riêng; cần xem handler của state đó trước khi kết luận mọi timeout đều reset modem. `GotoSubState()` đặt timer tối thiểu 100 ms. Yêu cầu được nhận gần kỳ LTE mute 300 giây thường bị hoãn theo điều kiện `elapsed < SHELL_STATE_PRE_MUTE_PERIOD_TICKS`, không lập tức chuyển state. Với các nhánh khác OMAN, đặc biệt Sierra hoặc Cat-M1/NB-IoT, nhiều bước và lệnh sẽ khác.

## Thứ tự thực thi Shell và phối hợp ba FSM

1. Bootstrap CMUX/SIM hoàn tất trên UART → Shell nhận bytes của kênh Shell. Timer `INIT/WAIT_STARTUP` → `LOCAL_ECHO_OFF`; timer kế tiếp mới phát `ATE0`.
2. Response `OK` → chọn substate kế → `GotoSubState()` đặt timer → khi timer message tới host thì `ProcessTimer()` mới phát AT tiếp. Như vậy chuyển substate và gửi lệnh AT là hai bước khác nhau, có khoảng chờ/scheduler ở giữa.
3. Shell lần lượt cấu hình radio/profile, đăng ký mạng và định danh → `INIT/READY`. Chưa READY, các yêu cầu mở client/server được giữ trong `u32GetRequestPending`; chúng không đồng nghĩa socket đã được mở.
4. `RX_WAKEUP` → `RxWakeUpProcess()` đặt pending PREPARE_CLIENT; `RX_START_SERVER` → `RxStartServer()` đặt pending OPEN_SERVER. Cả hai gọi `RxOpenPDPContext()` trước.
5. PDP `STARTUP` chọn nhánh theo lần đầu/mở server → `QICSGP` → `QIACT?` → `QIACT=1` nếu chưa active → `QIACT?` lấy IP. IP chưa có thì lặp activate; IP hợp lệ thì báo context opened và vào `PDP_END`.
6. Timer của `PDP_END` gọi `ShellHost::Reset()` **qua queue**. `ResetProcess()` đưa state lớn về INIT và chọn request pending; khi READY, `RxPrepareClient()` queue WAKEUP cho Client hoặc `RxOpenServer()` queue WAKEUP cho Server.
7. Sau khi Client/Server đóng socket, chúng yêu cầu Shell đóng PDP. Shell kiểm tra cả hai FSM đang INIT rồi mới gửi `QIDEACT=1`; nếu còn FSM hoạt động thì không thực hiện deactivation ở nhánh đó.
8. Nếu AT init/PDP timeout vượt retry, Shell queue hard reset sang ServerHost; ServerHost điều khiển ModemHost/driver power, rồi CMUX bootstrap và các host khởi tạo lại.

**Thứ tự ưu tiên pending** trong `ShellHost::ResetProcess()` là chuỗi `if/else if`, không phải FIFO: OPEN_PDP → PREPARE_CLIENT → OPEN_SERVER → CLOSE_TCP_SERVER → CLOSE_TCP_CLIENT → CLOSE_PDP → MUTE → PIN → CONNECTION_REJECTION → các thao tác operator/cell theo feature → NTP → diagnostic và các nhánh tiếp theo trong source. Mỗi lần xử lý chỉ chọn nhánh phù hợp đầu tiên. Điều kiện READY, mute và pending khiến lịch thực tế có thể xen kẽ monitoring với socket activity.

Chi tiết thứ tự gọi từ Application, sơ đồ sequence mở PDP/push và đường đi mỗi AT/response nằm ở [Client/Server — thứ tự gọi](README-cellular-client-server-fsm.md#thứ-tự-gọi-thực-tế-khởi-tạo-kết-nối-và-dữ-liệu). Ngoài ba FSM có **SMS FSM, CMUX, ModemHost, driver, adapter và parser**; bảng phạm vi target nằm ở [các thành phần tương tác modem](README-cellular-client-server-fsm.md#ngoài-shell-client-server-còn-ai-tương-tác-với-modem).

## Tra cứu chi tiết từng substate Shell theo handler

Một số tên chỉ xuất hiện trong header, không có `case` xử lý cùng tên trong `ShellState.cpp`: `SHELL_STATE_INIT_CONFIGURE_GPRS_CONNECTION`, `SHELL_STATE_INIT_WAIT_GPRS_NETWORK_REGISTRATION`, `SHELL_STATE_GET_OPERATOR_INFO`, `SHELL_STATE_GET_OPERATOR_INFO_END`. Không thể gán cho chúng một luồng chạy từ khai báo enum. Lớp operator-info thực tế dùng các tên `GET_LIST_OPERATOR`/`END_GET_LIST_OPERATOR` trong handler của nó. Các state phụ Sierra/FOTA/Cat-M1 trong bản tra cứu không thuộc đường EG91/OMAN mặc định.

Phần này liệt kê riêng từng substate trong source, bao gồm nhánh OMAN và các nhánh sản phẩm khác. Đây là bản tra cứu **source chưa tiền xử lý**: các lệnh/đích khác nhau dưới `#if` là lựa chọn thay thế, không phải đều được chạy liên tiếp. Phần phân tích OMAN phía trên xác định đường áp dụng. Điều kiện, response và đích được liệt kê theo từng handler; dùng liên kết source để xem cặp điều kiện–hành động và phạm vi `switch` chính xác. Các handler dùng chung như kiểm tra timer ID/retry ở đầu hàm vẫn áp dụng dù không lặp lại ở từng hàng. `ProcessRxFromCmux` chỉ chuyển bytes vào parser, nên không lặp lại ở đây.

### C_ShellStateInit

#### `SHELL_STATE_INIT_WAIT_STARTUP`

**ProcessRx** — [source L654](src/Driver/USB_Modem/Src/ShellState.cpp#L654).

- Giữ state hiện tại hoặc chỉ làm bước chờ/thoát của handler; không phát AT hay chuyển state tại nhánh này.

**ProcessRxCmd** — [source L1403](src/Driver/USB_Modem/Src/ShellState.cpp#L1403).

- Giữ state hiện tại hoặc chỉ làm bước chờ/thoát của handler; không phát AT hay chuyển state tại nhánh này.

**ProcessTimer** — [source L1738](src/Driver/USB_Modem/Src/ShellState.cpp#L1738).

- Đích chuyển state/substate có trong handler: `SHELL_STATE_INIT_LOCAL_ECHO_OFF`.

- Tác động/callback: `Update_LTE_mute_status`.

#### `SHELL_STATE_INIT_LOCAL_ECHO_OFF`

**ProcessRx** — [source L657](src/Driver/USB_Modem/Src/ShellState.cpp#L657).

- Response/URC và mã status được kiểm tra: `HL7690_OK`.

- Đích chuyển state/substate có trong handler: `SHELL_STATE_INIT_CIMI`, `SHELL_STATE_INIT_CHECK_SPEED_MODEM_MODE`, `SHELL_STATE_INIT_UNREGISTER_FROM_THE_NETWORK`.

- Các điều kiện kiểm tra: `m_bWaitResponse`, `HL7690_OK == pMsg->id`.

**ProcessRxCmd** — [source L1405](src/Driver/USB_Modem/Src/ShellState.cpp#L1405).

- Giữ state hiện tại hoặc chỉ làm bước chờ/thoát của handler; không phát AT hay chuyển state tại nhánh này.

**ProcessTimer** — [source L1748](src/Driver/USB_Modem/Src/ShellState.cpp#L1748).

- Lệnh gửi/tạo: `ATE0`.

- Timer được chọn: `SHELL_STATE_TIMER_SHORT_AT_COMMAND`.

#### `SHELL_STATE_INIT_CHECK_SPEED_MODEM_MODE`

**ProcessRx** — [source L680](src/Driver/USB_Modem/Src/ShellState.cpp#L680).

- Response/URC và mã status được kiểm tra: `QCFG_POW`.

- Đích chuyển state/substate có trong handler: `SHELL_STATE_INIT_UNREGISTER_FROM_THE_NETWORK`, `SHELL_STATE_INIT_SET_SPEED_MODEM_MODE`.

- Các điều kiện kiểm tra: `m_bWaitResponse`, `QCFG_POW == pMsg->id`, `status == 1`.

**ProcessRxCmd** — [source L1424](src/Driver/USB_Modem/Src/ShellState.cpp#L1424).

- Giữ state hiện tại hoặc chỉ làm bước chờ/thoát của handler; không phát AT hay chuyển state tại nhánh này.

**ProcessTimer** — [source L1757](src/Driver/USB_Modem/Src/ShellState.cpp#L1757).

- Lệnh gửi/tạo: `AT+QCFG="fast/poweroff"`.

- Timer được chọn: `SHELL_STATE_TIMER_SHORT_AT_COMMAND`.

#### `SHELL_STATE_INIT_SET_SPEED_MODEM_MODE`

**ProcessRx** — [source L695](src/Driver/USB_Modem/Src/ShellState.cpp#L695).

- Response/URC và mã status được kiểm tra: `HL7690_OK`.

- Đích chuyển state/substate có trong handler: `SHELL_STATE_INIT_UNREGISTER_FROM_THE_NETWORK`.

- Các điều kiện kiểm tra: `m_bWaitResponse`, `HL7690_OK == pMsg->id`.

**ProcessRxCmd** — [source L1425](src/Driver/USB_Modem/Src/ShellState.cpp#L1425).

- Giữ state hiện tại hoặc chỉ làm bước chờ/thoát của handler; không phát AT hay chuyển state tại nhánh này.

**ProcessTimer** — [source L1762](src/Driver/USB_Modem/Src/ShellState.cpp#L1762).

- Lệnh gửi/tạo: `AT+QCFG="fast/poweroff",1`.

- Timer được chọn: `SHELL_STATE_TIMER_SHORT_AT_COMMAND`.

#### `SHELL_STATE_INIT_QICSGP`

**ProcessRx** — [source L707](src/Driver/USB_Modem/Src/ShellState.cpp#L707).

- Response/URC và mã status được kiểm tra: `HL7690_OK`.

- Đích chuyển state/substate có trong handler: `SHELL_STATE_INIT_PSM`, `SHELL_STATE_INIT_SCAN_MODE`, `SHELL_STATE_INIT_CONFIGURE_NETWORK_FORMAT`.

- Các điều kiện kiểm tra: `m_bWaitResponse`, `HL7690_OK == pMsg->id`.

**ProcessRxCmd** — [source L1422](src/Driver/USB_Modem/Src/ShellState.cpp#L1422).

- Giữ state hiện tại hoặc chỉ làm bước chờ/thoát của handler; không phát AT hay chuyển state tại nhánh này.

**ProcessTimer** — [source L1769](src/Driver/USB_Modem/Src/ShellState.cpp#L1769).

- Lệnh gửi/tạo: `AT+QICSGP=1,1,"apn","%s","%s"`, `AT+QICSGP=1,1,"%s","%s","%s"`.

- Timer được chọn: `SHELL_STATE_TIMER_LONG_AT_COMMAND`.

- Các điều kiện kiểm tra: `0 == strlen((char *)(pModemConfig->APN_Name))`.

#### `SHELL_STATE_INIT_UNREGISTER_FROM_THE_NETWORK`

**ProcessRx** — [source L722](src/Driver/USB_Modem/Src/ShellState.cpp#L722).

- Response/URC và mã status được kiểm tra: `HL7690_OK`.

- Đích chuyển state/substate có trong handler: `SHELL_STATE_INIT_QICSGP`, `SHELL_STATE_INIT_DEFINE_PDP_CONTEXT`.

- Các điều kiện kiểm tra: `m_bWaitResponse`, `HL7690_OK == pMsg->id`.

**ProcessRxCmd** — [source L1406](src/Driver/USB_Modem/Src/ShellState.cpp#L1406).

- Giữ state hiện tại hoặc chỉ làm bước chờ/thoát của handler; không phát AT hay chuyển state tại nhánh này.

**ProcessTimer** — [source L1784](src/Driver/USB_Modem/Src/ShellState.cpp#L1784).

- Lệnh gửi/tạo: `AT+CFUN=0`, `AT+CFUN=4`.

- Timer được chọn: `SHELL_STATE_TIMER_SHORT_AT_COMMAND`.

#### `SHELL_STATE_INIT_DEFINE_PDP_CONTEXT`

**ProcessRx** — [source L735](src/Driver/USB_Modem/Src/ShellState.cpp#L735).

- Response/URC và mã status được kiểm tra: `HL7690_OK`.

- Đích chuyển state/substate có trong handler: `SHELL_STATE_INIT_ENABLE_REPORTING_CME_ERRORS`.

- Các điều kiện kiểm tra: `m_bWaitResponse`, `HL7690_OK == pMsg->id`.

**ProcessRxCmd** — [source L1407](src/Driver/USB_Modem/Src/ShellState.cpp#L1407).

- Giữ state hiện tại hoặc chỉ làm bước chờ/thoát của handler; không phát AT hay chuyển state tại nhánh này.

**ProcessTimer** — [source L1899](src/Driver/USB_Modem/Src/ShellState.cpp#L1899).

- Lệnh gửi/tạo: `AT+CGDCONT=1,"IP","%s"`.

- Timer được chọn: `SHELL_STATE_TIMER_FIVE_SECONDS`.

#### `SHELL_STATE_INIT_ENABLE_REPORTING_CME_ERRORS`

**ProcessRx** — [source L744](src/Driver/USB_Modem/Src/ShellState.cpp#L744).

- Response/URC và mã status được kiểm tra: `HL7690_OK`.

- Đích chuyển state/substate có trong handler: `SHELL_STATE_INIT_START_FULL_FUNCTIONALITY (delay GetRandomDelay()`, `SHELL_STATE_INIT_ENABLE_GPRS_NETWORK_NOTIFICATION`.

- Các điều kiện kiểm tra: `m_bWaitResponse`, `HL7690_OK == pMsg->id`.

**ProcessRxCmd** — [source L1408](src/Driver/USB_Modem/Src/ShellState.cpp#L1408).

- Giữ state hiện tại hoặc chỉ làm bước chờ/thoát của handler; không phát AT hay chuyển state tại nhánh này.

**ProcessTimer** — [source L1907](src/Driver/USB_Modem/Src/ShellState.cpp#L1907).

- Lệnh gửi/tạo: `AT+CMEE=1`.

- Timer được chọn: `SHELL_STATE_TIMER_SHORT_AT_COMMAND`.

#### `SHELL_STATE_INIT_CFG_URC_PORT`

**ProcessRx** — [source L760](src/Driver/USB_Modem/Src/ShellState.cpp#L760).

- Response/URC và mã status được kiểm tra: `HL7690_OK`.

- Đích chuyển state/substate có trong handler: `SHELL_STATE_INIT_ENABLE_REPORTING_CME_ERRORS`.

- Các điều kiện kiểm tra: `m_bWaitResponse`, `HL7690_OK == pMsg->id`.

**ProcessRxCmd** — [source L1421](src/Driver/USB_Modem/Src/ShellState.cpp#L1421).

- Giữ state hiện tại hoặc chỉ làm bước chờ/thoát của handler; không phát AT hay chuyển state tại nhánh này.

**ProcessTimer** — [source L1914](src/Driver/USB_Modem/Src/ShellState.cpp#L1914).

- Lệnh gửi/tạo: `AT+QCFG="cmux/urcport",1 `.

- Timer được chọn: `SHELL_STATE_TIMER_SHORT_AT_COMMAND`.

#### `SHELL_STATE_INIT_ENABLE_NETWORK_NOTIFICATION`

**ProcessRx** — [source L769](src/Driver/USB_Modem/Src/ShellState.cpp#L769).

- Response/URC và mã status được kiểm tra: `HL7690_OK`.

- Đích chuyển state/substate có trong handler: `SHELL_STATE_INIT_UNREGISTER_FROM_THE_NETWORK`.

- Các điều kiện kiểm tra: `m_bWaitResponse`, `HL7690_OK == pMsg->id`.

**ProcessRxCmd** — [source L1409](src/Driver/USB_Modem/Src/ShellState.cpp#L1409).

- Giữ state hiện tại hoặc chỉ làm bước chờ/thoát của handler; không phát AT hay chuyển state tại nhánh này.

**ProcessTimer** — [source L1920](src/Driver/USB_Modem/Src/ShellState.cpp#L1920).

- Lệnh gửi/tạo: `AT+CREG=2`.

- Timer được chọn: `SHELL_STATE_TIMER_SHORT_AT_COMMAND`.

#### `SHELL_STATE_INIT_ENABLE_GPRS_NETWORK_NOTIFICATION`

**ProcessRx** — [source L778](src/Driver/USB_Modem/Src/ShellState.cpp#L778).

- Response/URC và mã status được kiểm tra: `HL7690_OK`.

- Đích chuyển state/substate có trong handler: `SHELL_STATE_INIT_ENABLE_PACKET_DOMAIN_NOTIFICATION`, `SHELL_STATE_INIT_UNREGISTER_FROM_THE_NETWORK`, `SHELL_STATE_INIT_CONFIGURE_NETWORK_FORMAT`.

- Các điều kiện kiểm tra: `m_bWaitResponse`, `HL7690_OK == pMsg->id`.

**ProcessRxCmd** — [source L1410](src/Driver/USB_Modem/Src/ShellState.cpp#L1410).

- Giữ state hiện tại hoặc chỉ làm bước chờ/thoát của handler; không phát AT hay chuyển state tại nhánh này.

**ProcessTimer** — [source L1926](src/Driver/USB_Modem/Src/ShellState.cpp#L1926).

- Lệnh gửi/tạo: `AT+CEREG=1`, `AT+CGREG=1`, `AT+CREG=2`.

- Response/URC và mã status được kiểm tra: `QUEC_4G_NETWORK`, `QUEC_LTE_NETWORK_CAT_M1_AND_NB1`.

- Timer được chọn: `SHELL_STATE_TIMER_SHORT_AT_COMMAND`.

- Các điều kiện kiểm tra: `m_u08Network_Mode == QUEC_4G_NETWORK`, `m_u08CatM1_Nb1_Mode <= QUEC_LTE_NETWORK_CAT_M1_AND_NB1`.

#### `SHELL_STATE_INIT_ENABLE_PACKET_DOMAIN_NOTIFICATION`

**ProcessRx** — [source L797](src/Driver/USB_Modem/Src/ShellState.cpp#L797).

- Response/URC và mã status được kiểm tra: `HL7690_OK`.

- Đích chuyển state/substate có trong handler: `SHELL_STATE_INIT_CONFIGURE_NETWORK_FORMAT`.

- Các điều kiện kiểm tra: `m_bWaitResponse`, `HL7690_OK == pMsg->id`.

**ProcessRxCmd** — [source L1411](src/Driver/USB_Modem/Src/ShellState.cpp#L1411).

- Giữ state hiện tại hoặc chỉ làm bước chờ/thoát của handler; không phát AT hay chuyển state tại nhánh này.

**ProcessTimer** — [source L1948](src/Driver/USB_Modem/Src/ShellState.cpp#L1948).

- Lệnh gửi/tạo: `AT+CGEREP=2,1`.

- Timer được chọn: `SHELL_STATE_TIMER_LONG_AT_COMMAND`.

#### `SHELL_STATE_INIT_SCAN_MODE`

**ProcessRx** — [source L809](src/Driver/USB_Modem/Src/ShellState.cpp#L809).

- Response/URC và mã status được kiểm tra: `HL7690_OK`, `HL7690_ERROR`.

- Đích chuyển state/substate có trong handler: `SHELL_STATE_INIT_BAND_CFG`, `SHELL_STATE_INIT_CATM1_NBIOT`.

- Các điều kiện kiểm tra: `m_bWaitResponse`, `(HL7690_OK == pMsg->id) \|\| (HL7690_ERROR == pMsg->id)`.

**ProcessRxCmd** — [source L1432](src/Driver/USB_Modem/Src/ShellState.cpp#L1432).

- Giữ state hiện tại hoặc chỉ làm bước chờ/thoát của handler; không phát AT hay chuyển state tại nhánh này.

**ProcessTimer** — [source L1796](src/Driver/USB_Modem/Src/ShellState.cpp#L1796).

- Lệnh gửi/tạo: `AT+QCFG="nwscanmode",%i,1`, `AT+QCFG="nwscanmode",3,1`, `AT+QCFG="nwscanmode",1,1`.

- Response/URC và mã status được kiểm tra: `QUEC_ERROR_ACCEPTED`, `QUEC_LTE_NETWORK_CAT_M1_AND_NB1`.

- Timer được chọn: `SHELL_STATE_TIMER_LONG_AT_COMMAND`.

- Các điều kiện kiểm tra: `pModemConfig`, `m_u08CatM1_Nb1_Mode <= QUEC_LTE_NETWORK_CAT_M1_AND_NB1`.

#### `SHELL_STATE_INIT_PSM`

**ProcessRx** — [source L822](src/Driver/USB_Modem/Src/ShellState.cpp#L822).

- Response/URC và mã status được kiểm tra: `HL7690_OK`.

- Đích chuyển state/substate có trong handler: `SHELL_STATE_INIT_SCAN_MODE`.

- Các điều kiện kiểm tra: `m_bWaitResponse`, `HL7690_OK == pMsg->id`.

**ProcessRxCmd** — [source L1435](src/Driver/USB_Modem/Src/ShellState.cpp#L1435).

- Giữ state hiện tại hoặc chỉ làm bước chờ/thoát của handler; không phát AT hay chuyển state tại nhánh này.

**ProcessTimer** — [source L1864](src/Driver/USB_Modem/Src/ShellState.cpp#L1864).

- Lệnh gửi/tạo: `AT+CPSMS=%i`.

- Timer được chọn: `SHELL_STATE_TIMER_LONG_AT_COMMAND`.

#### `SHELL_STATE_INIT_CATM1_NBIOT`

**ProcessRx** — [source L829](src/Driver/USB_Modem/Src/ShellState.cpp#L829).

- Response/URC và mã status được kiểm tra: `HL7690_OK`.

- Đích chuyển state/substate có trong handler: `SHELL_STATE_INIT_BAND_CFG`.

- Các điều kiện kiểm tra: `m_bWaitResponse`, `HL7690_OK == pMsg->id`.

**ProcessRxCmd** — [source L1436](src/Driver/USB_Modem/Src/ShellState.cpp#L1436).

- Giữ state hiện tại hoặc chỉ làm bước chờ/thoát của handler; không phát AT hay chuyển state tại nhánh này.

**ProcessTimer** — [source L1874](src/Driver/USB_Modem/Src/ShellState.cpp#L1874).

- Lệnh gửi/tạo: `AT+QCFG="iotopmode",%i,1`, `AT+QCFG="iotopmode",0,1`.

- Response/URC và mã status được kiểm tra: `QUEC_LTE_NETWORK_CAT_M1_AND_NB1`.

- Timer được chọn: `SHELL_STATE_TIMER_LONG_AT_COMMAND`.

- Các điều kiện kiểm tra: `m_u08CatM1_Nb1_Mode <= QUEC_LTE_NETWORK_CAT_M1_AND_NB1`.

#### `SHELL_STATE_INIT_SET_RADIO_ACCESS_TECHNOLOGY`

**ProcessRx** — [source L840](src/Driver/USB_Modem/Src/ShellState.cpp#L840).

- Response/URC và mã status được kiểm tra: `HL7690_OK`.

- Đích chuyển state/substate có trong handler: `SHELL_STATE_INIT_CHECK_NETWORK_REGISTRATION_STATUS`.

- Các điều kiện kiểm tra: `m_bWaitResponse`, `HL7690_OK == pMsg->id`.

**ProcessRxCmd** — [source L1439](src/Driver/USB_Modem/Src/ShellState.cpp#L1439).

- Giữ state hiện tại hoặc chỉ làm bước chờ/thoát của handler; không phát AT hay chuyển state tại nhánh này.

**ProcessTimer** — [source L1892](src/Driver/USB_Modem/Src/ShellState.cpp#L1892).

- Lệnh gửi/tạo: `AT+KSRAT=9`.

- Timer được chọn: `SHELL_STATE_TIMER_SHORT_AT_COMMAND`.

#### `SHELL_STATE_INIT_BAND_CFG`

**ProcessRx** — [source L852](src/Driver/USB_Modem/Src/ShellState.cpp#L852).

- Response/URC và mã status được kiểm tra: `HL7690_OK`.

- Đích chuyển state/substate có trong handler: `SHELL_STATE_INIT_CONFIGURE_NETWORK_FORMAT`.

- Các điều kiện kiểm tra: `m_bWaitResponse`, `HL7690_OK == pMsg->id`.

**ProcessRxCmd** — [source L1433](src/Driver/USB_Modem/Src/ShellState.cpp#L1433).

- Giữ state hiện tại hoặc chỉ làm bước chờ/thoát của handler; không phát AT hay chuyển state tại nhánh này.

**ProcessTimer** — [source L1829](src/Driver/USB_Modem/Src/ShellState.cpp#L1829).

- Lệnh gửi/tạo: `AT+QCFG="band",%x,0,1`, `AT+QCFG="band",0,%x,1`, `AT+QCFG="band",%x,%x,1`.

- Response/URC và mã status được kiểm tra: `QUEC_2G_NETWORK`, `QUEC_EG95_ALL_BAND_GPRS`, `QUEC_4G_NETWORK`, `QUEC_EG95_ALL_BAND_LTE`, `QUEC_AUTOMATIC_NETWORK`.

- Timer được chọn: `SHELL_STATE_TIMER_LONG_AT_COMMAND`.

- Các điều kiện kiểm tra: `pModemConfig->Radio_Access_Technology == QUEC_2G_NETWORK`, `pModemConfig->Radio_Access_Technology == QUEC_4G_NETWORK`, `pModemConfig->Radio_Access_Technology == QUEC_AUTOMATIC_NETWORK`.

#### `SHELL_STATE_INIT_CONFIGURE_NETWORK_FORMAT`

**ProcessRx** — [source L860](src/Driver/USB_Modem/Src/ShellState.cpp#L860).

- Response/URC và mã status được kiểm tra: `HL7690_OK`.

- Đích chuyển state/substate có trong handler: `SHELL_STATE_INIT_ENABLE_REPORTING_CME_ERRORS`, `SHELL_STATE_INIT_START_FULL_FUNCTIONALITY (delay GetRandomDelay()`.

- Các điều kiện kiểm tra: `m_bWaitResponse`, `HL7690_OK == pMsg->id`.

**ProcessRxCmd** — [source L1412](src/Driver/USB_Modem/Src/ShellState.cpp#L1412).

- Giữ state hiện tại hoặc chỉ làm bước chờ/thoát của handler; không phát AT hay chuyển state tại nhánh này.

**ProcessTimer** — [source L1955](src/Driver/USB_Modem/Src/ShellState.cpp#L1955).

- Lệnh gửi/tạo: `AT+COPS=3,0`.

- Timer được chọn: `SHELL_STATE_TIMER_COPS_AT_COMMAND`.

#### `SHELL_STATE_INIT_START_FULL_FUNCTIONALITY`

**ProcessRx** — [source L874](src/Driver/USB_Modem/Src/ShellState.cpp#L874).

- Response/URC và mã status được kiểm tra: `HL7690_OK`.

- Đích chuyển state/substate có trong handler: `SHELL_STATE_INIT_CHECK_NETWORK_REGISTRATION_STATUS`, `SHELL_STATE_INIT_CHECK_NETWORK_REGISTRATION_STATUS (delay 5000)`, `SHELL_STATE_INIT_SET_RADIO_ACCESS_TECHNOLOGY`.

- Các điều kiện kiểm tra: `m_bWaitResponse`, `HL7690_OK == pMsg->id`.

**ProcessRxCmd** — [source L1413](src/Driver/USB_Modem/Src/ShellState.cpp#L1413).

- Giữ state hiện tại hoặc chỉ làm bước chờ/thoát của handler; không phát AT hay chuyển state tại nhánh này.

**ProcessTimer** — [source L1962](src/Driver/USB_Modem/Src/ShellState.cpp#L1962).

- Lệnh gửi/tạo: `AT+CFUN=1`.

- Timer được chọn: `SHELL_STATE_TIMER_SHORT_AT_COMMAND`.

#### `SHELL_STATE_INIT_CHECK_NETWORK_REGISTRATION_STATUS`

**ProcessRx** — [source L894](src/Driver/USB_Modem/Src/ShellState.cpp#L894).

- Response/URC và mã status được kiểm tra: `HL7690_COPS`, `HL7690_OK`.

- Đích chuyển state/substate có trong handler: `SHELL_STATE_INIT_REGISTER_TO_NETWORK`, `SHELL_STATE_INIT_WAIT_NETWORK_REGISTRATION`.

- Các điều kiện kiểm tra: `m_bWaitResponse`, `HL7690_COPS == pMsg->id`, `HL7690_OK == pMsg->id`, `status == 2`.

**ProcessRxCmd** — [source L1414](src/Driver/USB_Modem/Src/ShellState.cpp#L1414).

- Giữ state hiện tại hoặc chỉ làm bước chờ/thoát của handler; không phát AT hay chuyển state tại nhánh này.

**ProcessTimer** — [source L1968](src/Driver/USB_Modem/Src/ShellState.cpp#L1968).

- Lệnh gửi/tạo: `AT+COPS?`.

- Timer được chọn: `SHELL_STATE_TIMER_COPS_AT_COMMAND`.

#### `SHELL_STATE_INIT_REGISTER_TO_NETWORK`

**ProcessRx** — [source L912](src/Driver/USB_Modem/Src/ShellState.cpp#L912).

- Response/URC và mã status được kiểm tra: `HL7690_OK`.

- Đích chuyển state/substate có trong handler: `SHELL_STATE_INIT_WAIT_NETWORK_REGISTRATION`.

- Các điều kiện kiểm tra: `m_bWaitResponse`, `HL7690_OK == pMsg->id`.

**ProcessRxCmd** — [source L1415](src/Driver/USB_Modem/Src/ShellState.cpp#L1415).

- Giữ state hiện tại hoặc chỉ làm bước chờ/thoát của handler; không phát AT hay chuyển state tại nhánh này.

**ProcessTimer** — [source L1974](src/Driver/USB_Modem/Src/ShellState.cpp#L1974).

- Lệnh gửi/tạo: `AT+COPS=0`.

- Timer được chọn: `SHELL_STATE_TIMER_COPS_AT_COMMAND`.

#### `SHELL_STATE_INIT_WAIT_NETWORK_REGISTRATION`

**ProcessRx** — [source L921](src/Driver/USB_Modem/Src/ShellState.cpp#L921).

- Response/URC và mã status được kiểm tra: `QUEC_LTE_NETWORK_CAT_M1_AND_NB1`, `QUEC_CEREG`, `HL7690_CGREG`, `QUEC_4G_NETWORK`, `HL7690_CREG`, `HL7690_CREG_STATUS_REGISTERED_HOME_NETWORK`, `HL7690_CREG_STATUS_REGISTERED_ROAMING`, `HL7690_CREG_STATUS_NOT_REGISTERED_SEARCH_FOR_NEW_OPERATOR_STARTED`, `HL7690_CREG_STATUS_REGISTRATION_DENIED`.

- Đích chuyển state/substate có trong handler: `SHELL_STATE_INIT_ATI1`, `SHELL_STATE_INIT_UNREGISTER_FROM_THE_NETWORK`, `SHELL_STATE_INIT_WAIT_NETWORK_REGISTRATION (delay 3000)`, `SHELL_STATE_INIT_EMMCAUSE`, `SHELL_STATE_INIT_CEER`.

- Tác động/callback: `Cmux_Send_Modem_Event_To_App`.

- Các điều kiện kiểm tra: `m_bWaitResponse`, `(m_u08CatM1_Nb1_Mode <= QUEC_LTE_NETWORK_CAT_M1_AND_NB1 && QUEC_CEREG == pMsg->id) \|\| (m_u08CatM1_Nb1_Mode > QUEC_LTE_NETWORK_CAT_M1_AND_NB1 && HL7690_CGREG == pMsg->id)`, `(m_u08Network_Mode == QUEC_4G_NETWORK && QUEC_CEREG == pMsg->id) \|\| (m_u08Network_Mode != QUEC_4G_NETWORK && HL7690_CGREG == pMsg->id)`, `HL7690_CREG == pMsg->id`, `s32registrationStatus == HL7690_CREG_STATUS_REGISTERED_HOME_NETWORK \|\| s32registrationStatus == HL7690_CREG_STATUS_REGISTERED_ROAMING`, `s32registrationStatus == HL7690_CREG_STATUS_NOT_REGISTERED_SEARCH_FOR_NEW_OPERATOR_STARTED`, `m_u16waitState > SHELL_MAX_WAIT_SEARCH_FOR_NEW_OPERATOR_PERIOD`, `s32registrationStatus == HL7690_CREG_STATUS_REGISTRATION_DENIED`, `m_u16waitState > 100`.

**ProcessRxCmd** — [source L1416](src/Driver/USB_Modem/Src/ShellState.cpp#L1416).

- Giữ state hiện tại hoặc chỉ làm bước chờ/thoát của handler; không phát AT hay chuyển state tại nhánh này.

**ProcessTimer** — [source L1982](src/Driver/USB_Modem/Src/ShellState.cpp#L1982).

- Lệnh gửi/tạo: `AT+CEREG?`, `AT+CGREG?`, `AT+CREG?`.

- Response/URC và mã status được kiểm tra: `QUEC_4G_NETWORK`, `QUEC_LTE_NETWORK_CAT_M1_AND_NB1`.

- Timer được chọn: `SHELL_STATE_TIMER_SHORT_AT_COMMAND`.

- Các điều kiện kiểm tra: `m_u08Network_Mode == QUEC_4G_NETWORK`, `m_u08CatM1_Nb1_Mode <= QUEC_LTE_NETWORK_CAT_M1_AND_NB1`.

#### `SHELL_STATE_INIT_EMMCAUSE`

**ProcessRx** — [source L990](src/Driver/USB_Modem/Src/ShellState.cpp#L990).

- Response/URC và mã status được kiểm tra: `HL7690_OK`, `QCFG_EMMCAUSE1`, `QCFG_EMMCAUSE2`.

- Đích chuyển state/substate có trong handler: `SHELL_STATE_INIT_CEER`.

- Tác động/callback: `Update_LTE_connection_rejection`.

- Các điều kiện kiểm tra: `m_bWaitResponse`, `HL7690_OK == pMsg->id`, `QCFG_EMMCAUSE1 == pMsg->id`, `QCFG_EMMCAUSE2 == pMsg->id`.

**ProcessRxCmd** — [source L1418](src/Driver/USB_Modem/Src/ShellState.cpp#L1418).

- Giữ state hiện tại hoặc chỉ làm bước chờ/thoát của handler; không phát AT hay chuyển state tại nhánh này.

**ProcessTimer** — [source L2004](src/Driver/USB_Modem/Src/ShellState.cpp#L2004).

- Lệnh gửi/tạo: `AT+QCFG="emmcause"`.

- Timer được chọn: `SHELL_STATE_TIMER_LONG_AT_COMMAND`.

#### `SHELL_STATE_INIT_CEER`

**ProcessRx** — [source L1012](src/Driver/USB_Modem/Src/ShellState.cpp#L1012).

- Response/URC và mã status được kiểm tra: `HL7690_CME_ERROR`.

- Đích chuyển state/substate có trong handler: `SHELL_STATE_INIT_UNREGISTER_FROM_THE_NETWORK`.

- Tác động/callback: `ManageConnexionRetry`.

- Các điều kiện kiểm tra: `m_bWaitResponse`, `HL7690_CME_ERROR == pMsg->id`.

**ProcessRxCmd** — [source L1419](src/Driver/USB_Modem/Src/ShellState.cpp#L1419).

- Giữ state hiện tại hoặc chỉ làm bước chờ/thoát của handler; không phát AT hay chuyển state tại nhánh này.

**ProcessTimer** — [source L2010](src/Driver/USB_Modem/Src/ShellState.cpp#L2010).

- Lệnh gửi/tạo: `AT+CEER`.

- Timer được chọn: `SHELL_STATE_TIMER_LONG_AT_COMMAND`.

#### `SHELL_STATE_INIT_ATI1`

**ProcessRx** — [source L1028](src/Driver/USB_Modem/Src/ShellState.cpp#L1028).

- Response/URC và mã status được kiểm tra: `HL7690_OK`, `HL7690_ATI1_R`, `HL7690_ATI1`.

- Đích chuyển state/substate có trong handler: `SHELL_STATE_INIT_ATI3`, `SHELL_STATE_INIT_CIMI`, `SHELL_STATE_INIT_CMGF`, `SHELL_STATE_READY`.

- Tác động/callback: `Update_modem_version`, `Update_FU_LTE_modem_status`, `Update_modem_manufacture`.

- Các điều kiện kiểm tra: `m_bWaitResponse`, `HL7690_OK == pMsg->id`, `HL7690_ATI1_R == pMsg->id`, `pOldModemFirmwareVersion[0] != 0`, `strcmp((char *)pModemFirmwareVersion, (char *)pOldModemFirmwareVersion) == 0`, `HL7690_ATI1 == pMsg->id`.

**ProcessRxCmd** — [source L1417](src/Driver/USB_Modem/Src/ShellState.cpp#L1417).

- Giữ state hiện tại hoặc chỉ làm bước chờ/thoát của handler; không phát AT hay chuyển state tại nhánh này.

**ProcessTimer** — [source L2015](src/Driver/USB_Modem/Src/ShellState.cpp#L2015).

- Lệnh gửi/tạo: `ATI1`.

- Timer được chọn: `SHELL_STATE_TIMER_SHORT_AT_COMMAND`.

#### `SHELL_STATE_INIT_CIMI`

**ProcessRx** — [source L1154](src/Driver/USB_Modem/Src/ShellState.cpp#L1154).

- Response/URC và mã status được kiểm tra: `HL7690_CIMI`.

- Đích chuyển state/substate có trong handler: `SHELL_STATE_INIT_MSISDN`.

- Các điều kiện kiểm tra: `m_bWaitResponse`, `HL7690_CIMI == pMsg->id`.

**ProcessRx** — [source L1343](src/Driver/USB_Modem/Src/ShellState.cpp#L1343).

- Response/URC và mã status được kiểm tra: `HL7690_DATA`.

- Đích chuyển state/substate có trong handler: `SHELL_STATE_INIT_CNUM`.

- Các điều kiện kiểm tra: `m_bWaitResponse`, `HL7690_DATA == pMsg->id`.

**ProcessTimer** — [source L2020](src/Driver/USB_Modem/Src/ShellState.cpp#L2020).

- Lệnh gửi/tạo: `AT+CIMI`.

- Timer được chọn: `SHELL_STATE_TIMER_LONG_AT_COMMAND`.

**ProcessTimer** — [source L2097](src/Driver/USB_Modem/Src/ShellState.cpp#L2097).

- Lệnh gửi/tạo: `AT+CIMI`.

- Timer được chọn: `SHELL_STATE_TIMER_LONG_AT_COMMAND`.

#### `SHELL_STATE_INIT_MSISDN`

**ProcessRx** — [source L1175](src/Driver/USB_Modem/Src/ShellState.cpp#L1175).

- Response/URC và mã status được kiểm tra: `HL7690_CNUM`.

- Đích chuyển state/substate có trong handler: `SHELL_STATE_INIT_IMEI`.

- Các điều kiện kiểm tra: `m_bWaitResponse`, `HL7690_CNUM == pMsg->id`.

**ProcessTimer** — [source L2028](src/Driver/USB_Modem/Src/ShellState.cpp#L2028).

- Lệnh gửi/tạo: `AT+CNUM`.

- Timer được chọn: `SHELL_STATE_TIMER_LONG_AT_COMMAND`.

#### `SHELL_STATE_INIT_IMEI`

**ProcessRx** — [source L1196](src/Driver/USB_Modem/Src/ShellState.cpp#L1196).

- Response/URC và mã status được kiểm tra: `HL7690_CGSN`.

- Đích chuyển state/substate có trong handler: `SHELL_STATE_INIT_ICCID`.

- Các điều kiện kiểm tra: `m_bWaitResponse`, `HL7690_CGSN == pMsg->id`.

**ProcessTimer** — [source L2036](src/Driver/USB_Modem/Src/ShellState.cpp#L2036).

- Lệnh gửi/tạo: `AT+CGSN`.

- Timer được chọn: `SHELL_STATE_TIMER_LONG_AT_COMMAND`.

#### `SHELL_STATE_INIT_ICCID`

**ProcessRx** — [source L1215](src/Driver/USB_Modem/Src/ShellState.cpp#L1215).

- Response/URC và mã status được kiểm tra: `HL7690_CCID`.

- Đích chuyển state/substate có trong handler: `SHELL_STATE_INIT_CFG_URC_PORT`, `SHELL_STATE_INIT_CMGF`, `SHELL_STATE_READY`, `SHELL_STATE_INIT_CONFIGURE_NETWORK_FORMAT`.

- Các điều kiện kiểm tra: `m_bWaitResponse`, `HL7690_CCID == pMsg->id`.

**ProcessTimer** — [source L2044](src/Driver/USB_Modem/Src/ShellState.cpp#L2044).

- Lệnh gửi/tạo: `AT+CCID`, `AT+CCID?`.

- Timer được chọn: `SHELL_STATE_TIMER_LONG_AT_COMMAND`.

#### `SHELL_STATE_INIT_ATI3`

**ProcessRx** — [source L1247](src/Driver/USB_Modem/Src/ShellState.cpp#L1247).

- Lệnh gửi/tạo: `ATI3 ---> OK`.

- Response/URC và mã status được kiểm tra: `HL7690_OK`, `HL7690_ATI3`.

- Đích chuyển state/substate có trong handler: `SHELL_STATE_INIT_CMGF`, `SHELL_STATE_READY`.

- Tác động/callback: `Update_modem_version`.

- Các điều kiện kiểm tra: `m_bWaitResponse`, `HL7690_OK == pMsg->id`, `HL7690_ATI3 == pMsg->id`.

**ProcessRxCmd** — [source L1429](src/Driver/USB_Modem/Src/ShellState.cpp#L1429).

- Giữ state hiện tại hoặc chỉ làm bước chờ/thoát của handler; không phát AT hay chuyển state tại nhánh này.

**ProcessTimer** — [source L2060](src/Driver/USB_Modem/Src/ShellState.cpp#L2060).

- Lệnh gửi/tạo: `ATI3`.

- Timer được chọn: `SHELL_STATE_TIMER_SHORT_AT_COMMAND`.

#### `SHELL_STATE_INIT_CRC`

**ProcessRx** — [source L1295](src/Driver/USB_Modem/Src/ShellState.cpp#L1295).

- Response/URC và mã status được kiểm tra: `HL7690_OK`.

- Đích chuyển state/substate có trong handler: `SHELL_STATE_INIT_CLIP`.

- Các điều kiện kiểm tra: `m_bWaitResponse`, `HL7690_OK == pMsg->id`.

**ProcessTimer** — [source L2068](src/Driver/USB_Modem/Src/ShellState.cpp#L2068).

- Lệnh gửi/tạo: `AT+CRC=1`.

- Timer được chọn: `SHELL_STATE_TIMER_SHORT_AT_COMMAND`.

#### `SHELL_STATE_INIT_CLIP`

**ProcessRx** — [source L1304](src/Driver/USB_Modem/Src/ShellState.cpp#L1304).

- Response/URC và mã status được kiểm tra: `HL7690_OK`.

- Đích chuyển state/substate có trong handler: `SHELL_STATE_INIT_CGREG_STAT`.

- Các điều kiện kiểm tra: `m_bWaitResponse`, `HL7690_OK == pMsg->id`.

**ProcessTimer** — [source L2073](src/Driver/USB_Modem/Src/ShellState.cpp#L2073).

- Lệnh gửi/tạo: `AT+CLIP=1`.

- Timer được chọn: `SHELL_STATE_TIMER_SHORT_AT_COMMAND`.

#### `SHELL_STATE_INIT_CMGF`

**ProcessRx** — [source L1316](src/Driver/USB_Modem/Src/ShellState.cpp#L1316).

- Response/URC và mã status được kiểm tra: `HL7690_OK`.

- Đích chuyển state/substate có trong handler: `SHELL_STATE_INIT_CNMI`.

- Các điều kiện kiểm tra: `m_bWaitResponse`, `HL7690_OK == pMsg->id`.

**ProcessTimer** — [source L2081](src/Driver/USB_Modem/Src/ShellState.cpp#L2081).

- Lệnh gửi/tạo: `AT+CMGF=1`.

- Timer được chọn: `SHELL_STATE_TIMER_SHORT_AT_COMMAND`.

#### `SHELL_STATE_INIT_CNMI`

**ProcessRx** — [source L1325](src/Driver/USB_Modem/Src/ShellState.cpp#L1325).

- Response/URC và mã status được kiểm tra: `HL7690_OK`.

- Đích chuyển state/substate có trong handler: `SHELL_STATE_INIT_CMGD`.

- Các điều kiện kiểm tra: `m_bWaitResponse`, `HL7690_OK == pMsg->id`.

**ProcessTimer** — [source L2086](src/Driver/USB_Modem/Src/ShellState.cpp#L2086).

- Lệnh gửi/tạo: `AT+CNMI=2,2,0,0,0`.

- Timer được chọn: `SHELL_STATE_TIMER_LONG_AT_COMMAND`.

#### `SHELL_STATE_INIT_CMGD`

**ProcessRx** — [source L1334](src/Driver/USB_Modem/Src/ShellState.cpp#L1334).

- Response/URC và mã status được kiểm tra: `HL7690_OK`.

- Đích chuyển state/substate có trong handler: `SHELL_STATE_INIT_CIMI`.

- Các điều kiện kiểm tra: `m_bWaitResponse`, `HL7690_OK == pMsg->id`.

**ProcessTimer** — [source L2092](src/Driver/USB_Modem/Src/ShellState.cpp#L2092).

- Lệnh gửi/tạo: `AT+CMGD=1,4`.

- Timer được chọn: `SHELL_STATE_TIMER_LONG_AT_COMMAND`.

#### `SHELL_STATE_INIT_CNUM`

**ProcessRx** — [source L1359](src/Driver/USB_Modem/Src/ShellState.cpp#L1359).

- Response/URC và mã status được kiểm tra: `HL7690_CNUM`.

- Đích chuyển state/substate có trong handler: `SHELL_STATE_READY`.

- Các điều kiện kiểm tra: `m_bWaitResponse`, `HL7690_CNUM == pMsg->id`.

**ProcessTimer** — [source L2103](src/Driver/USB_Modem/Src/ShellState.cpp#L2103).

- Lệnh gửi/tạo: `AT+CNUM`.

- Timer được chọn: `SHELL_STATE_TIMER_LONG_AT_COMMAND`.

#### `SHELL_STATE_READY`

**ProcessRx** — [source L1389](src/Driver/USB_Modem/Src/ShellState.cpp#L1389).

- Giữ state hiện tại hoặc chỉ làm bước chờ/thoát của handler; không phát AT hay chuyển state tại nhánh này.

**ProcessRxCmd** — [source L1442](src/Driver/USB_Modem/Src/ShellState.cpp#L1442).

- Đích chuyển state/substate có trong handler: `SHELL_STATE_GET_MODEM_STATISTICS`, `SHELL_STATE_GET_NETWORK_OPERATOR_LIST`, `SHELL_STATE_GET_NETWORK_OPERATOR_INFO`, `SHELL_STATE_GET_MAIN_CELL_INFO`, `SHELL_STATE_GET_NEIGHBOUR_CELL_INFO`, `SHELL_STATE_GET_LTE_CONNECTION_REJECTION`, `SHELL_STATE_GET_LTE_NTP_CLOCK`, `SHELL_STATE_GET_LTE_MONITORING`, `SHELL_STATE_SET_LTE_MUTE`, `SHELL_STATE_SET_PIN`, `SHELL_STATE_CLOSE_LTE_TCP_SESSION`, `SHELL_STATE_OPEN_PDP_CONTEXT`, `SHELL_STATE_CLOSE_PDP_CONTEXT`, `SHELL_STATE_START_FOTA_TRANSFER`, `SHELL_STATE_DISABLE_FOTA_TRANSFER`, `SHELL_STATE_ERASE_FPLMN_LIST`, `SHELL_STATE_SET_RETRANS_CFG`, `SHELL_STATE_INIT_WAIT_STARTUP`.

- Message phối hợp: `SHELL_HOST_MSG_RX_GET_LTE_MODEM_STATISTICS`, `SHELL_HOST_MSG_RX_GET_NETWORK_OPERATOR_LIST`, `SHELL_HOST_MSG_RX_GET_NETWORK_OPERATOR_INFO`, `SHELL_HOST_MSG_RX_GET_MAIN_CELL_INFO`, `SHELL_HOST_MSG_RX_GET_NEIGHBOUR_CELL_INFO`, `SHELL_HOST_MSG_RX_GET_LTE_CONNECTION_REJECTION`, `SHELL_HOST_MSG_RX_GET_NTP_CLOCK_SYNC`, `SHELL_HOST_MSG_RX_GET_LTE_MONITORING`, `SHELL_HOST_MSG_RX_SET_LTE_MUTE`, `SHELL_HOST_MSG_PIN_UPDATE`, `SHELL_HOST_MSG_RX_CLOSE_TCP_SERVER_SESSION`, `SHELL_HOST_MSG_OPEN_PDP_CONTEXT`, `SHELL_HOST_MSG_CLOSE_PDP_CONTEXT`, `SHELL_HOST_MSG_RX_CLOSE_TCP_CLIENT_SESSION`, `SHELL_HOST_MSG_RX_START_FOTA_TRANSFER`, `SHELL_HOST_MSG_RX_DISABLE_FOTA_TRANSFER`, `SHELL_HOST_MSG_RX_ERASE_FPLMN_LIST`, `SHELL_HOST_MSG_SET_RETRANS_CFG`, `SHELL_HOST_MSG_RE_START_SHELL`.

- Tác động/callback: `Update_LTE_mute_status`.

- Các điều kiện kiểm tra: `elapsed < 0`, `elapsed > SHELL_STATE_POST_MUTE_PERIOD_TICKS`, `0 != m_u32TickOnLastMute`, `elapsed > SHELL_STATE_PRE_PING_PERIOD_TICKS`, `pGenericMsg->U16MessageType == SHELL_HOST_MSG_RX_GET_LTE_MODEM_STATISTICS`, `elapsed < SHELL_STATE_PRE_MUTE_PERIOD_TICKS`, `pGenericMsg->U16MessageType == SHELL_HOST_MSG_RX_GET_NETWORK_OPERATOR_LIST`, `pGenericMsg->U16MessageType == SHELL_HOST_MSG_RX_GET_NETWORK_OPERATOR_INFO`, `pGenericMsg->U16MessageType == SHELL_HOST_MSG_RX_GET_MAIN_CELL_INFO`, `pGenericMsg->U16MessageType == SHELL_HOST_MSG_RX_GET_NEIGHBOUR_CELL_INFO`, `pGenericMsg->U16MessageType == SHELL_HOST_MSG_RX_GET_LTE_CONNECTION_REJECTION`, `pGenericMsg->U16MessageType == SHELL_HOST_MSG_RX_GET_NTP_CLOCK_SYNC`, `pGenericMsg->U16MessageType == SHELL_HOST_MSG_RX_GET_LTE_MONITORING`, `pGenericMsg->U16MessageType == SHELL_HOST_MSG_RX_SET_LTE_MUTE`, `pGenericMsg->U16MessageType == SHELL_HOST_MSG_PIN_UPDATE`, `pGenericMsg->U16MessageType == SHELL_HOST_MSG_RX_CLOSE_TCP_SERVER_SESSION`, `pGenericMsg->U16MessageType == SHELL_HOST_MSG_OPEN_PDP_CONTEXT`, `pGenericMsg->U16MessageType == SHELL_HOST_MSG_CLOSE_PDP_CONTEXT`, `pGenericMsg->U16MessageType == SHELL_HOST_MSG_RX_CLOSE_TCP_CLIENT_SESSION`, `pGenericMsg->U16MessageType == SHELL_HOST_MSG_RX_START_FOTA_TRANSFER`, `pGenericMsg->U16MessageType == SHELL_HOST_MSG_RX_DISABLE_FOTA_TRANSFER`, `pGenericMsg->U16MessageType == SHELL_HOST_MSG_RX_ERASE_FPLMN_LIST`, `pGenericMsg->U16MessageType == SHELL_HOST_MSG_SET_RETRANS_CFG`, `pGenericMsg->U16MessageType == SHELL_HOST_MSG_RE_START_SHELL`.

**ProcessTimer** — [source L2110](src/Driver/USB_Modem/Src/ShellState.cpp#L2110).

- Timer được chọn: `SHELL_STATE_TIMER_FIVE_SECONDS`.

- Tác động/callback: `Cmux_Send_Modem_Event_To_App`, `ResetProcessIfReady`.

- Các điều kiện kiểm tra: `0 == ((cnt/1000)%30)`.

### C_ShellStateGetModemStatistics

#### `SHELL_STATE_GET_CSQ`

**ProcessRx** — [source L2208](src/Driver/USB_Modem/Src/ShellState.cpp#L2208).

- Response/URC và mã status được kiểm tra: `HL7690_CSQ`, `HL7690_OK`, `HL7690_CME_ERROR`, `HL7690_ERROR`.

- Đích chuyển state/substate có trong handler: `SHELL_STATE_GET_CGREG`.

- Tác động/callback: `Cmux_Send_Modem_Event_To_App`.

- Các điều kiện kiểm tra: `m_bWaitResponse`, `HL7690_CSQ == pMsg->id`, `pLteDiagnostic->ps_status == E_PS_STATUS_GPRS`, `(pLteDiagnostic->stCellInfoType.signalStrength < 5) \|\| (pLteDiagnostic->stCellInfoType.signalStrength == 99)`, `HL7690_OK == pMsg->id`, `(HL7690_CME_ERROR == pMsg->id) \|\| (HL7690_ERROR == pMsg->id)`.

**ProcessTimer** — [source L3152](src/Driver/USB_Modem/Src/ShellState.cpp#L3152).

- Lệnh gửi/tạo: `AT+CSQ`.

- Timer được chọn: `SHELL_STATE_TIMER_LONG_AT_COMMAND`.

#### `SHELL_STATE_GET_COPS`

**ProcessRx** — [source L2262](src/Driver/USB_Modem/Src/ShellState.cpp#L2262).

- Response/URC và mã status được kiểm tra: `HL7690_DATA`, `HL7690_OK`, `HL7690_ERROR`, `HL7690_COPS`, `HL7690_CME_ERROR`.

- Đích chuyển state/substate có trong handler: `SHELL_STATE_GET_CGREG`, `SHELL_STATE_GET_CSQ`.

- Tác động/callback: `Cmux_Send_Modem_Event_To_App`, `Update_modem_BitStatus`.

- Các điều kiện kiểm tra: `m_bWaitResponse`, `HL7690_DATA == pMsg->id`, `(2 < len) && (len < LEN_OPERATOR + 1)`, `HL7690_OK == pMsg->id`, `HL7690_ERROR == pMsg->id`, `HL7690_COPS == pMsg->id`, `2 < len`, `len <= LEN_OPERATOR + 1`, `ps_status == 8`, `ps_status == 9`, `ps_status == 7`, `ps_status == 2`, `(ps_status == 4) \|\| (ps_status == 5) \|\| (ps_status == 6)`, `(ps_status == 0) \|\| (ps_status == 3)`, `ps_status == 100`, `g_stBitStatus.u8IndexBit != 0xff`, `(HL7690_CME_ERROR == pMsg->id) \|\| (HL7690_ERROR == pMsg->id)`.

**ProcessTimer** — [source L3137](src/Driver/USB_Modem/Src/ShellState.cpp#L3137).

- Lệnh gửi/tạo: `AT+COPS?`.

- Timer được chọn: `SHELL_STATE_TIMER_COPS_AT_COMMAND`, `SHELL_STATE_TIMER_LONG_AT_COMMAND`.

#### `SHELL_STATE_GET_CGREG`

**ProcessRx** — [source L2390](src/Driver/USB_Modem/Src/ShellState.cpp#L2390).

- Response/URC và mã status được kiểm tra: `QUEC_4G_NETWORK`, `QUEC_CEREG`, `HL7690_CGREG`, `QUEC_LTE_NETWORK_CAT_M1_AND_NB1`, `HL7690_CREG`, `HL7690_CREG_STATUS_REGISTERED_HOME_NETWORK`, `HL7690_CREG_STATUS_REGISTERED_ROAMING`, `HL7690_CREG_STATUS_NOT_REGISTERED_NO_SEARCH_FOR_NEW_OPERATOR`, `HL7690_CREG_STATUS_REGISTRATION_UNKNOWN`, `HL7690_CREG_STATUS_REGISTRATION_DENIED`, `HL7690_OK`, `HL7690_CME_ERROR`, `HL7690_ERROR`.

- Đích chuyển state/substate có trong handler: `SHELL_STATE_GET_CGATT`.

- Tác động/callback: `Cmux_Send_Modem_Event_To_App`, `Get_LTE_connection_rejection`.

- Các điều kiện kiểm tra: `m_bWaitResponse`, `(m_u08Network_Mode == QUEC_4G_NETWORK && QUEC_CEREG == pMsg->id) \|\| (m_u08Network_Mode != QUEC_4G_NETWORK && HL7690_CGREG == pMsg->id)`, `(m_u08CatM1_Nb1_Mode <= QUEC_LTE_NETWORK_CAT_M1_AND_NB1 && QUEC_CEREG == pMsg->id) \|\| (m_u08CatM1_Nb1_Mode > QUEC_LTE_NETWORK_CAT_M1_AND_NB1 && HL7690_CGREG == pMsg->id)`, `HL7690_CREG == pMsg->id`, `service_status.operator_info.rat == CELL_RAT_GSM`, `(pMsg->rx.params[2].s32Param != HL7690_CREG_STATUS_REGISTERED_HOME_NETWORK) && (pMsg->rx.params[2].s32Param != HL7690_CREG_STATUS_REGISTERED_ROAMING)`, `(pMsg->rx.params[2].s32Param == HL7690_CREG_STATUS_NOT_REGISTERED_NO_SEARCH_FOR_NEW_OPERATOR) \|\| (pMsg->rx.params[2].s32Param == HL7690_CREG_STATUS_REGISTRATION_UNKNOWN)`, `m_u08NbOfUnknownCregState > SHELL_STATE_QUEC_UNKNONW_ATTACH_STATE_LIMIT`, `pMsg->rx.params[2].s32Param == HL7690_CREG_STATUS_REGISTRATION_DENIED`, `HL7690_OK == pMsg->id`, `(HL7690_CME_ERROR == pMsg->id) \|\| (HL7690_ERROR == pMsg->id)`.

**ProcessTimer** — [source L3158](src/Driver/USB_Modem/Src/ShellState.cpp#L3158).

- Lệnh gửi/tạo: `AT+CREG?`, `AT+CEREG?`, `AT+CGREG?`.

- Response/URC và mã status được kiểm tra: `QUEC_4G_NETWORK`, `QUEC_LTE_NETWORK_CAT_M1_AND_NB1`.

- Timer được chọn: `SHELL_STATE_TIMER_LONG_AT_COMMAND`.

- Các điều kiện kiểm tra: `m_u08Network_Mode == QUEC_4G_NETWORK`, `m_u08CatM1_Nb1_Mode <= QUEC_LTE_NETWORK_CAT_M1_AND_NB1`.

#### `SHELL_STATE_GET_CGATT`

**ProcessRx** — [source L2480](src/Driver/USB_Modem/Src/ShellState.cpp#L2480).

- Response/URC và mã status được kiểm tra: `HL7690_CGATT`, `HL7690_OK`, `HL7690_CME_ERROR`, `HL7690_ERROR`.

- Đích chuyển state/substate có trong handler: `SHELL_STATE_GET_XMETRIC`, `SHELL_STATE_GET_CELL_INFORMATION`, `SHELL_STATE_END_GET_MODEM_STATISTICS`, `SHELL_STATE_GET_SERVING_CELL_INFORMATION`.

- Tác động/callback: `Update_modem_statistics`, `Cmux_Send_Modem_Event_To_App`.

- Các điều kiện kiểm tra: `m_bWaitResponse`, `HL7690_CGATT == pMsg->id`, `cs_attachment == 0`, `HL7690_OK == pMsg->id`, `(HL7690_CME_ERROR == pMsg->id) \|\| (HL7690_ERROR == pMsg->id)`.

**ProcessTimer** — [source L3182](src/Driver/USB_Modem/Src/ShellState.cpp#L3182).

- Lệnh gửi/tạo: `AT+CGATT?`.

- Timer được chọn: `SHELL_STATE_TIMER_LONG_AT_COMMAND`.

#### `SHELL_STATE_GET_XMETRIC`

**ProcessRx** — [source L2537](src/Driver/USB_Modem/Src/ShellState.cpp#L2537).

- Response/URC và mã status được kiểm tra: `HL7690_XMETRIC`, `HL7690_OK`.

- Đích chuyển state/substate có trong handler: `SHELL_STATE_STOP_XMETRIC`.

- Các điều kiện kiểm tra: `m_bWaitResponse`, `HL7690_XMETRIC == pLongMsg->id`, `HL7690_OK == pMsg->id`.

**ProcessTimer** — [source L3188](src/Driver/USB_Modem/Src/ShellState.cpp#L3188).

- Lệnh gửi/tạo: `AT+XMETRIC=1,1,2`.

- Timer được chọn: `SHELL_STATE_TIMER_LONG_AT_COMMAND`.

#### `SHELL_STATE_STOP_XMETRIC`

**ProcessRx** — [source L2599](src/Driver/USB_Modem/Src/ShellState.cpp#L2599).

- Response/URC và mã status được kiểm tra: `HL7690_OK`.

- Đích chuyển state/substate có trong handler: `SHELL_STATE_GET_CELL_INFORMATION`.

- Các điều kiện kiểm tra: `m_bWaitResponse`, `HL7690_OK == pMsg->id`.

**ProcessTimer** — [source L3194](src/Driver/USB_Modem/Src/ShellState.cpp#L3194).

- Lệnh gửi/tạo: `AT+XMETRIC=0,1,2`.

- Timer được chọn: `SHELL_STATE_TIMER_LONG_AT_COMMAND`.

#### `SHELL_STATE_GET_SERVING_CELL_INFORMATION`

**ProcessRx** — [source L2624](src/Driver/USB_Modem/Src/ShellState.cpp#L2624).

- Response/URC và mã status được kiểm tra: `HL7690_OK`, `HL7690_ERROR`, `HL7690_CME_ERROR`.

- Đích chuyển state/substate có trong handler: `SHELL_STATE_GET_NEIGHBOUR_CELL_INFORMATION`.

- Tác động/callback: `Get_LTE_connection_rejection`, `Cmux_Send_Modem_Event_To_App`, `Update_LTE_monitoring`, `Update_modem_statistics`.

- Các điều kiện kiểm tra: `m_bWaitResponse`, `QENG_SERV == pLongMsg->id`, `strcmp(pMsg->rx.params[1].strParam, "\"SEARCH\"") == 0`, `m_u08NbOfServingCellSearch > SHELL_STATE_QUEC_UNKNONW_ATTACH_STATE_LIMIT`, `pLteDiagnostic->ps_status == E_PS_STATUS_GPRS \|\| pLteDiagnostic->ps_status == E_PS_STATUS_UMTS`, `pLteDiagnostic->ps_status == E_PS_STATUS_LTE \|\| pLteDiagnostic->ps_status == E_PS_STATUS_CAT_M1 \|\| pLteDiagnostic->ps_status == E_PS_STATUS_NB_1 \|\| pLteDiagnostic->ps_status == E_PS_STATUS_NB_2`, `pLteMonitoring`, `(strcmp(pMsg->rx.params[13].strParam, "-") != 0)`, `pLteDiagnostic`, `(pLteDiagnostic->ps_status == E_PS_STATUS_LTE) \|\| (pLteDiagnostic->ps_status == E_PS_STATUS_CAT_M1) \|\| (pLteDiagnostic->ps_status == E_PS_STATUS_NB_1) \|\| (pLteDiagnostic->ps_status == E_PS_STATUS_NB_2)`, `(pLteDiagnostic->stCellInfoType .signalStrength < 5) \|\| (pLteDiagnostic->stCellInfoType .signalStrength == 99)`, `HL7690_OK == pMsg->id`, `(HL7690_ERROR == pMsg->id) \|\| (HL7690_CME_ERROR == pMsg->id)`.

**ProcessTimer** — [source L3204](src/Driver/USB_Modem/Src/ShellState.cpp#L3204).

- Lệnh gửi/tạo: `AT+QENG ="servingcell"`.

- Timer được chọn: `SHELL_STATE_TIMER_LONG_AT_COMMAND`.

#### `SHELL_STATE_GET_NEIGHBOUR_CELL_INFORMATION`

**ProcessRx** — [source L2747](src/Driver/USB_Modem/Src/ShellState.cpp#L2747).

- Response/URC và mã status được kiểm tra: `HL7690_OK`, `HL7690_ERROR`, `HL7690_CME_ERROR`.

- Đích chuyển state/substate có trong handler: `SHELL_STATE_GET_QISTATE_SERVER_INFORMATION`.

- Tác động/callback: `Update_LTE_monitoring`, `Update_modem_statistics`.

- Các điều kiện kiểm tra: `m_bWaitResponse`, `QENG_NEIGHBOUR == pMsg->id`, `pLteDiagnostic->ps_status == E_PS_STATUS_LTE \|\| pLteDiagnostic->ps_status == E_PS_STATUS_CAT_M1 \|\| pLteDiagnostic->ps_status == E_PS_STATUS_NB_1 \|\| pLteDiagnostic->ps_status == E_PS_STATUS_NB_2`, `(pLteDiagnostic->ps_status == E_PS_STATUS_GPRS \|\| pLteDiagnostic->ps_status == E_PS_STATUS_UMTS) && (strcmp(pMsg->rx.params[1].strParam, "\"LTE\"") == 0)`, `(pLteDiagnostic->ps_status == E_PS_STATUS_GPRS \|\| pLteDiagnostic->ps_status == E_PS_STATUS_UMTS) && (strcmp(pMsg->rx.params[1].strParam, "\"GSM\"") == 0)`, `pLteDiagnostic->stAdjacentCellInfo[i].cellID == 0xFFFFFFFF`, `pLteDiagnostic->stAdjacentCellInfo[i] .signalQuality < rsrp`, `j < LEN_MAX_ADJACENT_CELL_INFO`, `pLteDiagnostic->stAdjacentCellInfo[i] .signalQuality < pLteDiagnostic->stAdjacentCellInfo[j] .signalQuality`, `HL7690_OK == pMsg->id`, `pLteDiagnostic->ps_status == E_PS_STATUS_GPRS`, `(HL7690_ERROR == pMsg->id) \|\| (HL7690_CME_ERROR == pMsg->id)`.

**ProcessTimer** — [source L3210](src/Driver/USB_Modem/Src/ShellState.cpp#L3210).

- Lệnh gửi/tạo: `AT+QENG ="neighbourcell"`.

- Timer được chọn: `SHELL_STATE_TIMER_LONG_AT_COMMAND`.

#### `SHELL_STATE_GET_QISTATE_SERVER_INFORMATION`

**ProcessRx** — [source L2851](src/Driver/USB_Modem/Src/ShellState.cpp#L2851).

- Response/URC và mã status được kiểm tra: `QUEC_QISTATE`, `HL7690_OK`, `HL7690_ERROR`, `HL7690_CME_ERROR`.

- Đích chuyển state/substate có trong handler: `SHELL_STATE_END_GET_MODEM_STATISTICS`.

- Tác động/callback: `Get_LTE_connection_rejection`, `Cmux_Send_Modem_Event_To_App`.

- Các điều kiện kiểm tra: `m_bWaitResponse`, `pMsg->id == QUEC_QISTATE`, `IsServerStarted() == TRUE`, `(u32SocketState != E_ServerSocketStateOpening) && (u32SocketState != E_ServerSocketStateConnected) && (u32SocketState != E_ServerSocketStateListening)`, `pMsg->id == HL7690_OK`, `(IsServerStarted() == TRUE) && (bIsServerCorrectlyWorking != TRUE)`, `(pMsg->id == HL7690_ERROR) \|\| (pMsg->id == HL7690_CME_ERROR)`.

**ProcessTimer** — [source L3216](src/Driver/USB_Modem/Src/ShellState.cpp#L3216).

- Lệnh gửi/tạo: `AT+QISTATE=1,%i`.

- Response/URC và mã status được kiểm tra: `QUEC_SERVER_CONNECT_ID`.

- Timer được chọn: `SHELL_STATE_TIMER_LONG_AT_COMMAND`.

#### `SHELL_STATE_GET_CELL_INFORMATION`

**ProcessRx** — [source L2896](src/Driver/USB_Modem/Src/ShellState.cpp#L2896).

- Response/URC và mã status được kiểm tra: `HL7690_XCELLINFO`, `HL7690_OK`, `HL7690_ERROR`, `HL7690_CME_ERROR`.

- Đích chuyển state/substate có trong handler: `SHELL_STATE_END_GET_MODEM_STATISTICS`.

- Tác động/callback: `Cmux_Send_Modem_Event_To_App`, `Update_modem_statistics`.

- Các điều kiện kiểm tra: `m_bWaitResponse`, `HL7690_XCELLINFO == pLongMsg->id`, `pLteDiagnostic->ps_status == E_PS_STATUS_UMTS \|\| pLteDiagnostic->ps_status == E_PS_STATUS_HSDPA`, `pLteDiagnostic->ps_status == E_PS_STATUS_LTE`, `pLongMsg->rx.params[2].s32Param == servingCell`, `bUpdateSignalStrength`, `(pLteDiagnostic->stCellInfoType .signalStrength < 5) \|\| (pLteDiagnostic->stCellInfoType .signalStrength == 99)`, `pMsg->rx.params[2].s32Param == neighborCell`, `pLteDiagnostic->stAdjacentCellInfo[i] .cellID == 0xFFFFFFFF`, `pLteDiagnostic->stAdjacentCellInfo[i] .signalQuality < rsrp`, `j < LEN_MAX_ADJACENT_CELL_INFO`, `pLteDiagnostic ->stAdjacentCellInfo[i] .signalQuality < pLteDiagnostic ->stAdjacentCellInfo[j] .signalQuality`, `HL7690_OK == pMsg->id`, `(HL7690_ERROR == pMsg->id) \|\| (HL7690_CME_ERROR == pMsg->id)`.

**ProcessTimer** — [source L3224](src/Driver/USB_Modem/Src/ShellState.cpp#L3224).

- Lệnh gửi/tạo: `AT+XCELLINFO?`.

- Timer được chọn: `SHELL_STATE_TIMER_LONG_AT_COMMAND`.

#### `SHELL_STATE_END_GET_MODEM_STATISTICS`

**ProcessTimer** — [source L3232](src/Driver/USB_Modem/Src/ShellState.cpp#L3232).

- Timer được chọn: `SHELL_STATE_TIMER_FIVE_SECONDS`.

- Tác động/callback: `Reset`.

### C_ShellStateGetLteMonitoring

#### `SHELL_STATE_GET_SIGNAL_INFORMATION_MONITORING`

**ProcessRx** — [source L3304](src/Driver/USB_Modem/Src/ShellState.cpp#L3304).

- Response/URC và mã status được kiểm tra: `QUEC_QCSQ`, `HL7690_OK`.

- Đích chuyển state/substate có trong handler: `SHELL_STATE_GET_CE_LEVEL_CELL_INFORMATION_MONITORING`.

- Các điều kiện kiểm tra: `m_bWaitResponse`, `QUEC_QCSQ == pMsg->id`, `HL7690_OK == pMsg->id`.

**ProcessTimer** — [source L3634](src/Driver/USB_Modem/Src/ShellState.cpp#L3634).

- Lệnh gửi/tạo: `AT+QCSQ`.

#### `SHELL_STATE_GET_CE_LEVEL_CELL_INFORMATION_MONITORING`

**ProcessRx** — [source L3328](src/Driver/USB_Modem/Src/ShellState.cpp#L3328).

- Response/URC và mã status được kiểm tra: `QCFG_CELEVEL`, `HL7690_OK`.

- Đích chuyển state/substate có trong handler: `SHELL_STATE_GET_EMMTIMER_CELL_INFORMATION_MONITORING`.

- Các điều kiện kiểm tra: `m_bWaitResponse`, `QCFG_CELEVEL == pMsg->id`, `HL7690_OK == pMsg->id`.

**ProcessTimer** — [source L3637](src/Driver/USB_Modem/Src/ShellState.cpp#L3637).

- Lệnh gửi/tạo: `AT+QCFG="celevel"`.

#### `SHELL_STATE_GET_EMMTIMER_CELL_INFORMATION_MONITORING`

**ProcessRx** — [source L3343](src/Driver/USB_Modem/Src/ShellState.cpp#L3343).

- Response/URC và mã status được kiểm tra: `QCFG_EMMTIMER`, `HL7690_OK`.

- Đích chuyển state/substate có trong handler: `SHELL_STATE_GET_SIBINFO_CELL_INFORMATION_MONITORING`, `SHELL_STATE_END_GET_LTE_MONITORING`.

- Các điều kiện kiểm tra: `m_bWaitResponse`, `QCFG_EMMTIMER == pMsg->id`, `HL7690_OK == pMsg->id`.

**ProcessTimer** — [source L3642](src/Driver/USB_Modem/Src/ShellState.cpp#L3642).

- Lệnh gửi/tạo: `AT+QCFG="emmtimer"`.

#### `SHELL_STATE_GET_SIBINFO_CELL_INFORMATION_MONITORING`

**ProcessRx** — [source L3360](src/Driver/USB_Modem/Src/ShellState.cpp#L3360).

- Response/URC và mã status được kiểm tra: `QCFG_SIB`, `HL7690_OK`.

- Đích chuyển state/substate có trong handler: `SHELL_STATE_SET_CEREG_CELL_INFORMATION_MONITORING`, `SHELL_STATE_END_GET_LTE_MONITORING`.

- Các điều kiện kiểm tra: `m_bWaitResponse`, `QCFG_SIB == pMsg->id`, `HL7690_OK == pMsg->id`.

**ProcessTimer** — [source L3647](src/Driver/USB_Modem/Src/ShellState.cpp#L3647).

- Lệnh gửi/tạo: `AT+QCFG="sibinfo"`.

#### `SHELL_STATE_SET_CEREG_CELL_INFORMATION_MONITORING`

**ProcessRx** — [source L3376](src/Driver/USB_Modem/Src/ShellState.cpp#L3376).

- Response/URC và mã status được kiểm tra: `HL7690_OK`.

- Đích chuyển state/substate có trong handler: `SHELL_STATE_GET_CEREG_CELL_INFORMATION_MONITORING`, `SHELL_STATE_END_GET_LTE_MONITORING`.

- Các điều kiện kiểm tra: `m_bWaitResponse`, `HL7690_OK == pMsg->id`.

**ProcessTimer** — [source L3652](src/Driver/USB_Modem/Src/ShellState.cpp#L3652).

- Lệnh gửi/tạo: `AT+CEREG=4`.

#### `SHELL_STATE_GET_CEREG_CELL_INFORMATION_MONITORING`

**ProcessRx** — [source L3389](src/Driver/USB_Modem/Src/ShellState.cpp#L3389).

- Response/URC và mã status được kiểm tra: `QUEC_CEREG`, `HL7690_OK`.

- Đích chuyển state/substate có trong handler: `SHELL_STATE_GET_DRX_CELL_INFORMATION_MONITORING`.

- Các điều kiện kiểm tra: `m_bWaitResponse`, `QUEC_CEREG == pMsg->id`, `HL7690_OK == pMsg->id`.

**ProcessTimer** — [source L3656](src/Driver/USB_Modem/Src/ShellState.cpp#L3656).

- Lệnh gửi/tạo: `AT+CEREG?`.

#### `SHELL_STATE_GET_DRX_CELL_INFORMATION_MONITORING`

**ProcessRx** — [source L3406](src/Driver/USB_Modem/Src/ShellState.cpp#L3406).

- Response/URC và mã status được kiểm tra: `QUEC_CEDRXRDP`.

- Đích chuyển state/substate có trong handler: `SHELL_STATE_SET_CEREG_CELL_INFORMATION_END`.

- Tác động/callback: `Update_LTE_monitoring`.

- Các điều kiện kiểm tra: `m_bWaitResponse`, `QUEC_CEDRXRDP == pMsg->id`, `pLteDiagnostic->ps_status == E_PS_STATUS_NB_1 \|\| pLteDiagnostic->ps_status == E_PS_STATUS_NB_2`.

**ProcessTimer** — [source L3660](src/Driver/USB_Modem/Src/ShellState.cpp#L3660).

- Lệnh gửi/tạo: `AT+CEDRXRDP`.

#### `SHELL_STATE_SET_CEREG_CELL_INFORMATION_END`

**ProcessRx** — [source L3433](src/Driver/USB_Modem/Src/ShellState.cpp#L3433).

- Response/URC và mã status được kiểm tra: `HL7690_OK`.

- Đích chuyển state/substate có trong handler: `SHELL_STATE_END_GET_LTE_MONITORING`, `SHELL_STATE_SET_CEREG_CELL_INFORMATION_END`.

- Các điều kiện kiểm tra: `m_bWaitResponse`, `HL7690_OK == pMsg->id`.

**ProcessTimer** — [source L3665](src/Driver/USB_Modem/Src/ShellState.cpp#L3665).

- Lệnh gửi/tạo: `AT+CEREG=1`.

#### `SHELL_STATE_GET_CELL_INFORMATION_MONITORING`

**ProcessRx** — [source L3443](src/Driver/USB_Modem/Src/ShellState.cpp#L3443).

- Response/URC và mã status được kiểm tra: `HL7690_XCELLINFO`, `HL7690_OK`.

- Đích chuyển state/substate có trong handler: `SHELL_STATE_END_GET_LTE_MONITORING`, `SHELL_STATE_GET_LTE_PARAMETERS`.

- Tác động/callback: `Update_LTE_monitoring`.

- Các điều kiện kiểm tra: `m_bWaitResponse`, `QENG_SERV == pMsg->id`, `HL7690_XCELLINFO == pMsg->id`, `pLongMsg->rx.params[2].s32Param == 5`, `HL7690_OK == pMsg->id`.

**ProcessTimer** — [source L3670](src/Driver/USB_Modem/Src/ShellState.cpp#L3670).

- Lệnh gửi/tạo: `AT+QENG="servingcell"`, `AT+XCELLINFO?`.

- Timer được chọn: `SHELL_STATE_TIMER_LONG_AT_COMMAND`.

#### `SHELL_STATE_GET_LTE_PARAMETERS`

**ProcessRx** — [source L3517](src/Driver/USB_Modem/Src/ShellState.cpp#L3517).

- Response/URC và mã status được kiểm tra: `QCFG_EMMTIMER`, `HL7690_KLTEPARAM`, `HL7690_OK`.

- Đích chuyển state/substate có trong handler: `SHELL_STATE_END_GET_LTE_MONITORING`.

- Tác động/callback: `Update_LTE_monitoring`.

- Các điều kiện kiểm tra: `m_bWaitResponse`, `QCFG_EMMTIMER == pMsg->id`, `HL7690_KLTEPARAM == pMsg->id`, `pLongMsg->rx.params[3].s32Param / 1000 <= 0xFFFF`, `pLongMsg->rx.params[5].s32Param / 1000 <= 0xFFFF`, `HL7690_OK == pMsg->id`.

**ProcessTimer** — [source L3681](src/Driver/USB_Modem/Src/ShellState.cpp#L3681).

- Lệnh gửi/tạo: `AT+KLTEPARAM?`.

- Timer được chọn: `SHELL_STATE_TIMER_LONG_AT_COMMAND`.

#### `SHELL_STATE_END_GET_LTE_MONITORING`

**ProcessTimer** — [source L3688](src/Driver/USB_Modem/Src/ShellState.cpp#L3688).

- Timer được chọn: `SHELL_STATE_TIMER_FIVE_SECONDS`.

- Tác động/callback: `Reset`.

### C_ShellStateGetLteConnectionRejection

#### `SHELL_STATE_LTE_CONNECTION_REJECTION_CEER`

**ProcessRx** — [source L3866](src/Driver/USB_Modem/Src/ShellState.cpp#L3866).

- Response/URC và mã status được kiểm tra: `QCFG_EMMCAUSE1`, `QCFG_EMMCAUSE2`, `HL7690_CEER_2`, `HL7690_CEER_1`, `HL7690_OK`.

- Đích chuyển state/substate có trong handler: `SHELL_STATE_LTE_CONNECTION_REJECTION_CEER2`, `SHELL_STATE_LTE_CONNECTION_REJECTION_COPS_NUM`, `SHELL_STATE_END_GET_LTE_CONNECTION_REJECTION`.

- Tác động/callback: `Update_LTE_connection_rejection`.

- Các điều kiện kiểm tra: `m_bWaitResponse`, `QCFG_EMMCAUSE1 == pMsg->id`, `QCFG_EMMCAUSE2 == pMsg->id`, `HL7690_CEER_2 == pMsg->id`, `HL7690_CEER_1 == pMsg->id`, `HL7690_OK == pMsg->id`.

**ProcessTimer** — [source L4059](src/Driver/USB_Modem/Src/ShellState.cpp#L4059).

- Lệnh gửi/tạo: `AT+QCFG="emmcause"`, `AT+CEER`.

- Timer được chọn: `SHELL_STATE_TIMER_LONG_AT_COMMAND`.

#### `SHELL_STATE_LTE_CONNECTION_REJECTION_CEER2`

**ProcessRx** — [source L3918](src/Driver/USB_Modem/Src/ShellState.cpp#L3918).

- Response/URC và mã status được kiểm tra: `HL7690_CME_ERROR`.

- Đích chuyển state/substate có trong handler: `SHELL_STATE_LTE_CONNECTION_REJECTION_DEACT_CONTEXT`.

- Tác động/callback: `ManageConnexionRetry`.

- Các điều kiện kiểm tra: `m_bWaitResponse`, `HL7690_CME_ERROR == pMsg->id`.

**ProcessTimer** — [source L4070](src/Driver/USB_Modem/Src/ShellState.cpp#L4070).

- Lệnh gửi/tạo: `AT+CEER`.

#### `SHELL_STATE_LTE_CONNECTION_REJECTION_COPS_NUM`

**ProcessRx** — [source L3932](src/Driver/USB_Modem/Src/ShellState.cpp#L3932).

- Response/URC và mã status được kiểm tra: `HL7690_OK`.

- Đích chuyển state/substate có trong handler: `SHELL_STATE_LTE_CONNECTION_REJECTION_READ_COPS`, `SHELL_STATE_END_GET_LTE_CONNECTION_REJECTION`.

- Các điều kiện kiểm tra: `m_bWaitResponse`, `HL7690_OK == pMsg->id`.

**ProcessTimer** — [source L4074](src/Driver/USB_Modem/Src/ShellState.cpp#L4074).

- Lệnh gửi/tạo: `AT+COPS=3,2`.

- Timer được chọn: `SHELL_STATE_TIMER_COPS_AT_COMMAND`.

#### `SHELL_STATE_LTE_CONNECTION_REJECTION_READ_COPS`

**ProcessRx** — [source L3944](src/Driver/USB_Modem/Src/ShellState.cpp#L3944).

- Response/URC và mã status được kiểm tra: `HL7690_COPS`, `HL7690_OK`.

- Đích chuyển state/substate có trong handler: `SHELL_STATE_LTE_CONNECTION_REJECTION_COPS_ALP`.

- Các điều kiện kiểm tra: `m_bWaitResponse`, `HL7690_COPS == pMsg->id`, `len == 7`, `HL7690_OK != pMsg->id`, `HL7690_OK == pMsg->id`.

**ProcessTimer** — [source L4080](src/Driver/USB_Modem/Src/ShellState.cpp#L4080).

- Lệnh gửi/tạo: `AT+COPS?`.

- Timer được chọn: `SHELL_STATE_TIMER_COPS_AT_COMMAND`.

#### `SHELL_STATE_LTE_CONNECTION_REJECTION_DEACT_CONTEXT`

**ProcessRx** — [source L3978](src/Driver/USB_Modem/Src/ShellState.cpp#L3978).

- Đích chuyển state/substate có trong handler: `SHELL_STATE_END_GET_LTE_CONNECTION_REJECTION`.

- Tác động/callback: `Update_modem_status`.

**ProcessTimer** — [source L4092](src/Driver/USB_Modem/Src/ShellState.cpp#L4092).

- Lệnh gửi/tạo: `AT+QIDEACT=1 `.

- Timer được chọn: `SHELL_STATE_TIMER_COPS_AT_COMMAND`.

#### `SHELL_STATE_LTE_CONNECTION_REJECTION_COPS_ALP`

**ProcessRx** — [source L3990](src/Driver/USB_Modem/Src/ShellState.cpp#L3990).

- Đích chuyển state/substate có trong handler: `SHELL_STATE_END_GET_LTE_CONNECTION_REJECTION`.

- Tác động/callback: `Update_LTE_connection_rejection`.

- Các điều kiện kiểm tra: `m_bWaitResponse`.

**ProcessTimer** — [source L4085](src/Driver/USB_Modem/Src/ShellState.cpp#L4085).

- Lệnh gửi/tạo: `AT+COPS=3,0`.

- Timer được chọn: `SHELL_STATE_TIMER_COPS_AT_COMMAND`.

#### `SHELL_STATE_END_GET_LTE_CONNECTION_REJECTION`

**ProcessTimer** — [source L4098](src/Driver/USB_Modem/Src/ShellState.cpp#L4098).

- Timer được chọn: `SHELL_STATE_TIMER_FIVE_SECONDS`.

- Tác động/callback: `Reset`, `RestartAllFsm`.

### C_ShellStateGetLteNTP

#### `SHELL_STATE_NTP_Sync`

**ProcessRx** — [source L4198](src/Driver/USB_Modem/Src/ShellState.cpp#L4198).

- Response/URC và mã status được kiểm tra: `HL7690_OK`, `HL7690_NOTIF_SHELL`.

- Đích chuyển state/substate có trong handler: `SHELL_STATE_NTP_CLK`, `SHELL_STATE_NTP_S`.

- Các điều kiện kiểm tra: `m_bWaitResponse`, `QNTP_SYNC == pMsg->id`, `pTimer`, `HL7690_OK == pMsg->id`.

**ProcessTimer** — [source L4298](src/Driver/USB_Modem/Src/ShellState.cpp#L4298).

- Lệnh gửi/tạo: `AT+QNTP=1,"%s",%s`.

- Response/URC và mã status được kiểm tra: `HL7690_NOTIF_SHELL`.

- Timer được chọn: `SHELL_STATE_TIMER_LONG_AT_COMMAND`.

#### `SHELL_STATE_NTP_CLK`

**ProcessRx** — [source L4229](src/Driver/USB_Modem/Src/ShellState.cpp#L4229).

- Response/URC và mã status được kiểm tra: `HL7690_OK`.

- Đích chuyển state/substate có trong handler: `SHELL_STATE_NTP_S`.

- Các điều kiện kiểm tra: `m_bWaitResponse`, `QNTP_CCLK == pMsg->id`, `HL7690_OK == pMsg->id`.

**ProcessTimer** — [source L4308](src/Driver/USB_Modem/Src/ShellState.cpp#L4308).

- Lệnh gửi/tạo: `AT+CCLK?`.

- Response/URC và mã status được kiểm tra: `HL7690_NOTIF_SHELL`.

- Timer được chọn: `SHELL_STATE_TIMER_LONG_AT_COMMAND`.

#### `SHELL_STATE_NTP_Sync_END`

**ProcessTimer** — [source L4314](src/Driver/USB_Modem/Src/ShellState.cpp#L4314).

- Timer được chọn: `SHELL_STATE_TIMER_FIVE_SECONDS`.

- Tác động/callback: `Reset`.

### C_ShellStateSetLteMute

#### `SHELL_STATE_EXECUTE_LTE_MUTE`

**ProcessRx** — [source L4369](src/Driver/USB_Modem/Src/ShellState.cpp#L4369).

- Response/URC và mã status được kiểm tra: `HL7690_OK`.

- Đích chuyển state/substate có trong handler: `SHELL_STATE_GET_LTE_MUTE`, `SHELL_STATE_END_SET_LTE_MUTE`.

- Các điều kiện kiểm tra: `m_bWaitResponse`, `HL7690_OK == pMsg->id`.

**ProcessTimer** — [source L4459](src/Driver/USB_Modem/Src/ShellState.cpp#L4459).

- Lệnh gửi/tạo: `AT+KLTEMUTE=1,20,1`.

- Response/URC và mã status được kiểm tra: `HL7690_NOTIF_SHELL`.

- Timer được chọn: `SHELL_STATE_TIMER_LONG_AT_COMMAND`.

- Tác động/callback: `Update_LTE_mute_status`.

#### `SHELL_STATE_GET_LTE_MUTE`

**ProcessRx** — [source L4381](src/Driver/USB_Modem/Src/ShellState.cpp#L4381).

- Response/URC và mã status được kiểm tra: `HL7690_KLTEMUTE`, `HL7690_NOTIF_SHELL`, `HL7690_OK`.

- Đích chuyển state/substate có trong handler: `SHELL_STATE_END_SET_LTE_MUTE`.

- Các điều kiện kiểm tra: `HL7690_KLTEMUTE == pMsg->id`, `pMsg->rx.params[1].s32Param == 1`, `HL7690_OK == pMsg->id`.

**ProcessTimer** — [source L4469](src/Driver/USB_Modem/Src/ShellState.cpp#L4469).

- Timer được chọn: `SHELL_STATE_TIMER_LONG_AT_COMMAND`.

#### `SHELL_STATE_END_SET_LTE_MUTE`

**ProcessTimer** — [source L4473](src/Driver/USB_Modem/Src/ShellState.cpp#L4473).

- Timer được chọn: `SHELL_STATE_TIMER_FIVE_SECONDS`.

- Tác động/callback: `Update_LTE_mute_status`, `Reset`.

### C_ShellStateCloseLteTcpSession

#### `SHELL_STATE_CLOSE_TCP_SERVER_SESSION`

**ProcessRx** — [source L4533](src/Driver/USB_Modem/Src/ShellState.cpp#L4533).

- Đích chuyển state/substate có trong handler: `SHELL_STATE_END_CLOSE_TCP_SERVER_SESSION`.

- Các điều kiện kiểm tra: `m_bWaitResponse`.

**ProcessTimer** — [source L4616](src/Driver/USB_Modem/Src/ShellState.cpp#L4616).

- Giữ state hiện tại hoặc chỉ làm bước chờ/thoát của handler; không phát AT hay chuyển state tại nhánh này.

#### `SHELL_STATE_CLOSE_TCP_CLIENT_SESSION`

**ProcessRx** — [source L4542](src/Driver/USB_Modem/Src/ShellState.cpp#L4542).

- Đích chuyển state/substate có trong handler: `SHELL_STATE_END_CLOSE_TCP_CLIENT_SESSION`.

- Các điều kiện kiểm tra: `m_bWaitResponse`.

**ProcessTimer** — [source L4621](src/Driver/USB_Modem/Src/ShellState.cpp#L4621).

- Lệnh gửi/tạo: `AT+QICLOSE=%i`, `AT+KTCPCLOSE=%i`.

- Timer được chọn: `SHELL_STATE_TIMER_LONG_AT_COMMAND`.

- Tác động/callback: `CloseTCP`.

#### `SHELL_STATE_END_CLOSE_TCP_CLIENT_SESSION`

**ProcessRx** — [source L4549](src/Driver/USB_Modem/Src/ShellState.cpp#L4549).

- Các điều kiện kiểm tra: `m_bWaitResponse`.

**ProcessTimer** — [source L4640](src/Driver/USB_Modem/Src/ShellState.cpp#L4640).

- Lệnh gửi/tạo: `AT+QICLOSE=%i`, `AT+KTCPDEL=%i`.

- Timer được chọn: `SHELL_STATE_TIMER_LONG_AT_COMMAND`.

- Tác động/callback: `CloseTCP`.

#### `SHELL_STATE_END_CLOSE_TCP_SERVER_SESSION`

**ProcessTimer** — [source L4633](src/Driver/USB_Modem/Src/ShellState.cpp#L4633).

- Timer được chọn: `SHELL_STATE_TIMER_FIVE_SECONDS`.

- Tác động/callback: `Reset`.

### C_ShellStateOpenPDPContext

#### `SHELL_STATE_PDP_STARTUP`

**ProcessRx** — [source L4716](src/Driver/USB_Modem/Src/ShellState.cpp#L4716).

- Giữ state hiện tại hoặc chỉ làm bước chờ/thoát của handler; không phát AT hay chuyển state tại nhánh này.

**ProcessTimer** — [source L4956](src/Driver/USB_Modem/Src/ShellState.cpp#L4956).

- Đích chuyển state/substate có trong handler: `SHELL_STATE_PDP_AT_KCNXCFG`, `SHELL_STATE_PDP_AT_CGDCONT`, `SHELL_STATE_PDP_AT_KCGPADDR`.

- Các điều kiện kiểm tra: `b_IsFirstInitPDP == TRUE`, `GetOpenServer()`.

#### `SHELL_STATE_PDP_AT_CGDCONT`

**ProcessRx** — [source L4719](src/Driver/USB_Modem/Src/ShellState.cpp#L4719).

- Response/URC và mã status được kiểm tra: `HL7690_OK`.

- Đích chuyển state/substate có trong handler: `SHELL_STATE_PDP_AT_KCNXCFG`, `SHELL_STATE_PDP_AT_CGACT`.

- Các điều kiện kiểm tra: `m_bWaitResponse`, `HL7690_OK == pMsg->id`.

**ProcessTimer** — [source L4971](src/Driver/USB_Modem/Src/ShellState.cpp#L4971).

- Lệnh gửi/tạo: `AT+CGDCONT=1,"IP","%s"`.

- Timer được chọn: `SHELL_STATE_TIMER_FIVE_SECONDS`.

#### `SHELL_STATE_PDP_AT_CGACT`

**ProcessRx** — [source L4732](src/Driver/USB_Modem/Src/ShellState.cpp#L4732).

- Response/URC và mã status được kiểm tra: `HL7690_OK`.

- Đích chuyển state/substate có trong handler: `SHELL_STATE_PDP_AT_KCNXCFG (delay 1000)`.

- Các điều kiện kiểm tra: `m_bWaitResponse`, `HL7690_OK == pMsg->id`.

**ProcessTimer** — [source L4978](src/Driver/USB_Modem/Src/ShellState.cpp#L4978).

- Lệnh gửi/tạo: `AT+CGACT=0`.

- Timer được chọn: `SERVER_STATE_TIMER_LONG_AT_COMMAND`.

#### `SHELL_STATE_PDP_AT_KCNXCFG`

**ProcessRx** — [source L4741](src/Driver/USB_Modem/Src/ShellState.cpp#L4741).

- Response/URC và mã status được kiểm tra: `HL7690_OK`.

- Đích chuyển state/substate có trong handler: `SHELL_STATE_PDP_CHECK_QIACT`, `SHELL_STATE_PDP_AT_KCNXUP`.

- Các điều kiện kiểm tra: `m_bWaitResponse`, `HL7690_OK == pMsg->id`.

**ProcessTimer** — [source L4984](src/Driver/USB_Modem/Src/ShellState.cpp#L4984).

- Lệnh gửi/tạo: `AT+QICSGP=1,1,"apn","%s","%s"`, `AT+KCNXCFG=1,"GPRS","apn","%s","%s"`, `AT+QICSGP=1,1,"%s","%s","%s"`, `AT+KCNXCFG=1,"GPRS","%s","%s","%s"`.

- Timer được chọn: `SHELL_STATE_TIMER_LONG_AT_COMMAND`.

- Các điều kiện kiểm tra: `0 == strlen((char *)(pModemConfig->APN_Name))`.

#### `SHELL_STATE_PDP_CHECK_QIACT`

**ProcessRx** — [source L4757](src/Driver/USB_Modem/Src/ShellState.cpp#L4757).

- Response/URC và mã status được kiểm tra: `QUEC_QIACT`.

- Đích chuyển state/substate có trong handler: `SHELL_STATE_PDP_AT_KCGPADDR`, `SHELL_STATE_PDP_AT_KCNXUP`.

- Tác động/callback: `Update_modem_BitStatus`.

- Các điều kiện kiểm tra: `m_bWaitResponse`, `QUEC_QIACT == pMsg->id`.

**ProcessTimer** — [source L5016](src/Driver/USB_Modem/Src/ShellState.cpp#L5016).

- Lệnh gửi/tạo: `AT+QIACT?`.

- Timer được chọn: `SHELL_STATE_TIMER_QIACT_AT_COMMAND`.

#### `SHELL_STATE_PDP_AT_KCNXUP`

**ProcessRx** — [source L4776](src/Driver/USB_Modem/Src/ShellState.cpp#L4776).

- Response/URC và mã status được kiểm tra: `HL7690_OK`, `HL7690_ERROR`.

- Đích chuyển state/substate có trong handler: `SHELL_STATE_PDP_AT_KCGPADDR`.

- Tác động/callback: `Cmux_Send_Modem_Event_To_App`, `Get_LTE_connection_rejection`, `Reset`.

- Các điều kiện kiểm tra: `m_bWaitResponse`, `HL7690_OK == pMsg->id`, `HL7690_ERROR == pMsg->id`.

**ProcessTimer** — [source L5021](src/Driver/USB_Modem/Src/ShellState.cpp#L5021).

- Lệnh gửi/tạo: `AT+QIACT=1`, `AT+KCNXUP=1`.

- Timer được chọn: `SHELL_STATE_TIMER_QIACT_AT_COMMAND`.

#### `SHELL_STATE_PDP_AT_KCGPADDR`

**ProcessRx** — [source L4791](src/Driver/USB_Modem/Src/ShellState.cpp#L4791).

- Response/URC và mã status được kiểm tra: `QUEC_QIACT`, `HL7690_KCGPADDR`, `HL7690_OK`.

- Đích chuyển state/substate có trong handler: `SHELL_STATE_PDP_AT_KCNXUP (delay 3000)`, `SHELL_STATE_PDP_END`.

- Tác động/callback: `Update_modem_status`, `Reply_get_modem_statistics`, `Get_modem_statistics`, `Cmux_Send_Modem_Event_To_App`.

- Các điều kiện kiểm tra: `m_bWaitResponse`, `QUEC_QIACT == pMsg->id`, `HL7690_KCGPADDR == pMsg->id`, `strchr(addr, '.')`, `(2 < len) && (len < (LEN_IPV4_ADDR + 2))`, `strchr(addr, ':')`, `(2 < len) && (len < (LEN_IPV6_ADDR + 2))`, `HL7690_OK == pMsg->id`, `(strlen((char *)pModemStatus->ipv6addr) < 2) && (strlen((char *)pModemStatus->ipv4addr) < 2)`, `strlen((char *)pModemStatus->ipv4addr) < 2`, `b_IsFirstInitPDP == TRUE`, `strnlen(pModemStatus->ipv4addr)`.

**ProcessTimer** — [source L5032](src/Driver/USB_Modem/Src/ShellState.cpp#L5032).

- Lệnh gửi/tạo: `AT+QIACT?`, `AT+KCGPADDR=1`.

- Timer được chọn: `SHELL_STATE_TIMER_QIACT_AT_COMMAND`.

#### `SHELL_STATE_PDP_END`

**ProcessRx** — [source L4894](src/Driver/USB_Modem/Src/ShellState.cpp#L4894).

- Giữ state hiện tại hoặc chỉ làm bước chờ/thoát của handler; không phát AT hay chuyển state tại nhánh này.

**ProcessTimer** — [source L5042](src/Driver/USB_Modem/Src/ShellState.cpp#L5042).

- Timer được chọn: `SHELL_STATE_TIMER_FIVE_SECONDS`.

- Tác động/callback: `SetOpenServer`, `Reset`.

### C_ShellStateClosePDPContext

#### `SHELL_STATE_CLOSE_PDP_AT_KCNXDOWN`

**ProcessRx** — [source L5096](src/Driver/USB_Modem/Src/ShellState.cpp#L5096).

- Response/URC và mã status được kiểm tra: `HL7690_OK`, `HL7690_KCNX_IND`, `HL7690_CNX_STATUS_STR`, `HL7690_CNX_STATUS_DISCONNECTED`, `HL7690_CNX_STATUS_CLOSED`.

- Đích chuyển state/substate có trong handler: `SHELL_STATE_CLOSE_PDP_END`.

- Tác động/callback: `Update_modem_status`.

- Các điều kiện kiểm tra: `m_bWaitResponse`, `HL7690_OK == pMsg->id`, `HL7690_KCNX_IND == pMsg->id`.

**ProcessTimer** — [source L5198](src/Driver/USB_Modem/Src/ShellState.cpp#L5198).

- Lệnh gửi/tạo: `AT+QIDEACT=1`, `AT+KCNXDOWN=1,1`.

- Timer được chọn: `SHELL_STATE_TIMER_LONG_AT_COMMAND`.

- Tác động/callback: `Reset`.

- Các điều kiện kiểm tra: `(C_ServerHost::GetInstance().GetCurrentServerState() == SERVER_STATE_INIT) && (C_ClientHost::GetInstance().GetCurrentClientState() == CLIENT_STATE_INIT)`.

#### `SHELL_STATE_CLOSE_PDP_END`

**ProcessRx** — [source L5136](src/Driver/USB_Modem/Src/ShellState.cpp#L5136).

- Giữ state hiện tại hoặc chỉ làm bước chờ/thoát của handler; không phát AT hay chuyển state tại nhánh này.

**ProcessTimer** — [source L5218](src/Driver/USB_Modem/Src/ShellState.cpp#L5218).

- Timer được chọn: `SHELL_STATE_TIMER_FIVE_SECONDS`.

- Tác động/callback: `Reset`.

### C_ShellStateStartFotaTransfer

#### `SHELL_STATE_FOTA_DL`

**ProcessRx** — [source L5289](src/Driver/USB_Modem/Src/ShellState.cpp#L5289).

- Response/URC và mã status được kiểm tra: `HL7690_CME_ERROR`, `HL7690_ERROR`, `QIND_FOTA`.

- Đích chuyển state/substate có trong handler: `SHELL_STATE_END_START_FOTA_TRANSFER`.

- Tác động/callback: `Update_FU_LTE_modem_status`.

- Các điều kiện kiểm tra: `m_bWaitResponse`, `(HL7690_CME_ERROR == pMsg->id) \|\| (HL7690_ERROR == pMsg->id)`, `QIND_FOTA == pMsg->id`, `strcmp(pMsg->rx.params[1].strParam, "\"FTPSTART\"") == 0`, `strcmp(pMsg->rx.params[1].strParam, "\"FTPEND\"") == 0`, `pMsg->rx.params[2].s32Param == 0`, `strcmp(pMsg->rx.params[1].strParam, "\"HTTPSTART\"") == 0`.

**ProcessTimer** — [source L5598](src/Driver/USB_Modem/Src/ShellState.cpp#L5598).

- Lệnh gửi/tạo: `AT+QFOTADL="%s"`.

- Timer được chọn: `SHELL_STATE_TIMER_FOTA_ACTIVATION`.

- Đích chuyển state/substate có trong handler: `SHELL_STATE_END_START_FOTA_TRANSFER`.

- Tác động/callback: `Update_FU_LTE_modem_status`.

- Các điều kiện kiểm tra: `m_bFotaInProgress`.

#### `SHELL_STATE_FOTA_GET_INDICATION_MSG`

**ProcessRx** — [source L5343](src/Driver/USB_Modem/Src/ShellState.cpp#L5343).

- Response/URC và mã status được kiểm tra: `HL7690_OK`, `HL7690_CME_ERROR`.

- Đích chuyển state/substate có trong handler: `SHELL_STATE_FOTA_CONFIGURE_DEVICE_SERVICE`, `SHELL_STATE_END_START_FOTA_TRANSFER`.

- Tác động/callback: `Update_FU_LTE_modem_status`.

- Các điều kiện kiểm tra: `m_bWaitResponse`, `HL7690_OK == pMsg->id`, `HL7690_CME_ERROR == pMsg->id`.

**ProcessTimer** — [source L5620](src/Driver/USB_Modem/Src/ShellState.cpp#L5620).

- Lệnh gửi/tạo: `AT+WDSI=3839`.

- Response/URC và mã status được kiểm tra: `HL7690_NOTIF_SHELL`.

- Timer được chọn: `SHELL_STATE_TIMER_LONG_AT_COMMAND`.

#### `SHELL_STATE_FOTA_CONFIGURE_DEVICE_SERVICE`

**ProcessRx** — [source L5357](src/Driver/USB_Modem/Src/ShellState.cpp#L5357).

- Response/URC và mã status được kiểm tra: `HL7690_OK`, `HL7690_CME_ERROR`.

- Đích chuyển state/substate có trong handler: `SHELL_STATE_FOTA_CONFIGURE_IP_ADDRESS_1`, `SHELL_STATE_FOTA_INITIATED_CONNECTION`, `SHELL_STATE_END_START_FOTA_TRANSFER`.

- Tác động/callback: `Update_FU_LTE_modem_status`.

- Các điều kiện kiểm tra: `m_bWaitResponse`, `HL7690_OK == pMsg->id`, `C_ShellHost::GetInstance().GetLenFotaIpAddress()`, `HL7690_CME_ERROR == pMsg->id`.

**ProcessTimer** — [source L5628](src/Driver/USB_Modem/Src/ShellState.cpp#L5628).

- Lệnh gửi/tạo: `AT+WDSS=0,"apn","%s","%s",1`, `AT+WDSS=0,"%s","%s","%s",1`.

- Response/URC và mã status được kiểm tra: `HL7690_NOTIF_SHELL`.

- Timer được chọn: `SHELL_STATE_TIMER_LONG_AT_COMMAND`.

- Các điều kiện kiểm tra: `0 == strlen((char *)(pModemConfig->APN_Name))`.

#### `SHELL_STATE_FOTA_CONFIGURE_IP_ADDRESS_1`

**ProcessRx** — [source L5377](src/Driver/USB_Modem/Src/ShellState.cpp#L5377).

- Response/URC và mã status được kiểm tra: `HL7690_PROMPT`, `HL7690_CME_ERROR`.

- Đích chuyển state/substate có trong handler: `SHELL_STATE_FOTA_CONFIGURE_IP_ADDRESS_2`, `SHELL_STATE_END_START_FOTA_TRANSFER`.

- Tác động/callback: `Update_FU_LTE_modem_status`.

- Các điều kiện kiểm tra: `m_bWaitResponse`, `HL7690_PROMPT == pMsg->id`, `HL7690_CME_ERROR == pMsg->id`.

**ProcessTimer** — [source L5645](src/Driver/USB_Modem/Src/ShellState.cpp#L5645).

- Lệnh gửi/tạo: `AT+WDSW=3,2,"./Wm/`.

- Response/URC và mã status được kiểm tra: `HL7690_NOTIF_SHELL`.

- Timer được chọn: `SHELL_STATE_TIMER_LONG_AT_COMMAND`.

#### `SHELL_STATE_FOTA_CONFIGURE_IP_ADDRESS_2`

**ProcessRx** — [source L5392](src/Driver/USB_Modem/Src/ShellState.cpp#L5392).

- Đích chuyển state/substate có trong handler: `SHELL_STATE_FOTA_CONFIGURE_IP_ADDRESS_3 (delay (2 *
                         SHELL_STATE_TIMER_TOUCH_AND_GO)`.

- Các điều kiện kiểm tra: `m_bWaitResponse`.

**ProcessTimer** — [source L5656](src/Driver/USB_Modem/Src/ShellState.cpp#L5656).

- Response/URC và mã status được kiểm tra: `HL7690_NOTIF_SHELL`.

- Timer được chọn: `SHELL_STATE_TIMER_LONG_AT_COMMAND`.

#### `SHELL_STATE_FOTA_CONFIGURE_IP_ADDRESS_3`

**ProcessRx** — [source L5402](src/Driver/USB_Modem/Src/ShellState.cpp#L5402).

- Response/URC và mã status được kiểm tra: `HL7690_OK`, `HL7690_CME_ERROR`.

- Đích chuyển state/substate có trong handler: `SHELL_STATE_FOTA_INITIATED_CONNECTION`, `SHELL_STATE_END_START_FOTA_TRANSFER`.

- Tác động/callback: `Update_FU_LTE_modem_status`.

- Các điều kiện kiểm tra: `m_bWaitResponse`, `HL7690_OK == pMsg->id`, `HL7690_CME_ERROR == pMsg->id`.

**ProcessTimer** — [source L5665](src/Driver/USB_Modem/Src/ShellState.cpp#L5665).

- Response/URC và mã status được kiểm tra: `HL7690_NOTIF_SHELL`.

- Timer được chọn: `SHELL_STATE_TIMER_LONG_AT_COMMAND`.

#### `SHELL_STATE_FOTA_INITIATED_CONNECTION`

**ProcessRx** — [source L5417](src/Driver/USB_Modem/Src/ShellState.cpp#L5417).

- Response/URC và mã status được kiểm tra: `HL7690_WDSI`, `HL7690_OK`.

- Đích chuyển state/substate có trong handler: `SHELL_STATE_END_START_FOTA_TRANSFER`, `SHELL_STATE_FOTA_TRANSFERT`.

- Message phối hợp: `SERVER_HOST_MSG_RX_FU_ENABLE`.

- Tác động/callback: `Update_FU_LTE_modem_status`.

- Các điều kiện kiểm tra: `m_bWaitResponse`, `HL7690_WDSI == pMsg->id`, `(pMsg->rx.params[1].s32Param == 5) \|\| (pMsg->rx.params[1].s32Param == 7)`, `pMsg->rx.params[1].s32Param == 6`, `(pMsg->rx.params[1].s32Param == 8) && (m_u08SubState != SHELL_STATE_FOTA_TRANSFERT)`, `pMsg->rx.params[1].s32Param == 9`, `pMsg->rx.params[1].s32Param == 10`, `pMsg->rx.params[1].s32Param == 11`, `pMsg->rx.params[1].s32Param == 12`, `pMsg->rx.params[1].s32Param == 13`, `pMsg->rx.params[1].s32Param == 14`, `HL7690_OK != pMsg->id`.

**ProcessTimer** — [source L5673](src/Driver/USB_Modem/Src/ShellState.cpp#L5673).

- Lệnh gửi/tạo: `AT+WDSS=1,1`.

- Response/URC và mã status được kiểm tra: `HL7690_NOTIF_SHELL`.

- Timer được chọn: `SHELL_STATE_TIMER_FOTA_AT_COMMAND`.

#### `SHELL_STATE_FOTA_TRANSFERT`

**ProcessRx** — [source L5419](src/Driver/USB_Modem/Src/ShellState.cpp#L5419).

- Response/URC và mã status được kiểm tra: `HL7690_WDSI`, `HL7690_OK`.

- Đích chuyển state/substate có trong handler: `SHELL_STATE_END_START_FOTA_TRANSFER`, `SHELL_STATE_FOTA_TRANSFERT`.

- Message phối hợp: `SERVER_HOST_MSG_RX_FU_ENABLE`.

- Tác động/callback: `Update_FU_LTE_modem_status`.

- Các điều kiện kiểm tra: `m_bWaitResponse`, `HL7690_WDSI == pMsg->id`, `(pMsg->rx.params[1].s32Param == 5) \|\| (pMsg->rx.params[1].s32Param == 7)`, `pMsg->rx.params[1].s32Param == 6`, `(pMsg->rx.params[1].s32Param == 8) && (m_u08SubState != SHELL_STATE_FOTA_TRANSFERT)`, `pMsg->rx.params[1].s32Param == 9`, `pMsg->rx.params[1].s32Param == 10`, `pMsg->rx.params[1].s32Param == 11`, `pMsg->rx.params[1].s32Param == 12`, `pMsg->rx.params[1].s32Param == 13`, `pMsg->rx.params[1].s32Param == 14`, `HL7690_OK != pMsg->id`.

**ProcessTimer** — [source L5680](src/Driver/USB_Modem/Src/ShellState.cpp#L5680).

- Response/URC và mã status được kiểm tra: `HL7690_NOTIF_SHELL`.

- Timer được chọn: `SHELL_STATE_TIMER_FOTA_AT_COMMAND`.

#### `SHELL_STATE_FOTA_TRANSFER_SUCCESS`

**ProcessTimer** — [source L5615](src/Driver/USB_Modem/Src/ShellState.cpp#L5615).

- Tác động/callback: `ResetHard`.

#### `SHELL_STATE_END_START_FOTA_TRANSFER`

**ProcessTimer** — [source L5689](src/Driver/USB_Modem/Src/ShellState.cpp#L5689).

- Timer được chọn: `SHELL_STATE_TIMER_FIVE_SECONDS`.

- Tác động/callback: `Reset`.

### C_ShellStateDisableFotaTransfer

#### `SHELL_STATE_FOTA_NO_INDICATION_MSG`

**ProcessRx** — [source L5732](src/Driver/USB_Modem/Src/ShellState.cpp#L5732).

- Đích chuyển state/substate có trong handler: `SHELL_STATE_FOTA_RELEASE_CONNECTION`.

- Các điều kiện kiểm tra: `m_bWaitResponse`.

**ProcessTimer** — [source L5804](src/Driver/USB_Modem/Src/ShellState.cpp#L5804).

- Lệnh gửi/tạo: `AT+WDSI=0`.

- Timer được chọn: `SHELL_STATE_TIMER_LONG_AT_COMMAND`.

#### `SHELL_STATE_FOTA_RELEASE_CONNECTION`

**ProcessRx** — [source L5739](src/Driver/USB_Modem/Src/ShellState.cpp#L5739).

- Đích chuyển state/substate có trong handler: `SHELL_STATE_END_DISABLE_FOTA_TRANSFER`.

- Tác động/callback: `Update_FU_LTE_modem_status`.

- Các điều kiện kiểm tra: `m_bWaitResponse`.

**ProcessTimer** — [source L5811](src/Driver/USB_Modem/Src/ShellState.cpp#L5811).

- Lệnh gửi/tạo: `AT+WDSS=1,0`.

- Timer được chọn: `SHELL_STATE_TIMER_LONG_AT_COMMAND`.

- Message phối hợp: `SERVER_HOST_MSG_RX_FU_ENABLE`.

#### `SHELL_STATE_END_DISABLE_FOTA_TRANSFER`

**ProcessTimer** — [source L5826](src/Driver/USB_Modem/Src/ShellState.cpp#L5826).

- Timer được chọn: `SHELL_STATE_TIMER_FIVE_SECONDS`.

- Tác động/callback: `Reset`.

### C_ShellStateSetPIN

#### `SHELL_STATE_EXECUTE_SET_PIN`

**ProcessRx** — [source L5882](src/Driver/USB_Modem/Src/ShellState.cpp#L5882).

- Response/URC và mã status được kiểm tra: `HL7690_CPIN`, `HL7690_OK`, `HL7690_CME_ERROR`.

- Đích chuyển state/substate có trong handler: `SHELL_STATE_END_SET_PIN`.

- Tác động/callback: `Reply_update_CPIN`.

- Các điều kiện kiểm tra: `m_bWaitResponse`, `HL7690_CPIN == pMsg->id`, `HL7690_OK == pMsg->id`, `HL7690_CME_ERROR == pMsg->id`.

**ProcessTimer** — [source L5975](src/Driver/USB_Modem/Src/ShellState.cpp#L5975).

- Timer được chọn: `SHELL_STATE_TIMER_LONG_AT_COMMAND`.

#### `SHELL_STATE_END_SET_PIN`

**ProcessRx** — [source L5916](src/Driver/USB_Modem/Src/ShellState.cpp#L5916).

- Giữ state hiện tại hoặc chỉ làm bước chờ/thoát của handler; không phát AT hay chuyển state tại nhánh này.

**ProcessTimer** — [source L5982](src/Driver/USB_Modem/Src/ShellState.cpp#L5982).

- Timer được chọn: `SHELL_STATE_TIMER_FIVE_SECONDS`.

- Tác động/callback: `Reset`.

### C_ShellStateEraseFPLMNList

#### `SHELL_STATE_SELECT_FPLMN_LIST`

**ProcessRx** — [source L6028](src/Driver/USB_Modem/Src/ShellState.cpp#L6028).

- Response/URC và mã status được kiểm tra: `HL7690_CSIM`, `HL7690_OK`.

- Đích chuyển state/substate có trong handler: `SHELL_STATE_MAJ_FPLMN_LIST`, `SHELL_STATE_END_ERASE_FPLMN_LIST`.

- Các điều kiện kiểm tra: `m_bWaitResponse`, `HL7690_CSIM == pMsg->id`, `HL7690_OK == pMsg->id`.

**ProcessTimer** — [source L6111](src/Driver/USB_Modem/Src/ShellState.cpp#L6111).

- Lệnh gửi/tạo: `AT+CSIM=14,"00A40004026F7B"`.

- Timer được chọn: `SHELL_STATE_TIMER_LONG_AT_COMMAND`.

#### `SHELL_STATE_MAJ_FPLMN_LIST`

**ProcessRx** — [source L6043](src/Driver/USB_Modem/Src/ShellState.cpp#L6043).

- Response/URC và mã status được kiểm tra: `HL7690_CSIM`, `HL7690_OK`.

- Đích chuyển state/substate có trong handler: `SHELL_STATE_END_ERASE_FPLMN_LIST`.

- Các điều kiện kiểm tra: `m_bWaitResponse`, `HL7690_CSIM == pMsg->id`, `HL7690_OK == pMsg->id`.

**ProcessTimer** — [source L6118](src/Driver/USB_Modem/Src/ShellState.cpp#L6118).

- Lệnh gửi/tạo: `AT+CSIM=34,"00D600000CFFFFFFFFFFFFFFFFFFFFFFFF"`.

- Timer được chọn: `SHELL_STATE_TIMER_LONG_AT_COMMAND`.

#### `SHELL_STATE_END_ERASE_FPLMN_LIST`

**ProcessTimer** — [source L6125](src/Driver/USB_Modem/Src/ShellState.cpp#L6125).

- Timer được chọn: `SHELL_STATE_TIMER_FIVE_SECONDS`.

- Tác động/callback: `Reset`.

### C_ShellStateSetRetransCfg

#### `SHELL_STATE_RETRANS_CFG_MIN_RTO`

**ProcessRx** — [source L6188](src/Driver/USB_Modem/Src/ShellState.cpp#L6188).

- Response/URC và mã status được kiểm tra: `HL7690_OK`.

- Đích chuyển state/substate có trong handler: `SHELL_STATE_RETRANS_CFG_MAX_RTO`.

- Các điều kiện kiểm tra: `m_bWaitResponse`, `HL7690_OK == pMsg->id`.

**ProcessTimer** — [source L6269](src/Driver/USB_Modem/Src/ShellState.cpp#L6269).

- Lệnh gửi/tạo: `AT+QICFG="tcp/minrto",%d`.

- Timer được chọn: `SHELL_STATE_TIMER_LONG_AT_COMMAND`.

#### `SHELL_STATE_RETRANS_CFG_MAX_RTO`

**ProcessRx** — [source L6199](src/Driver/USB_Modem/Src/ShellState.cpp#L6199).

- Response/URC và mã status được kiểm tra: `HL7690_OK`.

- Đích chuyển state/substate có trong handler: `SHELL_STATE_RETRANS_CFG_END`.

- Các điều kiện kiểm tra: `m_bWaitResponse`, `HL7690_OK == pMsg->id`.

**ProcessTimer** — [source L6276](src/Driver/USB_Modem/Src/ShellState.cpp#L6276).

- Lệnh gửi/tạo: `AT+QICFG="tcp/retranscfg",%d,%d`.

- Timer được chọn: `SHELL_STATE_TIMER_LONG_AT_COMMAND`.

#### `SHELL_STATE_RETRANS_CFG_END`

**ProcessTimer** — [source L6283](src/Driver/USB_Modem/Src/ShellState.cpp#L6283).

- Timer được chọn: `SHELL_STATE_TIMER_FIVE_SECONDS`.

- Tác động/callback: `Reset`.

### C_ShellGetNetworkOperatorList

#### `SHELL_STATE_GET_LIST_OPERATOR`

**ProcessRx** — [source L6332](src/Driver/USB_Modem/Src/ShellState.cpp#L6332).

- Response/URC và mã status được kiểm tra: `HL7690_DATA`, `HL7690_CME_ERROR`, `HL7690_ERROR`.

- Đích chuyển state/substate có trong handler: `SHELL_STATE_END_GET_LIST_OPERATOR`.

- Tác động/callback: `Update_modem_List_Operator`.

- Các điều kiện kiểm tra: `m_bWaitResponse`, `HL7690_DATA == pMsg->id`, `pTokenCops != NULL`, `pToken != NULL`, `uSize + uLen >= sizeof(pBuff)`, `uSize != 0`, `(HL7690_CME_ERROR == pMsg->id) \|\| (HL7690_ERROR == pMsg->id)`.

**ProcessTimer** — [source L6447](src/Driver/USB_Modem/Src/ShellState.cpp#L6447).

- Lệnh gửi/tạo: `AT+COPS=?`.

- Timer được chọn: `SHELL_STATE_TIMER_COPS_AT_COMMAND`.

#### `SHELL_STATE_END_GET_LIST_OPERATOR`

**ProcessRx** — [source L6383](src/Driver/USB_Modem/Src/ShellState.cpp#L6383).

- Giữ state hiện tại hoặc chỉ làm bước chờ/thoát của handler; không phát AT hay chuyển state tại nhánh này.

**ProcessTimer** — [source L6452](src/Driver/USB_Modem/Src/ShellState.cpp#L6452).

- Timer được chọn: `SHELL_STATE_TIMER_FIVE_SECONDS`.

- Tác động/callback: `Reset`.

### C_ShellGetNetworkOperatorInfo

#### `SHELL_STATE_GET_LIST_OPERATOR`

**ProcessRx** — [source L6499](src/Driver/USB_Modem/Src/ShellState.cpp#L6499).

- Response/URC và mã status được kiểm tra: `HL7690_DATA`, `HL7690_CME_ERROR`, `HL7690_ERROR`.

- Đích chuyển state/substate có trong handler: `SHELL_STATE_END_GET_LIST_OPERATOR`.

- Tác động/callback: `Update_modem_Info_Operator`.

- Các điều kiện kiểm tra: `m_bWaitResponse`, `HL7690_DATA == pMsg->id`, `(HL7690_CME_ERROR == pMsg->id) \|\| (HL7690_ERROR == pMsg->id)`.

**ProcessTimer** — [source L6583](src/Driver/USB_Modem/Src/ShellState.cpp#L6583).

- Lệnh gửi/tạo: `AT+COPS?`.

- Timer được chọn: `SHELL_STATE_TIMER_COPS_AT_COMMAND`.

#### `SHELL_STATE_END_GET_LIST_OPERATOR`

**ProcessRx** — [source L6519](src/Driver/USB_Modem/Src/ShellState.cpp#L6519).

- Giữ state hiện tại hoặc chỉ làm bước chờ/thoát của handler; không phát AT hay chuyển state tại nhánh này.

**ProcessTimer** — [source L6588](src/Driver/USB_Modem/Src/ShellState.cpp#L6588).

- Timer được chọn: `SHELL_STATE_TIMER_FIVE_SECONDS`.

- Tác động/callback: `Reset`.

### C_ShellGetMainCellInfo

#### `SHELL_STATE_MAIN_CELL_INFO`

**ProcessRx** — [source L6635](src/Driver/USB_Modem/Src/ShellState.cpp#L6635).

- Response/URC và mã status được kiểm tra: `HL7690_DATA`, `HL7690_CME_ERROR`, `HL7690_ERROR`.

- Đích chuyển state/substate có trong handler: `SHELL_STATE_MAIN_CELL_INFO_END`.

- Tác động/callback: `Update_modem_Info_ServingCell`.

- Các điều kiện kiểm tra: `m_bWaitResponse`, `HL7690_DATA == pMsg->id`, `(HL7690_CME_ERROR == pMsg->id) \|\| (HL7690_ERROR == pMsg->id)`.

**ProcessTimer** — [source L6714](src/Driver/USB_Modem/Src/ShellState.cpp#L6714).

- Lệnh gửi/tạo: `AT+QENG="servingcell"`.

- Timer được chọn: `SHELL_STATE_TIMER_COPS_AT_COMMAND`.

#### `SHELL_STATE_MAIN_CELL_INFO_END`

**ProcessRx** — [source L6655](src/Driver/USB_Modem/Src/ShellState.cpp#L6655).

- Giữ state hiện tại hoặc chỉ làm bước chờ/thoát của handler; không phát AT hay chuyển state tại nhánh này.

**ProcessTimer** — [source L6720](src/Driver/USB_Modem/Src/ShellState.cpp#L6720).

- Timer được chọn: `SHELL_STATE_TIMER_FIVE_SECONDS`.

- Tác động/callback: `Reset`.

### C_ShellGetNeighbourCellInfo

#### `SHELL_STATE_NEIGHBOUR_CELL_INFO`

**ProcessRx** — [source L6767](src/Driver/USB_Modem/Src/ShellState.cpp#L6767).

- Response/URC và mã status được kiểm tra: `HL7690_DATA`, `HL7690_CME_ERROR`, `HL7690_ERROR`.

- Đích chuyển state/substate có trong handler: `SHELL_STATE_NEIGHBOUR_CELL_INFO_END`.

- Tác động/callback: `Update_modem_Info_NeighbourCell`.

- Các điều kiện kiểm tra: `m_bWaitResponse`, `HL7690_DATA == pMsg->id`, `pTokenCops != NULL`, `pToken != NULL`, `uSize + uLen >= sizeof(pBuff)`, `uSize != 0`, `(pToken != NULL) \|\| (pTokenOk != NULL)`, `(HL7690_CME_ERROR == pMsg->id) \|\| (HL7690_ERROR == pMsg->id)`.

**ProcessTimer** — [source L6881](src/Driver/USB_Modem/Src/ShellState.cpp#L6881).

- Lệnh gửi/tạo: `AT+QENG="neighbourcell"`.

- Timer được chọn: `SHELL_STATE_TIMER_COPS_AT_COMMAND`.

#### `SHELL_STATE_NEIGHBOUR_CELL_INFO_END`

**ProcessRx** — [source L6819](src/Driver/USB_Modem/Src/ShellState.cpp#L6819).

- Giữ state hiện tại hoặc chỉ làm bước chờ/thoát của handler; không phát AT hay chuyển state tại nhánh này.

**ProcessTimer** — [source L6887](src/Driver/USB_Modem/Src/ShellState.cpp#L6887).

- Timer được chọn: `SHELL_STATE_TIMER_FIVE_SECONDS`.

- Tác động/callback: `Reset`.
