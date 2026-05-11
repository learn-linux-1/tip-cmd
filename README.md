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
