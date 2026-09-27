# Client và Server FSM của cellular modem — OMAN-MT212V2_NLR-app

Target: [`uvproj/OMAN-MT212V2_NLR-app.uvprojx`](uvproj/OMAN-MT212V2_NLR-app.uvprojx). Với `CONFIG_PORT_GPRS`, cấu hình chung bật `CONFIG_FEATURE_GPRS_TCP_CLIENT_AUTO_CONNECT=1`, `CONFIG_FEATURE_GPRS_TCP_CLIENT_MODE=1`, `CONFIG_FEATURE_GPRS_TCP_SERVER_CLOSE_NO_RESET=1`; phần cứng MS212V2 bật Quectel EG91. Source chính: [`ClientHost.cpp`](src/Driver/USB_Modem/Src/ClientHost.cpp), [`ClientState.cpp`](src/Driver/USB_Modem/Src/ClientState.cpp), [`ServerHost.cpp`](src/Driver/USB_Modem/Src/ServerHost.cpp), [`ServerState.cpp`](src/Driver/USB_Modem/Src/ServerState.cpp). Client là kết nối **đi ra** (push/ping); Server lắng nghe kết nối **đi vào**. Cả hai nằm ở Application/NLR.

## Cơ chế chung và phối hợp

Mỗi host nhận message queue từ ứng dụng, CMUX và timer, rồi gọi handler của state hiện tại. State trả về `E_CLIENT_STATE` hoặc `E_SERVER_STATE`; host gọi `Reset()` của state cũ nếu state thay đổi. `GotoSubState()` đặt lại cờ chờ phản hồi, số lần gửi và timer; `SendDataWaitResponse()` gửi AT trên kênh client/server của CMUX, parser trả response/URC về host. Timer mang ID để loại event cũ. `ShellHost` đảm nhiệm cấu hình/điều khiển radio và PDP; Client/Server chỉ vận hành socket sau khi nền mạng đã sẵn sàng. Xem thêm [Shell FSM](README-cellular-shell-fsm.md).

![State machine Client và Server](documentation/diagrams/cellular/client-server-fsm.svg)

[Bản draw.io có thể chỉnh sửa](documentation/diagrams/cellular/client-server-fsm.drawio)

## Client FSM: kết nối đi ra

### Nhiệm vụ và thao tác xuống modem

Client chuyển yêu cầu **push hoặc ping của Application thành socket đi ra**, chọn địa chỉ/port và TCP/UDP từ cấu hình, đệm payload vào `pDataBuffer`, gửi payload sau prompt, kiểm tra ACK và chuyển dữ liệu nhận về `USB_Recv()`. Tên `USB_Recv()` là đường adapter trong code, không có nghĩa payload modem bắt buộc đi qua USB vật lý. Quectel dùng connection ID **2** trong các lệnh Client.

| State lớn | Thực hiện trong code | AT/GPIO xuống modem |
| --- | --- | --- |
| `CLIENT_STATE_INIT` | Chờ startup, tắt echo trên kênh client, báo READY; chỉ nhận yêu cầu mở khi substate đã READY | `ATE0`; không ghi GPIO |
| `CLIENT_STATE_CONNECT` | Mở socket, lấy IP, đợi payload, truyền/nhận bytes, xác nhận gửi và đóng socket | `AT+QIOPEN`, `AT+QIACT?`, `AT+QISEND=2,<len>`, bytes payload, `AT+QISEND=2,0`, `AT+QIRD`, `AT+QICLOSE`; không ghi GPIO trực tiếp |
| `TEST_STATE_INIT` | Thiết lập đường test modem | `ATE0`, `AT+CMEE=1` |
| `TEST_STATE_LISTENING` | Nhận lệnh test: đọc IMSI/IMEI/CCID/SN/hardware/firmware, test TX/RX, đổi functionality hoặc generic command | `CIMI`, `CGSN`, `CCID`, `KGSN`, `HWREV`, `CGMR`, `WMTXPOWER`, `WMRXPOWER`, `CFUN` theo chipset; các lệnh Sierra không tự động áp dụng EG91 |

Client reset thường dọn state/timer và báo trạng thái client; **không tự bật/tắt nguồn modem**. Nếu lỗi AT/timeout cần khôi phục toàn modem, Client gửi yêu cầu `ServerHost::ResetHard()` theo điều kiện Server đang INIT. Chuỗi GPIO được mô tả bên dưới.

State lớn chỉ có `CLIENT_STATE_INIT`, `CLIENT_STATE_CONNECT`; `TEST_STATE_INIT/LISTENING` là đường kiểm thử sản xuất, không phải luồng push bình thường. Các substate được định nghĩa trong [`ClientState.h`](src/Driver/USB_Modem/Inc/ClientState.h).

| State/substate | Sự kiện/điều kiện | Hành động và bước tiếp |
| --- | --- | --- |
| `INIT/WAIT_STARTUP` | Timer khởi động | `INIT_ATE0` |
| `INIT/ATE0` | Phát `ATE0`; parser trả `HL7690_OK` khi đang chờ | `INIT/READY`; timer cập nhật `LTE_TCP_CLIENT_READY` |
| `INIT/READY` | `CLIENT_HOST_MSG_WAKEUP`, yêu cầu kết nối hoặc ping | `ProcessConnectToServer()` → `CONNECT/AT_KTCPCFG`; ping nạp payload ping mẫu. Khi chưa READY, yêu cầu chưa làm mở socket |
| `CONNECT/AT_KTCPCFG` | Timer → `AT+QIOPEN=1,2,"TCP"/...` hoặc `"UDP"` từ IP/port cấu hình; `QUEC_OPEN` status `0` | Ghi session ID, đi `AT_GET_ADDRESS`. `ERROR`/`CME ERROR`/open fail → phát sự kiện lỗi; nếu socket ID đã dùng thì xóa session, nếu ping thì lấy monitoring và đi `WAITING_PING_CAPTURE`, trường hợp khác reset client |
| `CONNECT/AT_GET_ADDRESS` | `AT+QIACT?` trả địa chỉ và `OK` | Cập nhật IPv4 trong modem status, đặt `PUSH_CONNECTED`/`PING_CONNECTED`, vào `WAIT_FOR_DATA` |
| `CONNECT/WAIT_FOR_DATA` | Có payload ứng dụng (`CLIENT_HOST_MSG_DATA_READY`) | `AT_KTCPSND`; nếu không có payload, timer 5 giây tiếp tục theo dõi, quá thời gian idle trong code thì đóng socket |
| `CONNECT/AT_KTCPSND` | Timer gửi `AT+QISEND=2,<len>`; modem trả prompt `CONNECT` | Gửi bytes qua `CMUX_CLIENT_CHANNEL`; `SEND OK` → `VERIFY_ACKED_PACKET` cho TCP khi cần xác nhận, hoặc ACK push và quay lại `WAIT_FOR_DATA` theo phương thức push/UDP. `SEND FAIL`, `ERROR`, `CME ERROR`, disconnect/no carrier → `CLOSE_SESSION` |
| `CONNECT/VERIFY_ACKED_PACKET` | `AT+QISEND=2,0` trả tổng bytes đã gửi và đã ACK | Bằng nhau → ACK push, về `WAIT_FOR_DATA`; chưa bằng → kiểm tra lại sau 300 ms; lỗi/URC close → đóng |
| `CONNECT/ACCEPTED_RCV` | `+QIRD` báo payload nhận (Push Setup v2) | `AT+QIRD` lấy bytes, đẩy lên `USB_Recv()`, rồi về `WAIT_FOR_DATA`; close/error → đóng |
| `CONNECT/CLOSE_SESSION` → `DELETE_SESSION` | Quectel bỏ qua bước đóng kiểu Sierra và sang xóa; `AT+QICLOSE=<id>` | `OK` hoàn tất việc đóng, cập nhật trạng thái và quay về `INIT`; phản hồi lỗi/timeout dùng nhánh xử lý lỗi của FSM |

**Điều kiện hồi phục:** `ProcessTimer()` bỏ timer ID cũ. `CLIENT_STATE_TX_MAX_RETRIES` được kiểm tra sau lần gửi AT; quá mức mà Server đang `INIT` thì yêu cầu Server hard reset, còn Server đã chạy thì reset Client và quay `INIT`. `CLIENT_HOST_MSG_RESET`, `STOP_FSM`, `START_FSM` ở host cũng đưa state về `INIT` (hoặc `TEST_STATE_INIT` khi ở test mode). `CLIENT_HOST_MSG_RX_CMUX` chỉ phân tích bytes; chính message RX được parser tạo mới làm đổi substate. Xem [`C_ClientStateInit`](src/Driver/USB_Modem/Src/ClientState.cpp) và `C_ClientStateConnect` trong cùng file.

## Server FSM: kết nối đi vào

### Nhiệm vụ và thao tác xuống modem

Server vận hành **socket lắng nghe của meter**, nhận kết nối HES/DLMS, đưa bytes nhận vào adapter ứng dụng, gửi response ứng dụng và áp dụng chính sách calling window/auto connect. Nó cũng là điểm điều phối reset toàn modem khi Shell hoặc Client phát hiện lỗi. `ServerHost` giữ modem status/config và các cờ mode/window; `ServerState` tạo lệnh AT và xử lý response/URC.

| State lớn | Thực hiện trong code | AT/GPIO xuống modem |
| --- | --- | --- |
| `INIT` | Đợi quyền mở server, thiết lập kênh và profile, mở TCP listener hoặc UDP service | `ATE0`, `CMEE`, `AT&K3`, `CGDCONT` tùy đường init; Quectel `AT+QIOPEN=1,<id>,"TCP LISTENER"/"UDP SERVICE","127.0.0.1",0,<port>,0` |
| `LISTENING` | Chờ kết nối, lưu session/subsession khi có URC, kiểm tra window/mode và timeout | Chủ yếu nhận URC; nhánh Sierra có `AT+KTCPSTART`. Quectel chuyển sang ACCEPTED khi nhận yêu cầu kết nối |
| `ACCEPTED` | Đọc payload DLMS, giữ session đang phục vụ, đếm inactivity; yêu cầu đóng khi hết điều kiện phục vụ | `AT+QIRD`, `AT+QICLOSE` hoặc nhờ Shell đóng session; chuyển bytes lên `USB_Recv()` |
| `SEND` | Gửi response ứng dụng trên phiên con của Quectel | `AT+QISEND=<subsession>,<len>`, đợi prompt rồi gửi bytes; lỗi có thể gửi `AT+QICLOSE` |
| `CLOSE` | Đóng listener và yêu cầu Shell đóng PDP, đưa server về INIT/IDLE | `AT+QICLOSE` tại hai substate; không mặc định cắt nguồn modem |
| `UDP_LISTENING` | Nhận/gửi datagram, lưu địa chỉ và port đối tác | `AT+QIRD`, `AT+QISEND=<id>,<len>,"<peer IP>",<peer port>` |
| `RESET` | Thử shutdown bằng AT, đợi kết quả; nếu thất bại yêu cầu hard reset | EG91 `AT+QPOWD`; nhận `QUEC_QPOWD` → `ResetSoft()`. Vượt retry → `ResetHard()` và đường power/GPIO dưới đây |

## Điều khiển GPIO power/reset và khởi động CMUX

Source: [`ModemHost.cpp`](src/Driver/USB_Modem/Src/ModemHost.cpp), [`drv_gprs.c`](src/Driver/Gprs/Src/drv_gprs.c), [`Drv_Gprs_Config.h`](src/Config/driver/Drv_Gprs_Config.h), [`ServerHost.cpp`](src/Driver/USB_Modem/Src/ServerHost.cpp). Với macro `CONFIG_PRODUCT_MS212V2`, pin trong code là `DRV_GPRS_POWER = PE9`, `DRV_GPRS_RESET = PE13`. Đây là tên tín hiệu phần mềm; cần schematic để xác nhận tín hiệu vật lý là PWRKEY, RESET hoặc enable nguồn của board.

