# Kiến trúc thiết kế AK-EDP (Tổng quan triển khai)

Tài liệu này tóm tắt các thiết kế đã được hiện thực trong mã nguồn AK-EDP (không phải CIEDPC). Nội dung tập trung vào:
- Quản lý bộ nhớ tin nhắn
- Quản lý ISR / cơ chế ngắt giao tiếp với AK-EDP
- Truyền dữ liệu giữa hai task (chuỗi, struct phức, dữ liệu bình thường)
- TSM (Task State Machine)
- FSM (Finite State Machine)
- Thiết kế phân định giá trị dãy tín hiệu

**Tài liệu tham khảo mã nguồn chính**
- Quản lý tin nhắn: [core message.h](AK%20Firmware%20F103%20Custom%20Porting/Drivers/EDP/core/inc/message.h#L1-L40), [core message.c](AK%20Firmware%20F103%20Custom%20Porting/Drivers/EDP/core/src/message.c#L1-L120)
- Task / scheduler: [core task.h](AK%20Firmware%20F103%20Custom%20Porting/Drivers/EDP/core/inc/task.h#L1-L40), [core task.c](AK%20Firmware%20F103%20Custom%20Porting/Drivers/EDP/core/src/task.c#L1-L120)
- FSM / TSM: [core fsm.h](AK%20Firmware%20F103%20Custom%20Porting/Drivers/EDP/core/inc/fsm.h#L1-L40), [core fsm.c](AK%20Firmware%20F103%20Custom%20Porting/Drivers/EDP/core/src/fsm.c#L1-L40), [core tsm.h](AK%20Firmware%20F103%20Custom%20Porting/Drivers/EDP/core/inc/tsm.h#L1-L40), [core tsm.c](AK%20Firmware%20F103%20Custom%20Porting/Drivers/EDP/core/src/tsm.c#L1-L80)
- Giao diện tác vụ / forward: [app task_if.cpp](AK%20Firmware%20F103%20Custom%20Porting/Drivers/EDP/app/task_if.cpp#L1-L80), [app app.cpp](AK%20Firmware%20F103%20Custom%20Porting/Drivers/EDP/app/app.cpp#L1-L120)
- Port / NVIC / critical: [sys platform.h](AK%20Firmware%20F103%20Custom%20Porting/Drivers/EDP/sys/platform.h#L1-L40)
- Container hỗ trợ: [fifo.h](AK%20Firmware%20F103%20Custom%20Porting/Drivers/EDP/common/container/fifo.h#L1-L40), [ring_buffer.h](AK%20Firmware%20F103%20Custom%20Porting/Drivers/EDP/common/container/ring_buffer.h#L1-L40), [log_queue.h](AK%20Firmware%20F103%20Custom%20Porting/Drivers/EDP/common/container/log_queue.h#L1-L40)

---

**1) Quản lý bộ nhớ tin nhắn**

Thiết kế tổng quan:
- Có ba loại tin nhắn: Pure (header only), Common (header + fixed-size data), Dynamic (header + heap data). Định nghĩa và macro: xem [message.h](AK%20Firmware%20F103%20Custom%20Porting/Drivers/EDP/core/inc/message.h#L1-L120).
- Ba pool tĩnh được khai báo trong `message.c`: `msg_pure_pool[]`, `msg_common_pool[]`, `msg_dynamic_pool[]` (kích thước qua defines trong `ak.cfg.mk`).
- Mỗi tin nhắn có `ref_count` và một số bit dùng làm type mask (`AK_MSG_TYPE_MASK`) + ref-count mask (`AK_MSG_REF_COUNT_MASK`). Ref-count tối đa giới hạn bằng `AK_MSG_REF_COUNT_MAX`.

Hoạt động:
- Lấy tin nhắn: `get_pure_msg()`, `get_common_msg()`, `get_dynamic_msg()` — tất cả trả về con trỏ vào pool và set metadata cơ bản. (xem [message.c](AK%20Firmware%20F103%20Custom%20Porting/Drivers/EDP/core/src/message.c#L1-L120)).
- Gán dữ liệu:
  - Common: `set_data_common_msg(msg, data, len)` — copy vào internal buffer cố định.
  - Dynamic: `set_data_dynamic_msg(msg, data, size)` — cấp phát bằng `ak_malloc()` và copy; freed khi `msg` trả về pool (`free_dynamic_msg` gọi `ak_free`).
- Giải phóng: `msg_dec_ref_count()` và `msg_free()` — giải phóng thật khi ref_count về 0; có `msg_force_free()` để cưỡng bức.

Thiết kế này cho phép:
- Truyền dữ liệu nhỏ hiệu quả bằng copy (common)
- Truyền dữ liệu lớn bằng cấp phát heap (dynamic)
- Chia sẻ tin nhắn giữa nhiều consumer bằng ref-count (không copy thêm)

**2) Quản lý ISR / ngắt giao tiếp với AK-EDP**

Các nguyên tắc được implement:
- Có cơ chế critical section wrapper: `platform_enter_critical()` / `platform_exit_critical()` và macro `entry_critical()` / `exit_critical()` (xem [platform.h](AK%20Firmware%20F103%20Custom%20Porting/Drivers/EDP/sys/platform.h#L1-L40)) — dùng để bảo vệ thao tác trên pool/hàng đợi trong ngữ cảnh ngắt.
- Thiết kế khuyến nghị: ISR thu dữ liệu -> tạo/giải mã thành `ak_msg_*_if_t` (interface message) -> đặt trường `if_*` tương ứng -> `task_post()` hoặc forward qua `task_post_*_msg()` từ context phù hợp. Trong code ví dụ, forwarding interface được xử lý trong `task_if` (xem [task_if.cpp](AK%20Firmware%20F103%20Custom%20Porting/Drivers/EDP/app/task_if.cpp#L1-L80)) bằng cách tăng ref-count (`msg_inc_ref_count`) và post tới task đích.
- `sys_ctrl` / `sys_cfg` chứa helpers về quản lý tick/clock/soft-watchdog; ISR có thể dùng `entry_critical()` để ngắt danh sách chung (xem [porting_compat.cpp / sys_ctrl](AK%20Firmware%20F103%20Custom%20Porting/Drivers/EDP/sys/porting_compat.cpp#L1-L80)).

Ghi chú thực tiễn:
- Không đặt logic nặng trong ISR — ISR nên tạo/điền message rồi post/forward vào task để xử lý (design của `task_if` minh họa pattern này).
- `log_queue` có thể bật để ghi log trong ngắt (có macro `AK_IRQ_OBJ_LOG_ENABLE`) — nhưng vẫn cần hạn chế thời gian ISR.

**3) Truyền dữ liệu giữa 2 task (char strings, struct phức tạp, dữ liệu bình thường)**

Các phương thức chính:
- `task_post_pure_msg(des, sig)` — gửi signal thuần túy (no payload).
- `task_post_common_msg(des, sig, data, len)` — gửi payload nhỏ; hàm copy dữ liệu vào vùng `data[]` kích thước cố định (AK_COMMON_MSG_DATA_SIZE). (xem [task.h](AK%20Firmware%20F103%20Custom%20Porting/Drivers/EDP/core/inc/task.h#L1-L80) và [task.c](AK%20Firmware%20F103%20Custom%20Porting/Drivers/EDP/core/src/task.c#L1-L160)).
- `task_post_dynamic_msg(des, sig, data, len)` — gửi payload lớn; hàm cấp phát bộ nhớ trong pool dynamic và copy dữ liệu. Receiver dùng `get_data_dynamic_msg()/get_data_len_dynamic_msg()`.
- Forwarding / sharing: khi muốn gửi cùng 1 `ak_msg_t*` đến nhiều task hoặc đi qua interface, code tăng `ref_count` bằng `msg_inc_ref_count()` trước khi `task_post()` (xem `task_if.cpp`). Khi xử lý xong, hệ thống gọi `msg_free()` để giảm ref-count.

Kết luận: hệ thống cung cấp 3 chiến lược truyền dữ liệu — không copy (pure header + refcount), copy nhanh cho payload nhỏ (common), và heap-allocated cho dữ liệu lớn (dynamic). Lựa chọn phụ thuộc vào kích thước payload và yêu cầu hiệu năng.

**4) TSM — Task State Machine**

Thiết kế & implement:
- TSM được định nghĩa bằng `tsm_tbl_t` chứa `state`, `on_state` callback và `table` là mảng con trỏ `tsm_t*` (xem [tsm.h](AK%20Firmware%20F103%20Custom%20Porting/Drivers/EDP/core/inc/tsm.h#L1-L80)).
- Mỗi `tsm_t` chứa `sig`, `next_state`, `tsm_func` (xử lý cho tín hiệu trong state đó).
- `tsm_dispatch()` tìm entry tương ứng với `msg->sig`, gọi `tsm_func` và nếu cần chuyển state sẽ gọi `tsm_tran()`; `tsm_tran()` gọi `on_state` khi state thay đổi (xem [tsm.c](AK%20Firmware%20F103%20Custom%20Porting/Drivers/EDP/core/src/tsm.c#L1-L80)).

Usage:
- TSM phù hợp để quản lý trạng thái cho một đối tượng/actor nội bộ của task, nơi transition phụ thuộc tín hiệu (msg->sig) và có callback on-state.

**5) FSM — Finite State Machine**

Thiết kế & implement:
- FSM đơn giản: `fsm_t` chỉ giữ con trỏ `state` kiểu `state_handler` (function pointer). Macro `FSM(me, init)` đặt state ban đầu; `fsm_dispatch(me, msg)` simply calls `me->state(msg)` (xem [fsm.h](AK%20Firmware%20F103%20Custom%20Porting/Drivers/EDP/core/inc/fsm.h#L1-L40) và [fsm.c](AK%20Firmware%20F103%20Custom%20Porting/Drivers/EDP/core/src/fsm.c#L1-L40)).

Usage:
- FSM hướng tới trường hợp nơi mỗi state là một hàm nhận `ak_msg_t*` và có thể chuyển state bằng cách thay `me->state` (macro `FSM_TRAN`). Đây là pattern nhẹ và trực tiếp.

**6) Thiết kế phân định giá trị dãy tín hiệu**

Cách tổ chức tín hiệu / signal:
- `sig` là `uint8_t` trong `ak_msg_t` (xem [message.h](AK%20Firmware%20F103%20Custom%20Porting/Drivers/EDP/core/inc/message.h#L1-L40)).
- Các giá trị tín hiệu do ứng dụng định nghĩa (ví dụ `APP_EDP_DBG_SIG_START`) và có phạm vi riêng (có macro như `AK_USER_DEFINE_SIG` trong `ak.h`).
- TSM sử dụng `sig` để index tìm handler trong bảng trạng thái; FSM handler cũng dùng `msg->sig` để quyết định hành vi.
- Ngoài ra các bitmask/flags (ví dụ `AK_MSG_TYPE_MASK`) phân định loại tin nhắn — tách rời trách nhiệm giữa `type` (pure/common/dynamic) và `sig` (ý nghĩa nghiệp vụ).

Task scheduling priority & ready-mask:
- `task_ready` là bitmask; macro `LOG2LKUP()` trong `platform.h` dùng để tìm index bit priority cao nhất set (xem [platform.h](AK%20Firmware%20F103%20Custom%20Porting/Drivers/EDP/sys/platform.h#L1-L40)). `task_sheduler()` sử dụng điều này để chọn task ưu tiên cao nhất đang ready (xem [task.c scheduler](AK%20Firmware%20F103%20Custom%20Porting/Drivers/EDP/core/src/task.c#L1-L120)).

---

**Tóm tắt nhanh — Khi cần thực hiện**
- Gửi dữ liệu nhỏ: gọi `task_post_common_msg()` → receiver `get_data_common_msg()`.
- Gửi dữ liệu lớn: gọi `task_post_dynamic_msg()` → receiver `get_data_dynamic_msg()` → sau xử lý `msg_free()`.
- Forward trong ISR/interface: tạo message interface (`ak_msg_*_if_t`), set `if_*` fields, tăng ref-count (`msg_inc_ref_count`) nếu forward đến nhiều task, sau đó `task_post()`.
- State machine nội bộ: dùng TSM để quản lý transitions theo tín hiệu; dùng FSM khi mỗi state là một handler riêng.

**Vị trí tham khảo trong mã (chính yếu)**
- Message core: [core/inc/message.h](AK%20Firmware%20F103%20Custom%20Porting/Drivers/EDP/core/inc/message.h#L1-L40), [core/src/message.c](AK%20Firmware%20F103%20Custom%20Porting/Drivers/EDP/core/src/message.c#L1-L200)
- Task / scheduler: [core/inc/task.h](AK%20Firmware%20F103%20Custom%20Porting/Drivers/EDP/core/inc/task.h#L1-L40), [core/src/task.c](AK%20Firmware%20F103%20Custom%20Porting/Drivers/EDP/core/src/task.c#L1-L200)
- FSM / TSM: [core/inc/fsm.h](AK%20Firmware%20F103%20Custom%20Porting/Drivers/EDP/core/inc/fsm.h#L1-L40), [core/inc/tsm.h](AK%20Firmware%20F103%20Custom%20Porting/Drivers/EDP/core/inc/tsm.h#L1-L80)
- Interface forwarding: [app/task_if.cpp](AK%20Firmware%20F103%20Custom%20Porting/Drivers/EDP/app/task_if.cpp#L1-L80)
- Critical / NVIC helper: [sys/platform.h](AK%20Firmware%20F103%20Custom%20Porting/Drivers/EDP/sys/platform.h#L1-L40)
- Containers: [common/container/ring_buffer.h](AK%20Firmware%20F103%20Custom%20Porting/Drivers/EDP/common/container/ring_buffer.h#L1-L40), [fifo.h](AK%20Firmware%20F103%20Custom%20Porting/Drivers/EDP/common/container/fifo.h#L1-L40), [log_queue.h](AK%20Firmware%20F103%20Custom%20Porting/Drivers/EDP/common/container/log_queue.h#L1-L40)

---

Muốn tôi mở rộng chi tiết vào phần nào (ví dụ minh họa luồng dữ liệu cụ thể, hoặc bản vẽ sequence cho ISR→task→TSM), bạn chọn phần để tôi bổ sung tiếp nhé.