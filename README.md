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