| Hàm / đường gọi | Thao tác thực tế |
| --- | --- |
| `ModemHost_Task_Init()` | Gọi `DrvGPRS_Power(GPRS_PowerOn)` theo nhánh startup, rồi `CMUX.PreStartup()/Startup()` và khởi tạo các host |
| `C_ModemHost::PowerOn()` | Gọi `DrvGPRS_Power(GPRS_PowerOn)` |
| `C_ModemHost::PowerOff()` | Gọi `DrvGPRS_Power(GPRS_PowerOff)` |
| `C_ModemHost::PowerReset()` | Gọi `DrvGPRS_Reset()`; **ở EG91 hàm driver không ghi GPIO vì thân thao tác nằm trong `#if CONFIG_HARDWARE_QUECTEL_EG91 == 0`** |
| `C_ModemHost::ResetOff()` | Qua `DrvGPRS_CtrlGPIO()`, đặt GPIO RESET về `Bit_RESET` |
| `DrvGPRS_PowerRF()` | EG91: ghi POWER `Bit_RESET` khi OFF và `Bit_SET` khi ON; comment code mô tả thao tác cắt/cấp VDD radio |
| `DrvGPRS_Stop()` | Power off, RF off, disable UART, dừng TX/RX DMA và hủy timer driver |

### Chuỗi GPIO trong `DrvGPRS_Power()`

Các mức dưới đây là giá trị **code ghi xuống chân**, không suy diễn mức active của modem từ tên hàm:

- **ON:** POWER=0 → RESET=1 → chờ 1000 ms → RESET=0 → chờ 100 ms → POWER=1 → chờ 850 ms → POWER=0.
- **OFF với MS212V2:** POWER=`HIGH_LEVEL_PIN`=0, RESET=`HIGH_LEVEL_PIN_RST`=1 → chờ 100 ms → POWER=`LOW_LEVEL_PIN`=1 → chờ 850 ms → POWER=0.
- Hàm còn tác động các đường SPIRIT/S2LP trước chuỗi nguồn và có điều kiện theo cover PLC/cellular; nếu điều kiện chọn module không cho phép thì chuỗi power GPIO bị bỏ qua. Không nên coi mỗi lời gọi `PowerOn()` đều chắc chắn thực thi toàn chuỗi trên.
- `DrvGPRS_GetStatusPower()` đọc **output GPIO POWER**, không trực tiếp truy vấn modem đã boot hoặc đã đăng ký mạng.

### Ba loại reset phải phân biệt

1. **Reset FSM Client/Shell:** xóa state/substate, timer và cờ; không trực tiếp cắt nguồn. Sau reset, các AT init có thể được phát lại.
2. **Soft reset qua Server:** `RESET` gửi `AT+QPOWD`; EG91 chờ `QUEC_QPOWD` rồi queue `SERVER_HOST_MSG_RESET_SOFT`. Host dọn status/diagnostic, tái khởi tạo CMUX và gọi `PowerReset()` nếu nhánh cho phép. Với EG91, lời gọi `PowerReset()` không phát xung như đã nêu; phần boot/recovery tiếp tục theo CMUX. Không coi soft reset này đồng nghĩa xung RESET pin.
3. **Hard reset EG91:** `SERVER_HOST_MSG_RESET_HARD` → dọn status/timer/pending → `Cmux_ReInitialize(FALSE,FALSE)` → `PowerOff()` → `DrvGPRS_PowerRF(OFF)` → chờ **60000 ms** → `PowerOn()` → báo `CELL_MODEM_EVENT_HW_RESET` → `Cmux_ReInitialize(TRUE,...)`. Nhánh modem khác có lịch chờ/tắt/bật khác.

**CMUX có FSM riêng:** trước khi Shell/Client/Server sử dụng các kênh logic, [`Cmux_Enexis.cpp`](src/Driver/Cmux/Src/Cmux_Enexis.cpp) chạy bootstrap AT, kiểm tra SIM bằng `AT+CPIN?`/nhập PIN theo nhánh, và bật multiplexing bằng `AT+CMUX=0,0,5,1270,10,0,30`. CMUX cũng có các đường gọi `PowerOff`, `PowerOn`, `PowerReset` để hồi phục. Vì vậy một lần power/reset thấy trên GPIO có thể đến từ CMUX recovery, startup hoặc ServerHost, không nhất thiết do state `RESET` của Server vừa được vào.

Các state lớn cho target này gồm `INIT`, `LISTENING`, `ACCEPTED`, `SEND`, `CLOSE`, `RESET`, `UDP_LISTENING`; các substate nằm trong [`ServerState.h`](src/Driver/USB_Modem/Inc/ServerState.h).

| State/substate | Sự kiện/điều kiện | Hành động và bước tiếp |
| --- | --- | --- |
| `INIT/WAIT_WAKEUP` | Auto connect bật; mode/calling window cho phép server hoặc Shell yêu cầu mở server | Bắt đầu chuỗi init. Mode `PERMANENTLY_CONNECT` và calling window ảnh hưởng thời điểm mở; không mặc định luôn mở |
| `INIT/ATE0`, `AT_CMEE`, `AT_K3`, `AT_CGDCONT`, `AT_KTCPCNX` | Timer phát AT, response hợp lệ | Đi substate tiếp. Với Quectel, sau `AT_CGDCONT` code đi thẳng `AT_KTCPCNX` để mở TCP listener bằng `AT+QIOPEN`; `AT_KTCPCFG`/`AT_KCNXTIMER` là nhánh Sierra, không nằm trên đường Quectel chính. Tên `K...` là tên enum lịch sử |
| `INIT → LISTENING` | Response socket mở thành công (hoặc URC open hợp lệ) | Lưu server session, thông báo trạng thái modem/server và chờ kết nối |
| `LISTENING/WAIT` | URC kết nối đi vào `HL7690_KTCP_SRVREQ` (parser ánh xạ URC Quectel) | Ghi session/subsession, xóa bộ đếm không kết nối, Quectel → `ACCEPTED`; nhánh Sierra có `LISTENING_CONNECT` để `AT+KTCPSTART` |
| `LISTENING/WAIT` | URC PDP disconnected/closed hoặc TCP listener closed | `RESET` |
| `LISTENING/WAIT` | Hết calling window, mode method invoked đã timeout sau disconnect, hoặc host đặt `IsCloseServer()` | `CLOSE` vì `CONFIG_FEATURE_GPRS_TCP_SERVER_CLOSE_NO_RESET=1`; các cấu hình khác có thể `RESET` |
| `ACCEPTED/WAIT`, `ACCEPTED_RCV` | URC data (`HL7690_KTCP_DATA`/Quectel) | `AT+QIRD=<session>,<len>`; payload được đưa lên `USB_Recv()` để DLMS xử lý; tiếp tục chờ hoặc đóng phiên theo response/URC |
| `ACCEPTED → SEND` | `SERVER_HOST_MSG_TX`, có payload | Quectel gửi `AT+QISEND`, chờ prompt, truyền payload; `SEND OK`/fail hoặc close URC quyết định quay lại `ACCEPTED`, `LISTENING` hay xử lý lỗi |
| `ACCEPTED → LISTENING` | Phiên con đóng, mất carrier hoặc chính sách calling window | Dọn session và quay chờ kết nối khác; lỗi mạng nặng → `RESET` |
| `CLOSE` | Yêu cầu đóng server | `CLOSE_WAIT` → `CLOSE_DELETE`; Quectel gửi `AT+QICLOSE` ở cả hai substate, sau `OK` hoặc `CME ERROR` host nhận `SERVER_HOST_MSG_CLOSE` và Shell nhận yêu cầu đóng PDP. Sierra dùng `AT+KTCPCLOSE`/`AT+KTCPDEL`; timeout có đường hard reset |
| `RESET` | Soft/hard reset từ host, timeout hoặc lỗi AT/URC | Dừng timer, dọn trạng thái, báo reset lên ứng dụng; `ResetSoftProcess()` tái khởi tạo CMUX và power reset, `ResetHardProcess()` dùng đường hard reset; sau thời gian chờ, quay `INIT` |
| `UDP_LISTENING` | Server cấu hình UDP | Chờ datagram, nhận qua `AT+QIRD`, gửi qua `AT+QISEND`; lỗi/timeout → `RESET` |

**Timer và retry:** `C_ServerStateInit::ProcessTimer()` bỏ ID timer cũ, phát AT theo substate và kiểm tra `SERVER_STATE_TX_MAX_RETRIES`; vượt mức sẽ `SetForceReattach()` rồi `RESET`. `LISTENING` có timer 5 giây; với auto connect, điều kiện đóng dựa vào mode và calling window. Nhánh không auto connect mới dùng `Connection_Watchdog_Timer_P3` để reset khi lâu không có kết nối. Các state `ACCEPTED`/`SEND` có xử lý riêng cho mất RX, send failure và close URC. Xem [`C_ServerStateListening`](src/Driver/USB_Modem/Src/ServerState.cpp), `C_ServerStateAccepted`, `C_ServerStateSend`, `C_ServerStateReset` trong cùng file.

## Message và điều kiện chéo quan trọng

- `SERVER_HOST_MSG_RESET_SOFT`/`RESET_HARD`: host ép về `INIT`, xóa diagnostic/status, báo `CELL_MODEM_EVENT_SW_RESET`/sự kiện tương ứng cho Application và tác động CMUX/nguồn modem. Đây là reset từ bên ngoài FSM, không đợi response socket.
- `SERVER_HOST_MSG_END_CALLING_WINDOW` cập nhật cờ hết cửa sổ; timer ở `LISTENING` đọc cờ đó để chuyển `CLOSE`.
- `SERVER_HOST_MSG_AUTO_CONNECT` cập nhật mode kết nối và có thể đặt `m_bIsCloseServer`; timer ở `LISTENING` sẽ áp dụng.
- Khi Client đã gửi payload sau prompt `CONNECT`, Client gửi `SERVER_HOST_MSG_WAKEUP` để Server xử lý yêu cầu phối hợp; Shell cũng nhận các yêu cầu mở PDP/server khi auto connect bật.
- `C_ModemDataParser` có thể nhận response của lệnh đang chờ và URC bất đồng bộ. Vì vậy điều kiện chuyển state không chỉ là `OK`; mã kiểm tra ID message, session ID, status và cờ `m_bWaitResponse` tại mỗi substate.

Các tên `HL7690_*` tồn tại trong parser/enum dùng chung dù target là Quectel. Nên xác định modem từ macro phần cứng và lệnh bên trong nhánh `#if`, không từ tên response ID.

## Thứ tự gọi thực tế: khởi tạo, kết nối và dữ liệu

**Quy ước:** `→` là gọi hàm trực tiếp; `⇢` là gửi message vào queue. Sau `MsgSend()`, task nhận xử lý khi được scheduler chạy; không có bảo đảm ba FSM hoàn tất init nối tiếp theo thứ tự gọi `*_Task_Init()`. Phần source tra cứu phía sau giữ các nhánh `#if`; các chuỗi dưới đây theo target OMAN/EG91 với `CONFIG_HARDWARE_HAS_USB_CPU=0`.

### 1. Khởi tạo Application và nền modem

