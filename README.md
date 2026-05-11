@startuml
skinparam maxMessageSize 200

box "Kernel Core" #LightBlue
participant "System PM\n(base/power/main.c)" as SysPM
end box

box "USB Subsystem" #LightGreen
participant "USB Snapshot\n(snapshot.c)" as Snapshot
participant "USB Core / Hub WQ\n(Workqueue)" as Hub
participant "USB Device\n(BT, Zigbee IoT, ...)" as Device
end box

== 1. Giai đoạn Khởi tạo (Boot / Module Init) ==
Snapshot -> Snapshot : init_parallel_resume()
activate Snapshot
note right Snapshot: Đăng ký các con trỏ hàm:\n- fn_ptr_invoke_instant_thread\n- fn_ptr_perform_priority_device_operation\n- fn_ptr_usb_kick_wq_list\n...
deactivate Snapshot

== 2. Giai đoạn Đánh thức Hệ thống (System Resume) ==
-> SysPM : Hệ thống bắt đầu thức dậy (Wakeup Event)
activate SysPM
SysPM -> SysPM : dpm_resume() / dpm_resume_early()
note right SysPM: Duyệt danh sách thiết bị.\nKhi gặp USB Device, kích hoạt\nblock code #USB_PARALLEL_RESUME

alt Nếu là USB Hub (Khởi tạo luồng xử lý nhanh)
    SysPM -> Snapshot : Kiểm tra & Gọi fn_ptr_invoke_instant_thread(hub)
    activate Snapshot
    Snapshot -> Snapshot : usb_resume_serdes_timeout() (Nếu cần)
    Snapshot -> Hub : Khởi tạo luồng (Instant Thread) cho Hub
    deactivate Snapshot
end

alt Nếu là Thiết bị ưu tiên (Priority Devices)
    SysPM -> Snapshot : Gọi fn_ptr_perform_priority_device_operation(udev)
    activate Snapshot
    Snapshot -> Device : Khôi phục trạng thái (Resume) ngay lập tức
    deactivate Snapshot
end

== 3. Giai đoạn Kích hoạt Workqueue Song song ==
SysPM -> Snapshot : Gọi fn_ptr_usb_kick_wq_list() / wake_wq()
activate Snapshot
Snapshot -> Hub : Kích hoạt các hàng đợi công việc (Workqueue)
deactivate Snapshot

Hub -> Device : Bắt đầu quá trình Reset/Resume song song\ncho các thiết bị USB còn lại
deactivate SysPM

== 4. Hoàn tất quá trình Resume ==
note over SysPM, Device: Toàn bộ thiết bị USB đã được khôi phục\nmà không làm chặn (block) tiến trình dpm_resume chính.

@enduml

