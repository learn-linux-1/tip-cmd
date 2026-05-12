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

== 1. Initialization Phase (Boot / Module Init) ==
Snapshot -> Snapshot : init_parallel_resume()
activate Snapshot
note right Snapshot: Register function pointers:\n- fn_ptr_invoke_instant_thread\n- fn_ptr_perform_priority_device_operation\n- fn_ptr_usb_kick_wq_list\n...
deactivate Snapshot

== 2. System Wakeup Phase (System Resume) ==
-> SysPM : System starts waking up (Wakeup Event)
activate SysPM
SysPM -> SysPM : dpm_resume() / dpm_resume_early()
note right SysPM: Iterate device list.\nWhen encountering a USB Device, trigger\ncode block #USB_PARALLEL_RESUME

alt If USB Hub (Initialize instant thread)
    SysPM -> Snapshot : Check & Call fn_ptr_invoke_instant_thread(hub)
    activate Snapshot
    Snapshot -> Snapshot : usb_resume_serdes_timeout() (If needed)
    Snapshot -> Hub : Initialize Instant Thread for Hub
    deactivate Snapshot
end

alt If Priority Device
    SysPM -> Snapshot : Call fn_ptr_perform_priority_device_operation(udev)
    activate Snapshot
    Snapshot -> Device : Resume immediately
    deactivate Snapshot
end

== 3. Parallel Workqueue Activation Phase ==
SysPM -> Snapshot : Call fn_ptr_usb_kick_wq_list() / wake_wq()
activate Snapshot
Snapshot -> Hub : Kick workqueues
deactivate Snapshot

Hub -> Device : Start parallel Reset/Resume process\nfor remaining USB devices
deactivate SysPM

== 4. Resume Process Completed ==
note over SysPM, Device: All USB devices have been resumed\nwithout blocking the main dpm_resume process.

@enduml




@startuml
skinparam maxMessageSize 200
skinparam ActivityBackgroundColor #LightCyan
skinparam ActivityBorderColor #006699

title Flowchart of USB Parallel Resume Process

start
:System Wakeup Event;
:System PM executes dpm_resume();

while (Iterate through all devices) is (More devices)
    if (Is it a USB Device?) then (Yes)
        if (USB_PARALLEL_RESUME enabled\n& pointers != NULL?) then (Yes)
            
            if (Is it a USB Hub?) then (Yes)
                :Call fn_ptr_invoke_instant_thread(hub);
                if (Kant.M Model / SERDES Check?) then (Yes)
                    :Execute usb_resume_serdes_timeout();
                else (No)
                endif
                :Initialize Instant Thread for Hub;
                
            elseif (Is it a Priority Device?) then (Yes)
                note right: e.g., BT, Zigbee IoT
                :Call fn_ptr_perform_priority_device_operation(udev);
                :Resume immediately (Synchronous);
                
            else (Normal USB Device)
                :Add to Workqueue list;
            endif
            
        else (No)
            :Standard USB Resume (Sequential);
        endif
    else (No)
        :Standard Device Resume;
    endif
endwhile (No more devices)

:Call fn_ptr_usb_kick_wq_list() / wake_wq();

fork
    :Main thread continues system resume;
fork again
    :Workqueue processes remaining\nUSB devices in parallel;
end fork

:Resume Process Completed;
stop

@enduml

Dưới đây là câu trả lời hoàn chỉnh bằng tiếng Anh (chuẩn văn phong kỹ sư phần mềm nhúng/kernel) để bạn gửi cho ông Lee.
Câu trả lời này đã giải thích rõ ràng **tại sao tính năng Async mặc định của Kernel không làm giảm tổng thời gian resume** (do vướng hàm đồng bộ async_synchronize_full), đồng thời lồng ghép giải pháp **"Tách luồng ngầm" (Deferred Resume)** mà chúng ta đã xây dựng để chứng minh bạn đã hiểu sâu vấn đề và có cách khắc phục triệt để.
### Bức thư gửi ông Lee (Tiếng Anh)
**Subject: RE: Questions regarding Async Resume and Snapshot Testing**
Hi Dongmin,
Thank you for your questions. Below are the detailed explanations regarding the USB async resume mechanism and the snapshot testing procedure.
**1. Regarding the Async Resume Mechanism in Linux Kernel (5.4+)**
**a & b. How Async Resume works and Code Location:**
In the Linux Kernel, the asynchronous resume mechanism is primarily handled in drivers/base/power/main.c.
When a device has dev->power.async_suspend = true (which is default for USB devices since kernel 3.2), the PM Core calls async_schedule() inside device_resume() to offload the resume callback to an asynchronous thread. This allows multiple devices to be resumed concurrently instead of sequentially.
**c. Why the total resume time did not decrease on Confluence (772.941 msecs):**
This is a very crucial point. Even though the USB resume is handled asynchronously, the PM Core (in dpm_resume()) has a strict synchronization barrier at the end of the resume phase. It calls the async_synchronize_full() function.
This function **blocks the main PM thread and waits for ALL async threads to finish** before it declares the system resume complete. Since the MediaTek Wi-Fi/BT hardware physically requires about ~450ms to reset and reload its firmware, the async_synchronize_full() barrier forces the entire system to wait for this 450ms. That is why the total PM log time does not drop.
*Note on our recent optimization:* To truly bypass this hardware bottleneck, we cannot rely on the native async_schedule. I have implemented a custom **Event-Driven Deferred Resume** patch in drivers/usb/core/driver.c. It intercepts the resume call, returns 0 immediately to the PM Core (bypassing async_synchronize_full), and safely resets the Wi-Fi/BT hardware in a completely detached background workqueue triggered by PM_POST_SUSPEND. This drops the PM wait time significantly.
**d. Measuring USB and Wi-Fi/BT Start/End Times:**
I will measure the exact start and end timestamps for the USB root hub and the specific Wi-Fi/BT module during the resume phase. I will compile these metrics and update the Confluence page shortly so we can review the exact hardware latency.
**2. Regarding the Test without Snapshot**
To ensure a clean test environment without the snapshot feature, I completely disabled it rather than just modifying code. Specifically, I removed/unloaded the snapshot .ko module (rmmod) and disabled its trigger service from the userspace/init scripts. This ensured that the system went through the standard Suspend-to-RAM (S3) flow naturally without any snapshot logic interfering with the baseline measurements.
Please let me know if you need further clarification on the code implementation or the logs.
Best regards,
[Tên của bạn]
### 💡 Lưu ý nhỏ cho bạn (Bùi Xuân Chinh):
 * Ở phần **1d**, tôi đã viết là bạn sẽ đo đạc và update lên Confluence sau. Bạn hãy nhớ dùng dmesg để lấy mốc thời gian (timestamp) lúc module WiFi bắt đầu resume và lúc nó in ra log kết thúc để bổ sung vào tài liệu nhé.
 * Ở phần **2**, tôi giả định cách bạn test bỏ snapshot là rmmod cái .ko ra và tắt service kích hoạt nó. Nếu thực tế bạn làm cách khác (ví dụ: build lại kernel và gỡ config snapshot), bạn có thể sửa lại đoạn *"I removed/unloaded the snapshot .ko module..."* thành *"I rebuilt the kernel with the snapshot configuration disabled..."*.