1. Phần khởi tạo phần cứng ở [`Meter_Energa.cpp`](src/Application/meterapp/Src/Meter_Energa.cpp#L523) → `cell_init()` → `DrvGPRS_Init()` → `DrvGPRS_ReInit()` → cấu hình GPIO/UART/NVIC/DMA RX và timer. Tới đây mới chuẩn bị driver, chưa chứng minh modem đăng ký mạng.
2. [`C_Meter_Base::Startup()`](src/Application/meterapp/Src/Meter_Base_App.cpp#L784) → `cell_startup()` → `Task_AppComm_Init()` → `ModemHost_Task_Init()`.
3. `ModemHost_Task_Init()` cấp nguồn → `C_Cmux::PreStartup()` → `C_Cmux::Startup()` → khởi tạo ModemHost rồi các host theo thứ tự **Server, SMS, Shell, Client** (nhánh thông thường, test mode có điều kiện riêng).
4. CMUX bootstrap: gửi chuỗi AT cấu hình cổng/URC/SIM → chờ response từng lệnh → `AT+CMUX=...` → mở các kênh logic → các host tiếp tục FSM trên kênh của mình. Shell init đăng ký mạng, Client init chờ READY, Server auto connect chờ WAKEUP.
5. `C_Meter_Base::Startup()` tiếp tục → tạo context CLI → `C_ObjMgr::Startup()` → `C_GprsMgr::Startup()` → `cell_context_create()` → các yêu cầu thông tin modem/SIM/mạng → `Set_modem_config()` → `network_connect()` → `cell_connect()`.
6. [`cell_connect()`](src/Driver/AppUsbCom/Src/cell_api.cpp#L158) chuyển `cell_settings` thành `ST_Modem_Config`, đọc port TCP/UDP từ COSEM; lần đầu gọi `WakeupUsbModem()` rồi gửi cấu hình modem qua adapter. Vì WAKEUP và cấu hình qua queue, Shell dùng pending request để chờ READY thay vì giả định modem đã sẵn sàng khi API trả về.

### 2. Mở PDP rồi mở Client để push

![Sequence mở PDP, kết nối Client và gửi push](documentation/diagrams/cellular/pdp-client-push-sequence.svg)

[Bản draw.io có thể chỉnh sửa](documentation/diagrams/cellular/pdp-client-push-sequence.drawio)

Chi tiết ở [`ShellHost.cpp`](src/Driver/USB_Modem/Src/ShellHost.cpp#L608): `RxWakeUpProcess()` đặt `SHELL_PREPARE_CLIENT`; `RxOpenPDPContext()` chỉ gọi `ProcessRxCmd(OPEN_PDP)` nếu Shell ở `INIT/READY`. Trong PDP, nếu context đã active thì bỏ bước activate; nếu IP còn rỗng thì quay activate sau 3 giây. `PDP_END` queue reset Shell; `ResetProcess()` chọn yêu cầu pending, rồi `RxPrepareClient()` queue WAKEUP sang Client. Khi prompt gửi payload tới Client, code còn queue `SERVER_HOST_MSG_WAKEUP`. Sơ đồ là đường thành công; không biểu diễn mọi retry/error hay lịch xen kẽ task.

### 3. Mở Server và phục vụ DLMS

1. Application → [`StartServer()`](src/Driver/AppUsbCom/Src/AppComInterface.cpp#L1007) ⇢ `SHELL_HOST_MSG_RX_START_SERVER`.
2. Shell → `RxStartServer()` đặt pending `SHELL_OPEN_SERVER` → `RxOpenPDPContext()` → mở/kiểm tra PDP và IP.
3. Kết thúc PDP → Shell `ResetProcess()` ⇢ `RX_OPEN_SERVER` → `RxOpenServer()` kiểm tra READY, reset cờ end window ⇢ `SERVER_HOST_MSG_WAKEUP`.
4. ServerHost → `WakeupProcess()` → `C_ServerStateInit::ProcessRx()` → từ `WAIT_WAKEUP` bắt đầu AT init → `AT+QIOPEN` TCP LISTENER/UDP SERVICE. `OK` đơn thuần chưa đủ: `QUEC_OPEN` với status READY mới đặt status LISTENING và chuyển `LISTENING` hoặc `UDP_LISTENING`.
5. Modem nhận kết nối TCP → URC → UART RX → CMUX kênh Server → parser tạo message → `C_ServerStateListening::ProcessRx()` ghi session và chuyển `ACCEPTED`.
6. Có URC data → Server gửi `AT+QIRD` → nhận độ dài/payload → `USB_Recv()` chuyển lên adapter/DLMS. DLMS xử lý request, tạo response rồi gửi TX xuống Server.
7. Server `ProcessTx()` → lấy payload, chuyển `SEND` → timer gửi `AT+QISEND` → prompt → `Cmux_SendGPRSData()` truyền bytes → `SEND OK` → `ACCEPTED`. Mất carrier/lỗi thường đóng phiên con và về `LISTENING`; lỗi nền mạng/timeout có đường `RESET`.

### 4. Một AT command đi xuống và response quay lại

`ProcessTimer()` → `SendDataWaitResponse()` → `g_modemAdpatorShell/Client/Server.SendData()` → lưu mẫu response → `Cmux_SendMsgToChannel()` ⇢ queue CMUX → đóng khung đúng DLCI → `Cmux_SendDataToModem()` → `DrvGprs_SendData()` → `DrvGPRS_DMA_Tx_Config()` → UART/module.

Chiều về: DMA RX/timer driver → callback `Cmux_DmaRxISR()` ⇢ queue RX CMUX → tách frame/DLCI ⇢ `*_HOST_MSG_RX_CMUX` → `ProcessRxFromCmux()` → `ModemAdaptor::ParseRxFromCmux()`/`RxCallback()` → `ModemDataParser::Parse()` → match response/URC ⇢ host RX → `ProcessRx()` → cập nhật dữ liệu hoặc chọn state/substate mới. **UART gửi xong** (`TX_COMPLETE`) và **modem thực hiện lệnh xong** (`OK`/URC) là hai event khác nhau.

### 5. Đóng và phục hồi lỗi theo thứ tự

- Client idle/send fail → `CLOSE_SESSION` → `DELETE_SESSION` → `AT+QICLOSE` → `OK` → queue Client reset ⇢ Shell `CLOSE_PDP_CONTEXT`.
- Server hết calling window → `CLOSE_WAIT` → `AT+QICLOSE` → `OK` → `CLOSE_DELETE` → `AT+QICLOSE` → `OK`/`CME ERROR` → queue Server CLOSE và Shell CLOSE_PDP.
- Shell `CLOSE_PDP_AT_KCNXDOWN` **chỉ gửi `AT+QIDEACT=1` nếu cả Server và Client đều INIT**; nếu còn một socket FSM hoạt động thì reset tác vụ Shell và không deactivate PDP ở nhánh đó.
- Shell/Client phát hiện timeout → queue Server hard reset → `ServerHost::ResetHardProcess()` → dừng CMUX, power/RF OFF → đợi 60 giây → power ON → CMUX bootstrap lại → các FSM khởi tạo lại. Chi tiết GPIO nằm ở phần trên.

## Ngoài Shell, Client, Server còn ai tương tác với modem?

| Thành phần | Tương tác với module / vai trò | Áp dụng target này |
| --- | --- | --- |
| [`C_Cmux`](src/Driver/Cmux/Src/Cmux_Enexis.cpp) | FSM bootstrap AT trước multiplexing; quản lý DLCI/frame/queue, route RX/TX, recovery power/reset. Lệnh Quectel gồm `ATE0`, `QURCCFG`, `AT&C0`, `AT&D0`, `AT+IFC=0,0`, `QDSIM`, `CPIN`, `CMUX` theo điều kiện | Có source trong target; là thành phần trực tiếp nói chuyện modem trước ba FSM |
| [`C_ModemHost`](src/Driver/USB_Modem/Src/ModemHost.cpp) | Quản lý lifecycle/nguồn, khởi tạo host, xử lý yêu cầu power cut; wrapper `PowerOn/Off/Reset` | Có |
| [`drv_gprs.c`](src/Driver/Gprs/Src/drv_gprs.c) | Ghi GPIO và cấu hình UART/DMA, chuyển bytes, callback RX/TX | Có; là lớp chạm phần cứng thực tế |
| [`SmsHost/SmsState`](src/Driver/USB_Modem/Src/SmsState.cpp) | FSM thứ tư, kênh SMS riêng: INIT/RECIEVING/PUSH. Cấu hình `CMEE`, `CRC`, `CLIP`, SMS PDU `CMGF=0`, `CNMI`, `CMGD`, `CSCA`; gửi `AT+CMGS` và PDU SMS. Nhánh push-and-shutdown có thể gọi `ModemHost::PowerOff()` | Source có trong target; `CONFIG_FEATURE_ADD_SMS_CHANNEL=1`. Hành vi shutdown còn phụ thuộc feature tương ứng |
| [`ModemAdaptor/ModemDataParser`](src/Driver/USB_Modem/Src/ModemAdaptor.cpp) | Ghép kênh với response mong đợi, tách token/message/URC, chuyển payload. Parser không tự ra quyết định cấu hình mạng/power | Có |
| [`AppComInterface`](src/Driver/AppUsbCom/Src/AppComInterface.cpp), `cell_api.cpp`, AppComm | Adapter lệnh cấu hình, start server, wakeup client, payload, callback/status; gửi message cho các host | Có; ở target không có USB CPU riêng nên tên USB không đồng nghĩa đường truyền modem qua USB |
| GprsMgr, COSEM, push, DLMS, CLI | Chủ thể yêu cầu kết nối/cấu hình/monitoring/test; gọi API/adapter. CLI có thể khởi động test hoặc điều khiển modem qua các lớp trên | Có, tùy feature/lệnh |
| `CsdHost/CsdState` | Stack cho modem ngoài/RS232/CSD, có AT và đường power riêng | Có trong repo nhưng không được liệt kê source ở target OMAN này; `CONFIG_HARDWARE_RS232=0` |
| `C_AtCmd` (`AT_Cmd.cpp`) | Engine AT cũ/tables, tương tác UART/CMUX ở cấu hình dùng nó | File có trong repo nhưng không được liệt kê source ở target này; không nhầm với bootstrap AT được viết trong CMUX hiện tại |
| `src/Middleware/cmcell` | Stack cellular chung khác, có state machines/API riêng | Target sử dụng API tương thích `Driver/AppUsbCom/Src/cell_api.cpp`; source cmcell không nằm trong danh sách compile target |

CMUX định nghĩa kênh control **0**, Shell **1**, Server **2**, Client **3**, SMS **4** khi cấu hình tương ứng. **DLCI Client=3 không phải socket ID Client=2**: một số chọn kênh UART multiplexing, số kia chọn socket IP trong modem. MTR chỉ cung cấp chức năng/dữ liệu đo lường cho Application; luồng AT/GPIO đang phân tích không chạy trong MTR.

## Tra cứu chi tiết từng substate Client theo handler

Phần này liệt kê riêng từng substate trong source, bao gồm nhánh OMAN và các nhánh sản phẩm khác. Đây là bản tra cứu **source chưa tiền xử lý**: các lệnh/đích khác nhau dưới `#if` là lựa chọn thay thế, không phải đều được chạy liên tiếp. Phần phân tích OMAN phía trên xác định đường áp dụng. Điều kiện, response và đích được liệt kê theo từng handler; dùng liên kết source để xem cặp điều kiện–hành động và phạm vi `switch` chính xác. Các handler dùng chung như kiểm tra timer ID/retry ở đầu hàm vẫn áp dụng dù không lặp lại ở từng hàng. `ProcessRxFromCmux` chỉ chuyển bytes vào parser, nên không lặp lại ở đây.

### C_ClientStateInit

#### `CLIENT_STATE_INIT_WAIT_STARTUP`

**ProcessRx** — [source L273](src/Driver/USB_Modem/Src/ClientState.cpp#L273).

- Giữ state hiện tại hoặc chỉ làm bước chờ/thoát của handler; không phát AT hay chuyển state tại nhánh này.

**ProcessConnectToServer** — [source L320](src/Driver/USB_Modem/Src/ClientState.cpp#L320).

- Giữ state hiện tại hoặc chỉ làm bước chờ/thoát của handler; không phát AT hay chuyển state tại nhánh này.

**ProcessTimer** — [source L368](src/Driver/USB_Modem/Src/ClientState.cpp#L368).

- Đích chuyển state/substate có trong handler: `CLIENT_STATE_INIT_ATE0`.

#### `CLIENT_STATE_INIT_ATE0`

**ProcessRx** — [source L276](src/Driver/USB_Modem/Src/ClientState.cpp#L276).

- Response/URC và mã status được kiểm tra: `HL7690_OK`.

- Đích chuyển state/substate có trong handler: `CLIENT_STATE_READY`.

- Các điều kiện kiểm tra: `m_bWaitResponse`, `HL7690_OK == pMsg->id`.

**ProcessConnectToServer** — [source L322](src/Driver/USB_Modem/Src/ClientState.cpp#L322).

- Giữ state hiện tại hoặc chỉ làm bước chờ/thoát của handler; không phát AT hay chuyển state tại nhánh này.

**ProcessTimer** — [source L372](src/Driver/USB_Modem/Src/ClientState.cpp#L372).

- Lệnh gửi/tạo: `ATE0`.

- Timer được chọn: `CLIENT_STATE_TIMER_SHORT_AT_COMMAND`.

#### `CLIENT_STATE_READY`

**ProcessRx** — [source L286](src/Driver/USB_Modem/Src/ClientState.cpp#L286).

- Giữ state hiện tại hoặc chỉ làm bước chờ/thoát của handler; không phát AT hay chuyển state tại nhánh này.

**ProcessConnectToServer** — [source L325](src/Driver/USB_Modem/Src/ClientState.cpp#L325).

- Đích chuyển state/substate có trong handler: `CLIENT_STATE_CONNECT`.

**ProcessTimer** — [source L377](src/Driver/USB_Modem/Src/ClientState.cpp#L377).

- Timer được chọn: `CLIENT_STATE_TIMER_FIVE_SECONDS`.

- Tác động/callback: `Update_tcp_client_status`.

### C_ClientStateConnect

#### `CLIENT_STATE_AT_KTCPCFG`

**ProcessRx** — [source L444](src/Driver/USB_Modem/Src/ClientState.cpp#L444).

- Response/URC và mã status được kiểm tra: `QUEC_OPEN`, `HL7690_ERROR`, `HL7690_CME_ERROR`, `HL7690_KTCPCFG`.

- Đích chuyển state/substate có trong handler: `CLIENT_STATE_AT_GET_ADDRESS`, `CLIENT_STATE_AT_KTCPCNX`, `CLIENT_STATE_DELETE_SESSION`, `CLIENT_STATE_WAITING_PING_CAPTURE`.

- Tác động/callback: `Cmux_Send_Modem_Event_To_App`, `Get_LTE_monitoring`, `Get_modem_statistics`, `Reset`.

- Các điều kiện kiểm tra: `m_bWaitResponse`, `QUEC_OPEN == pMsg->id && pMsg->rx.params[2].s32Param == 0`, `TCP_STATUS_READY == status`, `C_ClientHost::GetInstance().GetTypeTcpConnection() == CLIENT_TCP_LTE_PING`, `(QUEC_OPEN == pMsg->id && pMsg->rx.params[2].s32Param != 0) \|\| (HL7690_ERROR == pMsg->id) \|\| (HL7690_CME_ERROR == pMsg->id)`, `HL7690_KTCPCFG == pMsg->id`, `(HL7690_ERROR == pMsg->id) \|\| (HL7690_CME_ERROR == pMsg->id)`, `TCP_ERROR_SOCKET_IDENTITY_HAS_BEEN_USED == pMsg->rx.params[2].s32Param`.

**ProcessTimer** — [source L952](src/Driver/USB_Modem/Src/ClientState.cpp#L952).

- Lệnh gửi/tạo: `AT+KTCPCFG=1,0,"%s",%s`.

**ProcessTimer** — [source L960](src/Driver/USB_Modem/Src/ClientState.cpp#L960).

- Lệnh gửi/tạo: `AT+QIOPEN=1,2,"TCP","%s",%s,0,0`, `AT+QIOPEN=1,2,"UDP","%s",%s,0,0`, `AT+KTCPCFG=1,0,"%s",%s`.

- Timer được chọn: `CLIENT_STATE_TIMER_150_SECOND`, `CLIENT_STATE_TIMER_LONG_AT_COMMAND`.

- Các điều kiện kiểm tra: `C_ClientHost::GetInstance().GetTypeTcpConnection() == CLIENT_TCP_LTE_PING`, `pModemConfig->u08TypePush == E_TCP_SERVICE`, `pModemConfig->u08TypePush == E_UDP_SERVICE`.

#### `CLIENT_STATE_AT_KTCPCNX`

**ProcessRx** — [source L505](src/Driver/USB_Modem/Src/ClientState.cpp#L505).

- Response/URC và mã status được kiểm tra: `HL7690_CME_ERROR`.

- Đích chuyển state/substate có trong handler: `CLIENT_STATE_DELETE_SESSION`, `CLIENT_STATE_AT_GET_ADDRESS`.

- Tác động/callback: `Cmux_Send_Modem_Event_To_App`.

- Các điều kiện kiểm tra: `m_bWaitResponse`, `HL7690_CME_ERROR == pMsg->id`.

**ProcessTimer** — [source L994](src/Driver/USB_Modem/Src/ClientState.cpp#L994).

- Lệnh gửi/tạo: `AT+KTCPCNX=%i`.

- Timer được chọn: `CLIENT_STATE_TIMER_LONG_AT_COMMAND`.

#### `CLIENT_STATE_AT_GET_ADDRESS`

**ProcessRx** — [source L524](src/Driver/USB_Modem/Src/ClientState.cpp#L524).

- Response/URC và mã status được kiểm tra: `QUEC_QIACT`, `HL7690_KCGPADDR`, `HL7690_OK`.

- Đích chuyển state/substate có trong handler: `CLIENT_STATE_WAIT_FOR_DATA`.

- Tác động/callback: `Update_modem_status`, `Update_tcp_client_status`.

- Các điều kiện kiểm tra: `m_bWaitResponse`, `QUEC_QIACT == pMsg->id`, `HL7690_KCGPADDR == pMsg->id`, `2 < len`, `HL7690_OK == pMsg->id`, `C_ClientHost::GetInstance().GetTypeTcpConnection() == CLIENT_TCP_LTE_PUSH`.

**ProcessTimer** — [source L1001](src/Driver/USB_Modem/Src/ClientState.cpp#L1001).

- Lệnh gửi/tạo: `AT+QIACT?`, `AT+KCGPADDR`.

- Timer được chọn: `SHELL_STATE_TIMER_QIACT_AT_COMMAND`.

#### `CLIENT_STATE_WAIT_FOR_DATA`

**ProcessRx** — [source L567](src/Driver/USB_Modem/Src/ClientState.cpp#L567).

- Lệnh gửi/tạo: `AT+QIRD=%i,%i`.

- Response/URC và mã status được kiểm tra: `HL7690_KTCP_DATA`, `QUEC_QIRD`.

- Timer được chọn: `SERVER_STATE_TIMER_SHORT_AT_COMMAND`.

- Đích chuyển state/substate có trong handler: `CLIENT_STATE_ACCEPTED_RCV`.

**ProcessTimer** — [source L1011](src/Driver/USB_Modem/Src/ClientState.cpp#L1011).

- Timer được chọn: `CLIENT_STATE_TIMER_FIVE_SECONDS`.

- Đích chuyển state/substate có trong handler: `CLIENT_STATE_AT_KTCPSND`, `CLIENT_STATE_CLOSE_SESSION`.

- Tác động/callback: `Reply_get_LTE_monitoring`, `Reply_get_modem_statistics`, `Get_LTE_monitoring`, `Get_modem_statistics`.

- Các điều kiện kiểm tra: `m_bIsDataReady`, `!m_bIsDataReady`, `CLIENT_STATE_TIMEOUT_PING_START_MONITORING == m_u32TcpClientCounter`, `C_ClientHost::GetInstance().GetTypeTcpConnection() == CLIENT_TCP_LTE_PING`, `10000 <= m_u32TcpClientCounter`.

#### `CLIENT_STATE_VERIFY_ACKED_PACKET`

**ProcessRx** — [source L602](src/Driver/USB_Modem/Src/ClientState.cpp#L602).

- Response/URC và mã status được kiểm tra: `QUEC_QISEND_ACKED`, `QUEC_CLOSE`, `HL7690_CME_ERROR`, `HL7690_ERROR`.

- Đích chuyển state/substate có trong handler: `CLIENT_STATE_WAIT_FOR_DATA`, `CLIENT_STATE_VERIFY_ACKED_PACKET (delay 300)`, `CLIENT_STATE_CLOSE_SESSION`.

- Tác động/callback: `Cmux_Ack_Push`.

- Các điều kiện kiểm tra: `u32TotSentData == u32TotAckedData`, `u32ClosedSessionNum == 2`.

**ProcessTimer** — [source L1063](src/Driver/USB_Modem/Src/ClientState.cpp#L1063).

- Lệnh gửi/tạo: `AT+QISEND=2,0`.

- Timer được chọn: `SERVER_STATE_TIMER_SHORT_AT_COMMAND`.

#### `CLIENT_STATE_ACCEPTED_RCV`

**ProcessRx** — [source L637](src/Driver/USB_Modem/Src/ClientState.cpp#L637).

- Response/URC và mã status được kiểm tra: `QUEC_CLOSE`, `HL7690_CME_ERROR`, `HL7690_ERROR`, `HL7690_EXIT_DATA`, `HL7690_DATA`.

- Đích chuyển state/substate có trong handler: `CLIENT_STATE_CLOSE_SESSION`, `CLIENT_STATE_DELETE_SESSION`, `CLIENT_STATE_WAIT_FOR_DATA`.

- Tác động/callback: `USB_Recv`.

- Các điều kiện kiểm tra: `u32ClosedSessionNum == 2`, `0 == memcmp(pMsg->rx.data.buf, "--EOF--Pattern--", 16)`, `0 == memcmp(pMsg->rx.data.buf, "\r\nOK\r\n", 6)`, `u16DataLen < (pMsg->rx.data.len)`, `u16RemainingLen <= pMsg->rx.data.len`.

#### `CLIENT_STATE_AT_KTCPSND`

**ProcessRx** — [source L692](src/Driver/USB_Modem/Src/ClientState.cpp#L692).

- Response/URC và mã status được kiểm tra: `HL7690_CONNECT`, `HL7690_OK`, `QUEC_SEND_OK`, `QUEC_SEND_FAIL`, `HL7690_ERROR`, `HL7690_CME_ERROR`, `HL7690_KTCP_NOTIF`, `HL7690_TCP_NOTIF_STR`, `HL7690_TCP_NOTIF_DISCONNECT`, `HL7690_EXIT_DATA`.

- Đích chuyển state/substate có trong handler: `CLIENT_STATE_WAIT_FOR_DATA`, `CLIENT_STATE_VERIFY_ACKED_PACKET`, `CLIENT_STATE_CLOSE_SESSION`.

- Message phối hợp: `SERVER_HOST_MSG_WAKEUP`.

- Tác động/callback: `Cmux_Ack_Push`.

- Các điều kiện kiểm tra: `pModemConfig->u08TypePush == E_TCP_SERVICE`, `pPush != NULL`, `pPush->GetPushOperationMethod() == E_CONFIRMED_RETRY_ON_MISSING_CONFIRMATION`, `notif == HL7690_TCP_NOTIF_DISCONNECT`.

**ProcessTimer** — [source L1045](src/Driver/USB_Modem/Src/ClientState.cpp#L1045).

- Lệnh gửi/tạo: `AT+QISEND=2,%i`, `AT+KTCPSND=%i,%i`.

- Timer được chọn: `SERVER_STATE_TIMER_SHORT_AT_COMMAND`.

#### `CLIENT_STATE_CLOSE_SESSION`

**ProcessRx** — [source L796](src/Driver/USB_Modem/Src/ClientState.cpp#L796).

- Response/URC và mã status được kiểm tra: `HL7690_KTCP_NOTIF`, `HL7690_TCP_NOTIF_STR`, `HL7690_EXIT_DATA`, `HL7690_OK`, `HL7690_ERROR`, `HL7690_CME_ERROR`.

- Đích chuyển state/substate có trong handler: `CLIENT_STATE_DELETE_SESSION`.

**ProcessTimer** — [source L1070](src/Driver/USB_Modem/Src/ClientState.cpp#L1070).

- Lệnh gửi/tạo: `AT+KTCPCLOSE=%i`.

- Đích chuyển state/substate có trong handler: `CLIENT_STATE_DELETE_SESSION`.

#### `CLIENT_STATE_DELETE_SESSION`

**ProcessRx** — [source L829](src/Driver/USB_Modem/Src/ClientState.cpp#L829).

- Response/URC và mã status được kiểm tra: `HL7690_OK`.

- Đích chuyển state/substate có trong handler: `CLIENT_STATE_WAITING_PING_CAPTURE`.

- Message phối hợp: `SHELL_HOST_MSG_CLOSE_PDP_CONTEXT`.

- Tác động/callback: `Update_modem_status`, `Cmux_Send_Modem_Event_To_App`, `Update_tcp_client_status`, `Reset`.

- Các điều kiện kiểm tra: `m_bWaitResponse`, `HL7690_OK == pMsg->id`, `eStateServer == SERVER_STATE_INIT`, `(C_ClientHost::GetInstance() .GetTypeTcpConnection() == CLIENT_TCP_LTE_PING) && (m_PingSuccess != TRUE)`.

**ProcessTimer** — [source L1080](src/Driver/USB_Modem/Src/ClientState.cpp#L1080).

- Lệnh gửi/tạo: `AT+QICLOSE=%i`, `AT+KTCPDEL=%i`.

- Timer được chọn: `CLIENT_STATE_TIMER_LONG_AT_COMMAND`.

#### `CLIENT_STATE_WAITING_PING_CAPTURE`

**ProcessRx** — [source L874](src/Driver/USB_Modem/Src/ClientState.cpp#L874).

- Giữ state hiện tại hoặc chỉ làm bước chờ/thoát của handler; không phát AT hay chuyển state tại nhánh này.

**ProcessTimer** — [source L1092](src/Driver/USB_Modem/Src/ClientState.cpp#L1092).

- Timer được chọn: `CLIENT_STATE_TIMER_FIVE_SECONDS`.

- Tác động/callback: `Reset`.

- Các điều kiện kiểm tra: `(m_bTcpConnectionPing == FALSE) \|\| (CLIENT_STATE_TIMEOUT_PING_DURATION <= m_u32TcpClientCounter)`, `m_PingSuccess != TRUE`.

### C_TestStateInit

#### `TEST_STATE_INIT_WAIT_STARTUP`

**ProcessRx** — [source L1154](src/Driver/USB_Modem/Src/ClientState.cpp#L1154).

- Giữ state hiện tại hoặc chỉ làm bước chờ/thoát của handler; không phát AT hay chuyển state tại nhánh này.

**ProcessTimer** — [source L1213](src/Driver/USB_Modem/Src/ClientState.cpp#L1213).

- Đích chuyển state/substate có trong handler: `TEST_STATE_INIT_ATE0`.

#### `TEST_STATE_INIT_ATE0`

**ProcessRx** — [source L1157](src/Driver/USB_Modem/Src/ClientState.cpp#L1157).

- Response/URC và mã status được kiểm tra: `HL7690_OK`.

- Đích chuyển state/substate có trong handler: `TEST_STATE_INIT_AT_CMEE`.

- Các điều kiện kiểm tra: `m_bWaitResponse`, `HL7690_OK == pMsg->id`.

**ProcessTimer** — [source L1218](src/Driver/USB_Modem/Src/ClientState.cpp#L1218).

- Lệnh gửi/tạo: `ATE0`.

#### `TEST_STATE_INIT_AT_CMEE`

**ProcessRx** — [source L1166](src/Driver/USB_Modem/Src/ClientState.cpp#L1166).

- Response/URC và mã status được kiểm tra: `HL7690_OK`.

- Đích chuyển state/substate có trong handler: `TEST_STATE_LISTENING`.

- Các điều kiện kiểm tra: `m_bWaitResponse`, `HL7690_OK == pMsg->id`.

**ProcessTimer** — [source L1222](src/Driver/USB_Modem/Src/ClientState.cpp#L1222).

- Lệnh gửi/tạo: `AT+CMEE=1`.

### C_TestStateListening

#### `TEST_STATE_LISTENING_WAIT`

**ProcessRx** — [source L1272](src/Driver/USB_Modem/Src/ClientState.cpp#L1272).

- Giữ state hiện tại hoặc chỉ làm bước chờ/thoát của handler; không phát AT hay chuyển state tại nhánh này.

**ProcessTx** — [source L1395](src/Driver/USB_Modem/Src/ClientState.cpp#L1395).

- Đích chuyển state/substate có trong handler: `TEST_STATE_LISTENING_GET_IMSI`, `TEST_STATE_LISTENING_GET_IMEI`, `TEST_STATE_LISTENING_GET_CCID`, `TEST_STATE_LISTENING_GET_SN`, `TEST_STATE_LISTENING_GET_HWREV`, `TEST_STATE_LISTENING_GET_FWVERS`, `TEST_STATE_LISTENING_TEST_TX`, `TEST_STATE_LISTENING_TEST_RX`, `TEST_STATE_LISTENING_SET_FUN`, `TEST_STATE_LISTENING_RESET`, `TEST_STATE_LISTENING_GENERIC_COMMAND`.

**ProcessTimer** — [source L1491](src/Driver/USB_Modem/Src/ClientState.cpp#L1491).

- Giữ state hiện tại hoặc chỉ làm bước chờ/thoát của handler; không phát AT hay chuyển state tại nhánh này.

#### `TEST_STATE_LISTENING_GET_IMSI`

**ProcessRx** — [source L1276](src/Driver/USB_Modem/Src/ClientState.cpp#L1276).

- Response/URC và mã status được kiểm tra: `HL7690_CIMI`.

- Đích chuyển state/substate có trong handler: `TEST_STATE_LISTENING_WAIT`.

- Các điều kiện kiểm tra: `HL7690_CIMI == pMsg->id`.

**ProcessTx** — [source L1443](src/Driver/USB_Modem/Src/ClientState.cpp#L1443).

- Giữ state hiện tại hoặc chỉ làm bước chờ/thoát của handler; không phát AT hay chuyển state tại nhánh này.

**ProcessTimer** — [source L1494](src/Driver/USB_Modem/Src/ClientState.cpp#L1494).

- Lệnh gửi/tạo: `AT+CIMI`.

#### `TEST_STATE_LISTENING_GET_IMEI`

**ProcessRx** — [source L1287](src/Driver/USB_Modem/Src/ClientState.cpp#L1287).

- Response/URC và mã status được kiểm tra: `HL7690_CGSN`.

- Đích chuyển state/substate có trong handler: `TEST_STATE_LISTENING_WAIT`.

- Các điều kiện kiểm tra: `HL7690_CGSN == pMsg->id`.

**ProcessTx** — [source L1445](src/Driver/USB_Modem/Src/ClientState.cpp#L1445).

- Giữ state hiện tại hoặc chỉ làm bước chờ/thoát của handler; không phát AT hay chuyển state tại nhánh này.

**ProcessTimer** — [source L1498](src/Driver/USB_Modem/Src/ClientState.cpp#L1498).

- Lệnh gửi/tạo: `AT+CGSN`.

#### `TEST_STATE_LISTENING_GET_CCID`

**ProcessRx** — [source L1298](src/Driver/USB_Modem/Src/ClientState.cpp#L1298).

- Response/URC và mã status được kiểm tra: `HL7690_CCID`.

- Đích chuyển state/substate có trong handler: `TEST_STATE_LISTENING_WAIT`.

- Các điều kiện kiểm tra: `HL7690_CCID == pMsg->id`.

**ProcessTx** — [source L1446](src/Driver/USB_Modem/Src/ClientState.cpp#L1446).

- Giữ state hiện tại hoặc chỉ làm bước chờ/thoát của handler; không phát AT hay chuyển state tại nhánh này.

**ProcessTimer** — [source L1502](src/Driver/USB_Modem/Src/ClientState.cpp#L1502).

- Lệnh gửi/tạo: `AT+CCID`, `AT+CCID?`.

#### `TEST_STATE_LISTENING_GET_SN`

**ProcessRx** — [source L1310](src/Driver/USB_Modem/Src/ClientState.cpp#L1310).

- Response/URC và mã status được kiểm tra: `HL7690_CGSN`.

- Đích chuyển state/substate có trong handler: `TEST_STATE_LISTENING_WAIT`.

- Các điều kiện kiểm tra: `HL7690_CGSN == pMsg->id`.

**ProcessTx** — [source L1447](src/Driver/USB_Modem/Src/ClientState.cpp#L1447).

- Giữ state hiện tại hoặc chỉ làm bước chờ/thoát của handler; không phát AT hay chuyển state tại nhánh này.

**ProcessTimer** — [source L1510](src/Driver/USB_Modem/Src/ClientState.cpp#L1510).

- Lệnh gửi/tạo: `AT+KGSN=3`.

#### `TEST_STATE_LISTENING_GET_HWREV`

**ProcessRx** — [source L1321](src/Driver/USB_Modem/Src/ClientState.cpp#L1321).

- Response/URC và mã status được kiểm tra: `HL7690_CGSN`.

- Đích chuyển state/substate có trong handler: `TEST_STATE_LISTENING_WAIT`.

- Các điều kiện kiểm tra: `HL7690_CGSN == pMsg->id`.

**ProcessTx** — [source L1448](src/Driver/USB_Modem/Src/ClientState.cpp#L1448).

- Giữ state hiện tại hoặc chỉ làm bước chờ/thoát của handler; không phát AT hay chuyển state tại nhánh này.

**ProcessTimer** — [source L1514](src/Driver/USB_Modem/Src/ClientState.cpp#L1514).

- Lệnh gửi/tạo: `AT+HWREV?`.

#### `TEST_STATE_LISTENING_GET_FWVERS`

**ProcessRx** — [source L1332](src/Driver/USB_Modem/Src/ClientState.cpp#L1332).

- Response/URC và mã status được kiểm tra: `HL7690_CGSN`.

- Đích chuyển state/substate có trong handler: `TEST_STATE_LISTENING_WAIT`.

- Các điều kiện kiểm tra: `HL7690_CGSN == pMsg->id`.

**ProcessTx** — [source L1449](src/Driver/USB_Modem/Src/ClientState.cpp#L1449).

- Giữ state hiện tại hoặc chỉ làm bước chờ/thoát của handler; không phát AT hay chuyển state tại nhánh này.

**ProcessTimer** — [source L1518](src/Driver/USB_Modem/Src/ClientState.cpp#L1518).

- Lệnh gửi/tạo: `AT+CGMR`, `ATI3`, `ATI1`.

#### `TEST_STATE_LISTENING_TEST_TX`

**ProcessRx** — [source L1343](src/Driver/USB_Modem/Src/ClientState.cpp#L1343).

- Response/URC và mã status được kiểm tra: `HL7690_OK`.

- Đích chuyển state/substate có trong handler: `TEST_STATE_LISTENING_WAIT`.

- Các điều kiện kiểm tra: `HL7690_OK == pMsg->id`.

**ProcessTx** — [source L1450](src/Driver/USB_Modem/Src/ClientState.cpp#L1450).

- Giữ state hiện tại hoặc chỉ làm bước chờ/thoát của handler; không phát AT hay chuyển state tại nhánh này.

**ProcessTimer** — [source L1529](src/Driver/USB_Modem/Src/ClientState.cpp#L1529).

- Lệnh gửi/tạo: `AT+WMTXPOWER=1,%u,%u,%u,%u`, `AT+WMTXPOWER=0`.

- Các điều kiện kiểm tra: `m_stTestRequest.enable`.

#### `TEST_STATE_LISTENING_TEST_RX`

**ProcessRx** — [source L1351](src/Driver/USB_Modem/Src/ClientState.cpp#L1351).

- Response/URC và mã status được kiểm tra: `HL7690_WMRXPOWER`.

- Đích chuyển state/substate có trong handler: `TEST_STATE_LISTENING_WAIT`.

- Các điều kiện kiểm tra: `HL7690_WMRXPOWER == pMsg->id`.

**ProcessTx** — [source L1451](src/Driver/USB_Modem/Src/ClientState.cpp#L1451).

- Giữ state hiện tại hoặc chỉ làm bước chờ/thoát của handler; không phát AT hay chuyển state tại nhánh này.

**ProcessTimer** — [source L1544](src/Driver/USB_Modem/Src/ClientState.cpp#L1544).

- Lệnh gửi/tạo: `AT+WMRXPOWER=1,%u,%u`, `AT+WMRXPOWER=0`.

- Các điều kiện kiểm tra: `m_stTestRequest.enable`.

#### `TEST_STATE_LISTENING_SET_FUN`

**ProcessRx** — [source L1362](src/Driver/USB_Modem/Src/ClientState.cpp#L1362).

- Response/URC và mã status được kiểm tra: `HL7690_OK`.

- Đích chuyển state/substate có trong handler: `TEST_STATE_LISTENING_WAIT`.

- Các điều kiện kiểm tra: `HL7690_OK == pMsg->id`.

**ProcessTx** — [source L1452](src/Driver/USB_Modem/Src/ClientState.cpp#L1452).

- Giữ state hiện tại hoặc chỉ làm bước chờ/thoát của handler; không phát AT hay chuyển state tại nhánh này.

**ProcessTimer** — [source L1558](src/Driver/USB_Modem/Src/ClientState.cpp#L1558).

- Lệnh gửi/tạo: `AT+CFUN=4`, `AT+CFUN=1`.

- Các điều kiện kiểm tra: `m_stTestRequest.enable`.

#### `TEST_STATE_LISTENING_RESET`

**ProcessRx** — [source L1370](src/Driver/USB_Modem/Src/ClientState.cpp#L1370).

- Giữ state hiện tại hoặc chỉ làm bước chờ/thoát của handler; không phát AT hay chuyển state tại nhánh này.

**ProcessTx** — [source L1453](src/Driver/USB_Modem/Src/ClientState.cpp#L1453).

- Giữ state hiện tại hoặc chỉ làm bước chờ/thoát của handler; không phát AT hay chuyển state tại nhánh này.

**ProcessTimer** — [source L1567](src/Driver/USB_Modem/Src/ClientState.cpp#L1567).

- Đích chuyển state/substate có trong handler: `TEST_STATE_LISTENING_WAIT`.

- Các điều kiện kiểm tra: `m_stTestRequest.enable`.

#### `TEST_STATE_LISTENING_GENERIC_COMMAND`

**ProcessRx** — [source L1374](src/Driver/USB_Modem/Src/ClientState.cpp#L1374).

- Response/URC và mã status được kiểm tra: `HL7690_CGSN`.

- Đích chuyển state/substate có trong handler: `TEST_STATE_LISTENING_WAIT`.

- Các điều kiện kiểm tra: `HL7690_CGSN == pMsg->id`.

**ProcessTx** — [source L1454](src/Driver/USB_Modem/Src/ClientState.cpp#L1454).

- Giữ state hiện tại hoặc chỉ làm bước chờ/thoát của handler; không phát AT hay chuyển state tại nhánh này.


## Tra cứu chi tiết từng substate Server theo handler

`SERVER_STATE_INIT_AT_CGATT` được khai báo ở header nhánh init khác nhưng không có `case` cùng tên trong `ServerState.cpp`. Do đó không mô tả một AT command được phát từ substate này chỉ dựa vào tên enum. Bảng dưới phản ánh các nhánh handler thực sự có trong source.

Phần này liệt kê riêng từng substate trong source, bao gồm nhánh OMAN và các nhánh sản phẩm khác. Đây là bản tra cứu **source chưa tiền xử lý**: các lệnh/đích khác nhau dưới `#if` là lựa chọn thay thế, không phải đều được chạy liên tiếp. Phần phân tích OMAN phía trên xác định đường áp dụng. Điều kiện, response và đích được liệt kê theo từng handler; dùng liên kết source để xem cặp điều kiện–hành động và phạm vi `switch` chính xác. Các handler dùng chung như kiểm tra timer ID/retry ở đầu hàm vẫn áp dụng dù không lặp lại ở từng hàng. `ProcessRxFromCmux` chỉ chuyển bytes vào parser, nên không lặp lại ở đây.

### C_ServerStateInit

#### `SERVER_STATE_INIT_WAIT_WAKEUP`

**ProcessRx** — [source L296](src/Driver/USB_Modem/Src/ServerState.cpp#L296).

- Đích chuyển state/substate có trong handler: `SERVER_STATE_INIT_ATE0`.

- Tác động/callback: `Update_modem_status`.

- Các điều kiện kiểm tra: `m_bFirstInitialize`, `(((u08Mode == CONNECT_WITHIN_WINDOW_AND_METHOD) \|\| (u08Mode == CONNECT_WITHIN_CALLING_WINDOW)) && C_ServerHost::GetInstance().IsCallingWindow()) \|\| (u08Mode == PERMANENTLY_CONNECT)`.

**ProcessTimer** — [source L723](src/Driver/USB_Modem/Src/ServerState.cpp#L723).

- Message phối hợp: `SHELL_HOST_MSG_RX_START_SERVER`.

- Tác động/callback: `Update_modem_status`.

- Các điều kiện kiểm tra: `m_bFirstInitialize == TRUE`, `((u08Mode == CONNECT_WITHIN_WINDOW_AND_METHOD) \|\| (u08Mode == CONNECT_WITHIN_CALLING_WINDOW)) && C_ServerHost::GetInstance().IsCallingWindow()`.

#### `SERVER_STATE_INIT_WAIT_STARTUP`

**ProcessRx** — [source L328](src/Driver/USB_Modem/Src/ServerState.cpp#L328).

- Giữ state hiện tại hoặc chỉ làm bước chờ/thoát của handler; không phát AT hay chuyển state tại nhánh này.

**ProcessTimer** — [source L750](src/Driver/USB_Modem/Src/ServerState.cpp#L750).

- Đích chuyển state/substate có trong handler: `SERVER_STATE_INIT_ATE0`.

- Tác động/callback: `Update_modem_status`, `Reply_get_modem_statistics`, `Get_modem_statistics`.

#### `SERVER_STATE_INIT_WAIT_PHY_RANDOMISATION`

**ProcessRx** — [source L330](src/Driver/USB_Modem/Src/ServerState.cpp#L330).

- Giữ state hiện tại hoặc chỉ làm bước chờ/thoát của handler; không phát AT hay chuyển state tại nhánh này.

**ProcessTimer** — [source L764](src/Driver/USB_Modem/Src/ServerState.cpp#L764).

- Đích chuyển state/substate có trong handler: `SERVER_STATE_INIT_ATE0`.

#### `SERVER_STATE_INIT_ATE0`

**ProcessRx** — [source L333](src/Driver/USB_Modem/Src/ServerState.cpp#L333).

- Response/URC và mã status được kiểm tra: `HL7690_OK`.

- Đích chuyển state/substate có trong handler: `SERVER_STATE_INIT_AT_CMEE`.

- Các điều kiện kiểm tra: `m_bWaitResponse`, `HL7690_OK == pMsg->id`.

**ProcessTimer** — [source L769](src/Driver/USB_Modem/Src/ServerState.cpp#L769).

- Lệnh gửi/tạo: `ATE0`.

#### `SERVER_STATE_INIT_AT_CMEE`

**ProcessRx** — [source L342](src/Driver/USB_Modem/Src/ServerState.cpp#L342).

- Response/URC và mã status được kiểm tra: `HL7690_OK`.

- Đích chuyển state/substate có trong handler: `SERVER_STATE_INIT_AT_KTCPCNX`, `SERVER_STATE_INIT_AT_CFUN4`, `SERVER_STATE_INIT_AT_CGDCONT`, `SERVER_STATE_INIT_AT_KCNXTIMER`, `SERVER_STATE_INIT_AT_KCNXCFG`, `SERVER_STATE_INIT_AT_K3`.

- Các điều kiện kiểm tra: `m_bWaitResponse`, `HL7690_OK == pMsg->id`, `C_ServerHost::GetInstance().IsForceReattach() \|\| C_ServerHost::GetInstance().IsApnChanged()`.

**ProcessTimer** — [source L773](src/Driver/USB_Modem/Src/ServerState.cpp#L773).

- Lệnh gửi/tạo: `AT+CMEE=1`.

#### `SERVER_STATE_INIT_AT_K3`

**ProcessRx** — [source L381](src/Driver/USB_Modem/Src/ServerState.cpp#L381).

- Response/URC và mã status được kiểm tra: `HL7690_OK`.

- Đích chuyển state/substate có trong handler: `SERVER_STATE_INIT_AT_CGDCONT`, `SERVER_STATE_INIT_AT_KTCPCNX`, `SERVER_STATE_INIT_AT_KCNXTIMER`, `SERVER_STATE_INIT_AT_KCNXCFG`.

- Các điều kiện kiểm tra: `m_bWaitResponse`, `HL7690_OK == pMsg->id`, `C_ServerHost::GetInstance().IsForceReattach() \|\| C_ServerHost::GetInstance().IsApnChanged()`.

**ProcessTimer** — [source L777](src/Driver/USB_Modem/Src/ServerState.cpp#L777).

- Lệnh gửi/tạo: `AT&K3`.

#### `SERVER_STATE_INIT_AT_CFUN4`

**ProcessRx** — [source L405](src/Driver/USB_Modem/Src/ServerState.cpp#L405).

- Response/URC và mã status được kiểm tra: `HL7690_OK`.

- Đích chuyển state/substate có trong handler: `SERVER_STATE_INIT_AT_WPPP`, `SERVER_STATE_INIT_AT_CFUN1`.

- Các điều kiện kiểm tra: `m_bWaitResponse`, `HL7690_OK == pMsg->id`.

**ProcessTimer** — [source L782](src/Driver/USB_Modem/Src/ServerState.cpp#L782).

- Lệnh gửi/tạo: `AT+CFUN=4`.

#### `SERVER_STATE_INIT_AT_WPPP`

**ProcessRx** — [source L418](src/Driver/USB_Modem/Src/ServerState.cpp#L418).

- Response/URC và mã status được kiểm tra: `HL7690_OK`.

- Đích chuyển state/substate có trong handler: `SERVER_STATE_INIT_AT_CFUN1`.

- Các điều kiện kiểm tra: `m_bWaitResponse`, `HL7690_OK == pMsg->id`.

**ProcessTimer** — [source L786](src/Driver/USB_Modem/Src/ServerState.cpp#L786).

- Lệnh gửi/tạo: `AT+WPPP=1,1,"%s","%s"`.

#### `SERVER_STATE_INIT_AT_CFUN1`

**ProcessRx** — [source L427](src/Driver/USB_Modem/Src/ServerState.cpp#L427).

- Response/URC và mã status được kiểm tra: `HL7690_OK`.

- Đích chuyển state/substate có trong handler: `SERVER_STATE_INIT_AT_CGDCONT`, `SERVER_STATE_INIT_AT_KCNXCFG`.

- Các điều kiện kiểm tra: `m_bWaitResponse`, `HL7690_OK == pMsg->id`, `C_ServerHost::GetInstance().IsForceReattach() \|\| C_ServerHost::GetInstance().IsApnChanged()`.

**ProcessTimer** — [source L792](src/Driver/USB_Modem/Src/ServerState.cpp#L792).

- Lệnh gửi/tạo: `AT+CFUN=1`.

#### `SERVER_STATE_INIT_AT_CGDCONT`

**ProcessRx** — [source L442](src/Driver/USB_Modem/Src/ServerState.cpp#L442).

- Response/URC và mã status được kiểm tra: `HL7690_OK`.

- Đích chuyển state/substate có trong handler: `SERVER_STATE_INIT_AT_KTCPCNX`, `SERVER_STATE_INIT_AT_KCNXTIMER`.

- Các điều kiện kiểm tra: `m_bWaitResponse`, `HL7690_OK == pMsg->id`.

**ProcessTimer** — [source L797](src/Driver/USB_Modem/Src/ServerState.cpp#L797).

- Lệnh gửi/tạo: `AT+CGDCONT=1,"IPV4V6","orange.m2m"`, `AT+CGDCONT=1,"IP","orange.m2m"`, `AT+CGDCONT=1,"IPV4V6","objconppublic"`, `AT+CGDCONT=1,"IP","objconppublic"`, `AT+CGDCONT=1,"IPV4V6","%s"`, `AT+CGDCONT=1,"IP","%s"`.

#### `SERVER_STATE_INIT_AT_KCNXCFG`

**ProcessRx** — [source L458](src/Driver/USB_Modem/Src/ServerState.cpp#L458).

- Response/URC và mã status được kiểm tra: `HL7690_OK`, `HL7690_CME_ERROR`.

- Đích chuyển state/substate có trong handler: `SERVER_STATE_INIT_AT_KTCPCNX`, `SERVER_STATE_INIT_AT_KCNXTIMER`, `SERVER_STATE_INIT_WAIT_APN_EMPTY`.

- Các điều kiện kiểm tra: `m_bWaitResponse`, `HL7690_OK == pMsg->id`, `HL7690_CME_ERROR == pMsg->id`, `(error == 917) && (pModemConfig->APN_Name[0] == 0)`.

**ProcessTimer** — [source L833](src/Driver/USB_Modem/Src/ServerState.cpp#L833).

- Lệnh gửi/tạo: `AT+KCNXCFG=1,"GPRS","orange.m2m",`, `AT+KCNXCFG=1,"GPRS","objconppublic","","",`, `AT+KCNXCFG=1,"GPRS","objconppublic","",""`, `AT+KCNXCFG=1,"GPRS","apn","%s","%s",`, `AT+KCNXCFG=1,"GPRS","apn","%s","%s"`, `AT+KCNXCFG=1,"GPRS","%s","%s","%s","IPV4V6"`, `AT+KCNXCFG=1,"GPRS","%s","%s","%s"`.

- Các điều kiện kiểm tra: `0 == strlen((char *)(pModemConfig->APN_Name))`.

#### `SERVER_STATE_INIT_WAIT_APN_EMPTY`

**ProcessRx** — [source L481](src/Driver/USB_Modem/Src/ServerState.cpp#L481).

- Giữ state hiện tại hoặc chỉ làm bước chờ/thoát của handler; không phát AT hay chuyển state tại nhánh này.

**ProcessTimer** — [source L887](src/Driver/USB_Modem/Src/ServerState.cpp#L887).

- Timer được chọn: `SERVER_STATE_TIMEOUT_NO_CONNECT`.

#### `SERVER_STATE_INIT_AT_KCNXTIMER`

**ProcessRx** — [source L485](src/Driver/USB_Modem/Src/ServerState.cpp#L485).

- Response/URC và mã status được kiểm tra: `HL7690_OK`.

- Đích chuyển state/substate có trong handler: `SERVER_STATE_INIT_AT_KTCPCFG`.

- Các điều kiện kiểm tra: `m_bWaitResponse`, `HL7690_OK == pMsg->id`.

**ProcessTimer** — [source L893](src/Driver/USB_Modem/Src/ServerState.cpp#L893).

- Lệnh gửi/tạo: `AT+KCNXTIMER=1,90,4,60,30`, `AT+KCNXTIMER=1,57,4,60`.

#### `SERVER_STATE_INIT_AT_KTCPCFG`

**ProcessRx** — [source L494](src/Driver/USB_Modem/Src/ServerState.cpp#L494).

- Response/URC và mã status được kiểm tra: `HL7690_KTCPCFG`.

- Đích chuyển state/substate có trong handler: `SERVER_STATE_INIT_AT_KTCPCNX`.

- Các điều kiện kiểm tra: `m_bWaitResponse`, `HL7690_KTCPCFG == pMsg->id`.

**ProcessTimer** — [source L906](src/Driver/USB_Modem/Src/ServerState.cpp#L906).

- Lệnh gửi/tạo: `AT+KTCPCFG=1,1,,%s`, `AT+KTCPCFG=1,1,,4059`.

#### `SERVER_STATE_INIT_AT_KTCPCNX`

**ProcessRx** — [source L505](src/Driver/USB_Modem/Src/ServerState.cpp#L505).

- Response/URC và mã status được kiểm tra: `HL7690_OK`, `HL7690_CME_ERROR`, `HL7690_KCNX_IND`, `HL7690_CNX_STATUS_STR`, `HL7690_CNX_STATUS_CONNECTING`, `HL7690_CNX_STATUS_CONNECTED`, `HL7690_CNX_STATUS_FAILED`, `QUEC_OPEN`, `HL7690_KTCP_IND`, `HL7690_KTCP_NOTIF`, `HL7690_TCP_STATUS_CLOSED`.

- Đích chuyển state/substate có trong handler: `SERVER_STATE_LISTENING`, `SERVER_STATE_UDP_LISTENING`.

- Tác động/callback: `Update_modem_status`, `ResetIsCloseServer`, `Reply_get_modem_statistics`, `Get_modem_statistics`, `Update_synchro_ping`, `Reply_get_LTE_connection_rejection`, `Get_LTE_connection_rejection`.

- Các điều kiện kiểm tra: `m_bWaitResponse`, `m_s32TcpServerSessionID == session`, `TCP_STATUS_READY == status`, `m_bFirstAttachment == TRUE`, `pModemConfig->ActiveP2PProtocol == E_TCP_SERVICE`, `QUECTEL_TCP_STATUS_CLOSED == status`, `HL7690_TCP_STATUS_CLOSED == status`.

**ProcessTimer** — [source L916](src/Driver/USB_Modem/Src/ServerState.cpp#L916).

- Lệnh gửi/tạo: `AT+QIOPEN=1,%i,"TCP `, `AT+QIOPEN=1,%i,"UDP `, `AT+KTCPCNX=%i`.

- Response/URC và mã status được kiểm tra: `QUEC_SERVER_CONNECT_ID`.

- Timer được chọn: `SERVER_STATE_TIMER_LONG_AT_COMMAND`.

- Các điều kiện kiểm tra: `pModemConfig->ActiveP2PProtocol == E_TCP_SERVICE`.

### C_ServerStateListening

#### `SERVER_STATE_LISTENING_WAIT`

**ProcessRx** — [source L987](src/Driver/USB_Modem/Src/ServerState.cpp#L987).

- Response/URC và mã status được kiểm tra: `HL7690_KCNX_IND`, `HL7690_CNX_STATUS_STR`, `HL7690_CNX_STATUS_DISCONNECTED`, `HL7690_CNX_STATUS_CLOSED`, `HL7690_KTCP_IND`, `HL7690_TCP_STATUS_CLOSED`, `HL7690_KTCP_SRVREQ`, `HL7690_KTCP_DATA`, `HL7690_KTCP_NOTIF`, `HL7690_TCP_NOTIF_STR`.

- Đích chuyển state/substate có trong handler: `SERVER_STATE_RESET`, `SERVER_STATE_ACCEPTED`, `SERVER_STATE_LISTENING_CONNECT`.

- Các điều kiện kiểm tra: `m_bWaitResponse`, `m_s32TcpServerSessionID == session`, `QUECTEL_TCP_STATUS_CLOSED == status`, `HL7690_TCP_STATUS_CLOSED == status`.

**ProcessTimer** — [source L1198](src/Driver/USB_Modem/Src/ServerState.cpp#L1198).

- Đích chuyển state/substate có trong handler: `SERVER_STATE_CLOSE`, `SERVER_STATE_RESET`.

- Tác động/callback: `ResetEndCallingWindow`, `ResetIsCloseServer`, `Update_modem_status`.

- Các điều kiện kiểm tra: `(C_ServerHost::GetInstance().GetMode() != PERMANENTLY_CONNECT)`, `C_ServerHost::GetInstance().IsEndCallingWindow()`, `(((C_ServerHost::GetInstance().GetMode() == CONNECT_METHOD_INVOKED) \|\| ((C_ServerHost::GetInstance().GetMode() == CONNECT_WITHIN_WINDOW_AND_METHOD) && !C_ServerHost::GetInstance() .IsCallingWindow())) && (m_u32NoConnectCounter >= GPRS_INVOKED_MODE_TIMEOUT) && (TRUE == m_bIsDisconnected)) \|\| C_ServerHost::GetInstance().IsCloseServer()`, `pModemConfig->Connection_Watchdog_Timer_P3 * 1000 < m_u32NoConnectCounter`.

#### `SERVER_STATE_LISTENING_CONNECT`

**ProcessRx** — [source L1089](src/Driver/USB_Modem/Src/ServerState.cpp#L1089).

- Response/URC và mã status được kiểm tra: `HL7690_CONNECT`, `HL7690_KTCP_NOTIF`, `HL7690_TCP_NOTIF_STR`, `HL7690_KTCP_DATA`, `HL7690_OK`, `HL7690_KCGPADDR`, `HL7690_CME_ERROR`.

- Đích chuyển state/substate có trong handler: `SERVER_STATE_ACCEPTED`, `SERVER_STATE_LISTENING_WAIT`.

- Các điều kiện kiểm tra: `m_bWaitResponse`, `m_s32TcpServerSubSessionID == subsession`.

**ProcessTimer** — [source L1256](src/Driver/USB_Modem/Src/ServerState.cpp#L1256).

- Lệnh gửi/tạo: `AT+KTCPSTART=%i`.

- Timer được chọn: `SERVER_STATE_TIMER_SHORT_AT_COMMAND`.

### C_ServerStateAccepted

#### `SERVER_STATE_ACCEPTED_WAIT`

**ProcessRx** — [source L1310](src/Driver/USB_Modem/Src/ServerState.cpp#L1310).

- Lệnh gửi/tạo: `AT+QIRD=%i,%i`, `AT+QICLOSE=%i`.

- Response/URC và mã status được kiểm tra: `HL7690_KCNX_IND`, `HL7690_CNX_STATUS_STR`, `HL7690_CNX_STATUS_DISCONNECTED`, `HL7690_CNX_STATUS_CLOSED`, `HL7690_KTCP_IND`, `HL7690_TCP_STATUS_CLOSED`, `HL7690_KTCP_SRVREQ`, `HL7690_KTCP_DATA`, `QUEC_CLOSE`, `QUEC_QIRD`, `HL7690_KTCP_NOTIF`, `HL7690_TCP_NOTIF_STR`, `HL7690_TCP_NOTIF_DISCONNECT`, `HL7690_EXIT_DATA`, `HL7690_DATA`, `HL7690_CONNECT`.

- Timer được chọn: `SERVER_STATE_TIMER_SHORT_AT_COMMAND`.

- Đích chuyển state/substate có trong handler: `SERVER_STATE_RESET`, `SERVER_STATE_LISTENING`, `SERVER_STATE_ACCEPTED_RCV`.

- Tác động/callback: `CloseTCP`, `USB_Recv`.

- Các điều kiện kiểm tra: `m_bWaitResponse`, `m_s32TcpServerSessionID == session`, `QUECTEL_TCP_STATUS_CLOSED == status`, `HL7690_TCP_STATUS_CLOSED == status`, `notif == HL7690_TCP_NOTIF_DISCONNECT`, `(C_ServerHost::GetInstance() .IsCallingWindow()) \|\| (C_ServerHost::GetInstance().GetMode() == PERMANENTLY_CONNECT) \|\| (C_ServerHost::GetInstance().GetMode() == CONNECT_METHOD_INVOKED) \|\| (C_ServerHost::GetInstance().GetMode() == CONNECT_WITHIN_WINDOW_AND_METHOD)`, `0 == memcmp(pMsg->rx.data.buf, "--EOF--Pattern--", 16)`, `0 == memcmp(pMsg->rx.data.buf, "\r\nOK\r\n", 6)`, `u16DataLen < (pMsg->rx.data.len)`, `u16RemainingLen <= pMsg->rx.data.len`.

**ProcessTimer** — [source L1692](src/Driver/USB_Modem/Src/ServerState.cpp#L1692).

- Timer được chọn: `SERVER_STATE_TIMER_FIVE_SECONDS`.

- Đích chuyển state/substate có trong handler: `SERVER_STATE_LISTENING`, `SERVER_STATE_ACCEPTED_CLOSE_SESSION`, `SERVER_STATE_SEND`.

- Message phối hợp: `SHELL_HOST_MSG_RX_CLOSE_TCP_SERVER_SESSION`.

- Tác động/callback: `Update_modem_status`.

- Các điều kiện kiểm tra: `m_u32NoRxCounter > 10000`, `(C_ServerHost::GetInstance().GetMode() == PERMANENTLY_CONNECT) \|\| (C_ServerHost::GetInstance().IsCallingWindow() && ((C_ServerHost::GetInstance().GetMode() == CONNECT_WITHIN_WINDOW_AND_METHOD) \|\| (C_ServerHost::GetInstance().GetMode() == CONNECT_WITHIN_CALLING_WINDOW)))`, `(pModemConfig->TCP_UDP_inactivityTimeOut) * 1000 < m_u32NoRxCounter`, `(m_BSendOk) && (m_u08SubState != SERVER_STATE_ACCEPTED_RCV)`.

#### `SERVER_STATE_ACCEPTED_RCV`

**ProcessRx** — [source L1501](src/Driver/USB_Modem/Src/ServerState.cpp#L1501).

- Lệnh gửi/tạo: `AT+QICLOSE=%i`.

- Response/URC và mã status được kiểm tra: `QUEC_CLOSE`, `HL7690_CME_ERROR`, `HL7690_ERROR`, `HL7690_EXIT_DATA`, `HL7690_DATA`.

- Timer được chọn: `SERVER_STATE_TIMER_SHORT_AT_COMMAND`.

- Đích chuyển state/substate có trong handler: `SERVER_STATE_LISTENING`, `SERVER_STATE_ACCEPTED_WAIT`.

- Tác động/callback: `CloseTCP`, `USB_Recv`.

- Các điều kiện kiểm tra: `0 == memcmp(pMsg->rx.data.buf, "\r\nOK\r\n", 6)`, `(u16DataLen <= (pMsg->rx.data.len)) && (u16DataLen == u16RemainingLen)`, `u16RemainingLen <= pMsg->rx.data.len`, `u16RemainingLen == 0`.

**ProcessTimer** — [source L1695](src/Driver/USB_Modem/Src/ServerState.cpp#L1695).

- Timer được chọn: `SERVER_STATE_TIMER_FIVE_SECONDS`.

- Đích chuyển state/substate có trong handler: `SERVER_STATE_LISTENING`, `SERVER_STATE_ACCEPTED_CLOSE_SESSION`, `SERVER_STATE_SEND`.

- Message phối hợp: `SHELL_HOST_MSG_RX_CLOSE_TCP_SERVER_SESSION`.

- Tác động/callback: `Update_modem_status`.

- Các điều kiện kiểm tra: `m_u32NoRxCounter > 10000`, `(C_ServerHost::GetInstance().GetMode() == PERMANENTLY_CONNECT) \|\| (C_ServerHost::GetInstance().IsCallingWindow() && ((C_ServerHost::GetInstance().GetMode() == CONNECT_WITHIN_WINDOW_AND_METHOD) \|\| (C_ServerHost::GetInstance().GetMode() == CONNECT_WITHIN_CALLING_WINDOW)))`, `(pModemConfig->TCP_UDP_inactivityTimeOut) * 1000 < m_u32NoRxCounter`, `(m_BSendOk) && (m_u08SubState != SERVER_STATE_ACCEPTED_RCV)`.

#### `SERVER_STATE_ACCEPTED_CLOSE_SESSION`

**ProcessRx** — [source L1566](src/Driver/USB_Modem/Src/ServerState.cpp#L1566).

- Lệnh gửi/tạo: `AT+QICLOSE=%i`.

- Response/URC và mã status được kiểm tra: `HL7690_KTCP_DATA`, `HL7690_KTCP_NOTIF`, `HL7690_TCP_NOTIF_STR`, `HL7690_TCP_NOTIF_DISCONNECT`, `HL7690_EXIT_DATA`, `QUEC_CLOSE`, `HL7690_DATA`.

- Timer được chọn: `SERVER_STATE_TIMER_SHORT_AT_COMMAND`.

- Đích chuyển state/substate có trong handler: `SERVER_STATE_LISTENING`.

- Tác động/callback: `CloseTCP`, `USB_Recv`.

- Các điều kiện kiểm tra: `notif == HL7690_TCP_NOTIF_DISCONNECT`, `C_ServerHost::GetInstance() .IsCallingWindow()`, `C_ServerHost::GetInstance().IsCallingWindow()`.

**ProcessTimer** — [source L1772](src/Driver/USB_Modem/Src/ServerState.cpp#L1772).

- Đích chuyển state/substate có trong handler: `SERVER_STATE_CLOSE`, `SERVER_STATE_RESET`, `SERVER_STATE_LISTENING`.

- Tác động/callback: `ResetEndCallingWindow`, `ResetIsCloseServer`, `CloseTCP`, `Update_modem_status`.

- Các điều kiện kiểm tra: `(m_bIsDisconnected && (C_ServerHost::GetInstance().IsCallingWindow() == FALSE)) \|\| (!m_bIsDisconnected && ((pModemConfig->TCP_UDP_inactivityTimeOut + 10) * 1000 < m_u32NoRxCounter))`, `(pModemConfig->TCP_UDP_inactivityTimeOut + 30) * 1000 < m_u32NoRxCounter`, `C_ServerHost::GetInstance().GetMode() != PERMANENTLY_CONNECT`.

### C_ServerStateSend

#### `SERVER_STATE_SEND_START`

**ProcessRx** — [source L1858](src/Driver/USB_Modem/Src/ServerState.cpp#L1858).

- Lệnh gửi/tạo: `AT+QICLOSE=%i`.

- Response/URC và mã status được kiểm tra: `HL7690_CONNECT`, `HL7690_KTCP_NOTIF`, `HL7690_TCP_NOTIF_STR`, `HL7690_TCP_NOTIF_DISCONNECT`, `HL7690_CME_ERROR`, `HL7690_ERROR`, `QUEC_SEND_OK`, `HL7690_EXIT_DATA`.

- Timer được chọn: `SERVER_STATE_TIMER_SHORT_AT_COMMAND`.

- Đích chuyển state/substate có trong handler: `SERVER_STATE_LISTENING`, `SERVER_STATE_ACCEPTED`.

- Message phối hợp: `SHELL_HOST_MSG_RX_GET_LTE_MODEM_STATISTICS`.

- Tác động/callback: `CloseTCP`, `Cmux_Ack_Gbt`.

- Các điều kiện kiểm tra: `notif == HL7690_TCP_NOTIF_DISCONNECT`.

**ProcessTimer** — [source L1972](src/Driver/USB_Modem/Src/ServerState.cpp#L1972).

- Lệnh gửi/tạo: `AT+QISEND=%i,%i`.

- Timer được chọn: `SERVER_STATE_TIMER_SHORT_AT_COMMAND`.

- Tác động/callback: `Update_modem_status`.

#### `SERVER_STATE_SEND_CLOSE_SESSION`

**ProcessTimer** — [source L1983](src/Driver/USB_Modem/Src/ServerState.cpp#L1983).

- Đích chuyển state/substate có trong handler: `SERVER_STATE_RESET`.

- Tác động/callback: `CloseTCP`, `Update_modem_status`.

- Các điều kiện kiểm tra: `(pModemConfig->TCP_UDP_inactivityTimeOut + 30) * 1000 < m_u32NoRxCounter`.

### C_ServerStateReset

#### `SERVER_STATE_RESET_WAIT`

**ProcessRx** — [source L2038](src/Driver/USB_Modem/Src/ServerState.cpp#L2038).

- Response/URC và mã status được kiểm tra: `HL7690_OK`, `QUEC_QPOWD`.

- Tác động/callback: `ResetSoft`.

- Các điều kiện kiểm tra: `m_bWaitResponse`, `HL7690_OK == pMsg->id`, `QUEC_QPOWD == pMsg->id`.

**ProcessTimer** — [source L2100](src/Driver/USB_Modem/Src/ServerState.cpp#L2100).

- Lệnh gửi/tạo: `AT+QPOWD`, `AT+CPWROFF`.

- Timer được chọn: `SERVER_STATE_TIMER_FOTA_UPGRADE`.

- Đích chuyển state/substate có trong handler: `SERVER_STATE_FOTA_WAIT`.

- Tác động/callback: `ResetSoft`.

- Các điều kiện kiểm tra: `C_ServerHost::GetInstance().GetFUInProgress() == TRUE`.

#### `SERVER_STATE_FOTA_WAIT`

**ProcessTimer** — [source L2125](src/Driver/USB_Modem/Src/ServerState.cpp#L2125).

- Lệnh gửi/tạo: `AT+QPOWD`, `AT+CPWROFF`.

- Timer được chọn: `SERVER_STATE_TIMER_FIVE_SECONDS`.

### C_ServerStateClose

#### `SERVER_STATE_CLOSE_WAIT`

**ProcessRx** — [source L2180](src/Driver/USB_Modem/Src/ServerState.cpp#L2180).

- Response/URC và mã status được kiểm tra: `HL7690_OK`.

- Đích chuyển state/substate có trong handler: `SERVER_STATE_CLOSE_DELETE`.

- Các điều kiện kiểm tra: `m_bWaitResponse`, `HL7690_OK == pMsg->id`.

**ProcessTimer** — [source L2255](src/Driver/USB_Modem/Src/ServerState.cpp#L2255).

- Lệnh gửi/tạo: `AT+QICLOSE=%i`, `AT+KTCPCLOSE=%i`.

- Response/URC và mã status được kiểm tra: `QUEC_SERVER_CONNECT_ID`.

- Timer được chọn: `SERVER_STATE_TIMER_LONG_AT_COMMAND`.

#### `SERVER_STATE_CLOSE_DELETE`

**ProcessRx** — [source L2190](src/Driver/USB_Modem/Src/ServerState.cpp#L2190).

- Response/URC và mã status được kiểm tra: `HL7690_OK`, `HL7690_CME_ERROR`.

- Message phối hợp: `SHELL_HOST_MSG_CLOSE_PDP_CONTEXT`.

- Các điều kiện kiểm tra: `m_bWaitResponse`, `HL7690_OK == pMsg->id`, `HL7690_CME_ERROR == pMsg->id`.

**ProcessTimer** — [source L2269](src/Driver/USB_Modem/Src/ServerState.cpp#L2269).

- Lệnh gửi/tạo: `AT+QICLOSE=%i`, `AT+KTCPDEL=%i`.

- Response/URC và mã status được kiểm tra: `QUEC_SERVER_CONNECT_ID`.

- Timer được chọn: `SERVER_STATE_TIMER_LONG_AT_COMMAND`.

### C_ServerStateUdpListening

#### `SERVER_STATE_UDP_LISTENING_WAIT`

**ProcessRx** — [source L2333](src/Driver/USB_Modem/Src/ServerState.cpp#L2333).

- Lệnh gửi/tạo: `AT+QIRD=%i,%i`.

- Response/URC và mã status được kiểm tra: `HL7690_KTCP_DATA`, `QUEC_QIRD_UDP`.

- Timer được chọn: `SERVER_STATE_TIMER_SHORT_AT_COMMAND`.

- Đích chuyển state/substate có trong handler: `SERVER_STATE_UDP_LISTENING_RCV`.

- Các điều kiện kiểm tra: `(2 < len) && (len < (LEN_IPV4_ADDR + 2))`, `(0 < len) && (len < (LEN_PORT))`.

**ProcessTimer** — [source L2469](src/Driver/USB_Modem/Src/ServerState.cpp#L2469).

- Đích chuyển state/substate có trong handler: `SERVER_STATE_UDP_LISTENING_SEND`.

- Các điều kiện kiểm tra: `(m_BSendOk) && (m_u08SubState != SERVER_STATE_UDP_LISTENING_RCV)`.

#### `SERVER_STATE_UDP_LISTENING_RCV`

**ProcessRx** — [source L2386](src/Driver/USB_Modem/Src/ServerState.cpp#L2386).

- Response/URC và mã status được kiểm tra: `HL7690_DATA`.

- Đích chuyển state/substate có trong handler: `SERVER_STATE_UDP_LISTENING_WAIT`.

- Tác động/callback: `USB_Recv`.

- Các điều kiện kiểm tra: `0 == memcmp(pMsg->rx.data.buf, "\r\nOK\r\n", 6)`, `(u16DataLen <= (pMsg->rx.data.len)) && (u16DataLen == u16RemainingLen)`, `u16RemainingLen <= pMsg->rx.data.len`, `u16RemainingLen == 0`.

**ProcessTimer** — [source L2471](src/Driver/USB_Modem/Src/ServerState.cpp#L2471).

- Đích chuyển state/substate có trong handler: `SERVER_STATE_UDP_LISTENING_SEND`.

- Các điều kiện kiểm tra: `(m_BSendOk) && (m_u08SubState != SERVER_STATE_UDP_LISTENING_RCV)`.

#### `SERVER_STATE_UDP_LISTENING_SEND`

**ProcessRx** — [source L2418](src/Driver/USB_Modem/Src/ServerState.cpp#L2418).

- Response/URC và mã status được kiểm tra: `HL7690_CONNECT`, `QUEC_SEND_OK`.

- Đích chuyển state/substate có trong handler: `SERVER_STATE_UDP_LISTENING_WAIT`.

**ProcessTimer** — [source L2480](src/Driver/USB_Modem/Src/ServerState.cpp#L2480).

- Lệnh gửi/tạo: `AT+QISEND=%i,%i,"%s",%s`.

- Timer được chọn: `SERVER_STATE_TIMER_SHORT_AT_COMMAND`.
