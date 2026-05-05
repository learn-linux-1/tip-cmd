# Role & Expertise
You are an expert Embedded Systems & Linux Kernel Engineer. You possess deep knowledge of Linux kernel internals, device drivers (specifically USB subsystems), power management (suspend/resume), and single-board computer architectures (ARM/ARM64).

# Technical Stack
- **Languages:** C (Primary, Kernel Standard), Bash (Scripting/Automation), Python (Data parsing/Log analysis).
- **Kernel Versions:** Proficient in porting and writing code across kernel versions (focusing on 5.4 to 6.12).
- **Hardware:** Orange Pi, Raspberry Pi, Luckfox, custom kiosk boards.
- **Subsystems:** USB (Host, Gadget, Core), GPIO, I2C, SPI, PM (Power Management).

# Coding Standards & Guidelines
1. **Kernel Coding Style:** Strictly adhere to the Linux kernel coding style. Code must pass `checkpatch.pl` without warnings or errors.
2. **Memory Management:** Ensure zero memory leaks. Always check return values of memory allocation functions (`kmalloc`, `kzalloc`, etc.). Avoid using floating-point operations in kernel space.
3. **Concurrency:** Properly use spinlocks, mutexes, and atomic operations to prevent race conditions.
4. **Error Handling:** Use `goto` statements for clean error handling and resource cleanup at the end of functions. Return appropriate negative error codes (e.g., `-ENOMEM`, `-EINVAL`).
5. **Logging:** Use standard kernel logging macros (`pr_info`, `pr_err`, `pr_debug`, `dev_dbg`) with appropriate log levels. Ensure logs provide sufficient context for debugging hardware interactions.

# Process Rules
- Do not make assumptions about hardware states; always verify registers or hardware responses if possible.
- When writing out-of-tree Loadable Kernel Modules (LKMs), include a proper `Makefile` compatible with the target kernel build system.
- Before suggesting a complex refactor, propose a small proof-of-concept (PoC) script or patch.
- Always maintain backward compatibility or provide clear preprocessor directives (`#if LINUX_VERSION_CODE`) when porting functions (e.g., `usb_find_symbol`).




# Project Plan: [Tên Dự Án Của Bạn - VD: USB Resume LKM / USB Log Analyzer]

## 🎯 Objective
[Mô tả ngắn gọn mục tiêu dự án. Ví dụ: Xây dựng một Out-of-tree Loadable Kernel Module (LKM) để xử lý tính năng resume cho các thiết bị USB trên board ARM, tối ưu hóa điện năng tiêu thụ khi sleep.]

## 📊 Current Status
- [ ] Planning & Setup
- [ ] Implementation
- [ ] Testing & Debugging
- [ ] Documentation

## 📋 Task Breakdown

### Phase 1: Environment & Setup (To Do / In Progress / Done)
- [ ] Tạo cấu trúc thư mục module cơ bản (`main.c`, `Makefile`).
- [ ] Cấu hình Makefile để trỏ tới đúng source tree của Kernel đang dùng (VD: Kernel 5.4 / 6.12).
- [ ] Viết hàm `init_module` và `cleanup_module` cơ bản với log `pr_info`.
- [ ] Build thử nghiệm và load module (`insmod`, `rmmod`, `dmesg`).

### Phase 2: Core Implementation
- [ ] Hook/Đăng ký vào USB subsystem hoặc định nghĩa các callback function cần thiết.
- [ ] Triển khai logic quản lý năng lượng (xử lý suspend/resume events).
- [ ] Xử lý tính tương thích ngược cho các hàm kernel cụ thể (vd: wrapper cho các symbol thay đổi giữa kernel 5.4 và 6.x).
- [ ] Tích hợp logic xử lý GPIO (nếu cần trigger bằng phím cứng).

### Phase 3: Testing & Log Analysis
- [ ] Viết script Bash/Python hỗ trợ tự động trigger trạng thái sleep/wake của board.
- [ ] Bắt dmesg log, phân tích các điểm nghẽn hoặc lỗi STT/timeout khi USB resume.
- [ ] Tối ưu hóa memory usage và kiểm tra rò rỉ bộ nhớ.
- [ ] Chạy `checkpatch.pl` và format lại toàn bộ source code.

### Phase 4: Documentation
- [ ] Cập nhật file README.md với hướng dẫn build và cross-compile.
- [ ] Tài liệu hóa các tham số của module (Module parameters).

## 💡 Notes & Constraints
- Đảm bảo module không làm kernel panic (Oops) khi rút/cắm nóng thiết bị USB (hot-plugging) trong trạng thái suspend.
- Ưu tiên sử dụng dynamic debug để tránh spam log trên môi trường production.


/*
 * usb_parallel_resume.c
 * 
 * Skeleton for Out-of-tree USB Parallel Resume LKM.
 * Dispatches USB device resume operations asynchronously using workqueues.
 */

#include <linux/module.h>
#include <linux/kernel.h>
#include <linux/init.h>
#include <linux/usb.h>
#include <linux/workqueue.h>
#include <linux/slab.h>
#include <linux/version.h>

/* Thông tin Module */
MODULE_LICENSE("GPL");
MODULE_AUTHOR("Your Name");
MODULE_DESCRIPTION("USB Parallel Resume Out-of-tree Module");
MODULE_VERSION("1.0");

/* 
 * Định nghĩa một Workqueue riêng biệt cho các tác vụ resume song song.
 * Cờ WQ_UNBOUND cho phép các worker thread chạy trên bất kỳ CPU nào.
 */
static struct workqueue_struct *usb_resume_wq;

/* 
 * Cấu trúc context cho mỗi tác vụ resume.
 * Chứa con trỏ tới usb_device và struct work_struct để enqueue.
 */
struct usb_resume_work_ctx {
    struct work_struct work;
    struct usb_device *udev;
    // Thêm các biến trạng thái hoặc completion (nếu cần đợi đồng bộ sau này)
};

/*
 * Hàm thực thi cho mỗi work item.
 * Đây là nơi logic resume thực sự cho từng thiết bị được gọi.
 */
static void parallel_resume_work_func(struct work_struct *work)
{
    struct usb_resume_work_ctx *ctx = container_of(work, struct usb_resume_work_ctx, work);
    struct usb_device *udev = ctx->udev;

    if (!udev) {
        pr_err("usb_parallel_resume: udev is NULL in work_func\n");
        goto cleanup;
    }

    pr_info("usb_parallel_resume: Starting async resume for USB device [devnum: %d]\n", udev->devnum);

    /* 
     * TODO: Gọi API resume thiết bị USB tại đây.
     * Lưu ý sự khác biệt API giữa Kernel 5.4 và 6.12.
     */
#if LINUX_VERSION_CODE >= KERNEL_VERSION(6, 12, 0)
    // Logic/API cho kernel 6.12+
    // Ví dụ: usb_resume_device(udev, PMSG_RESUME); (Cần kiểm tra xem hàm này có được export không)
#else
    // Logic/API cho kernel 5.4
#endif

    pr_info("usb_parallel_resume: Finished async resume for USB device [devnum: %d]\n", udev->devnum);

cleanup:
    /* Giải phóng memory đã cấp phát cho context */
    kfree(ctx);
}

/*
 * Hàm helper để dispatch một tác vụ resume cho một thiết bị cụ thể.
 */
static int dispatch_usb_resume_async(struct usb_device *udev)
{
    struct usb_resume_work_ctx *ctx;

    if (!udev)
        return -EINVAL;

    ctx = kzalloc(sizeof(*ctx), GFP_KERNEL);
    if (!ctx) {
        pr_err("usb_parallel_resume: Failed to allocate memory for work context\n");
        return -ENOMEM;
    }

    ctx->udev = udev;
    INIT_WORK(&ctx->work, parallel_resume_work_func);

    /* Đưa tác vụ vào workqueue để thực thi song song */
    queue_work(usb_resume_wq, &ctx->work);
    
    return 0;
}

/*
 * Hàm Hook/Trigger. 
 * Bạn sẽ gọi hàm này (thông qua sysfs, ioctl, hoặc hook vào power management notifier)
 * để duyệt qua các thiết bị USB và kích hoạt quá trình resume song song.
 */
static int trigger_parallel_resume_all(void)
{
    pr_info("usb_parallel_resume: Triggering parallel resume for all target devices...\n");

    /* 
     * TODO: Duyệt qua bus/thiết bị USB. 
     * Nếu bạn dùng hàm như usb_find_symbol hoặc quét bus, hãy thực hiện ở đây.
     * Giả mã (Pseudocode):
     * for_each_usb_device(udev) {
     *     if (is_target_device(udev)) {
     *         dispatch_usb_resume_async(udev);
     *     }
     * }
     */

    return 0;
}

static int __init usb_parallel_resume_init(void)
{
    pr_info("usb_parallel_resume: Loading module...\n");

    /* Khởi tạo workqueue chuyên dụng */
    usb_resume_wq = alloc_workqueue("usb_parallel_resume_wq", WQ_UNBOUND | WQ_HIGHPRI, 0);
    if (!usb_resume_wq) {
        pr_err("usb_parallel_resume: Failed to create workqueue\n");
        return -ENOMEM;
    }

    /* 
     * TODO: Đăng ký các hook cần thiết.
     * Ví dụ: sysfs entry để user-space có thể trigger, 
     * hoặc đăng ký kprobe/notifier block để bắt sự kiện system resume.
     */

    pr_info("usb_parallel_resume: Module loaded successfully.\n");
    return 0;
}

static void __exit usb_parallel_resume_exit(void)
{
    pr_info("usb_parallel_resume: Unloading module...\n");

    /* Hủy workqueue. Hàm này sẽ block cho đến khi tất cả các work đang chạy hoàn thành */
    if (usb_resume_wq) {
        flush_workqueue(usb_resume_wq);
        destroy_workqueue(usb_resume_wq);
    }

    /* TODO: Gỡ bỏ các hook, kprobes, sysfs entries đã đăng ký */

    pr_info("usb_parallel_resume: Module unloaded.\n");
}

module_init(usb_parallel_resume_init);
module_exit(usb_parallel_resume_exit);



Dưới đây là một file Markdown (coding_guide.md) được thiết kế đặc biệt để bạn cung cấp cho Cline. File này đi sâu vào **logic nghiệp vụ, các ràng buộc kỹ thuật và cách thức implement code chi tiết** thay vì chỉ hướng dẫn build thông thường.
Nó tập trung mạnh vào việc xử lý đồng thời (concurrency) và tính tương thích trên các phiên bản kernel khác nhau, điều rất quan trọng khi porting code trên các hệ thống nhúng thực tế.
```markdown
# Hướng dẫn Yêu cầu và Triển khai Code: USB Parallel Resume LKM

Tài liệu này xác định các yêu cầu kỹ thuật cốt lõi và hướng dẫn cách viết mã cho module `usb_parallel_resume`. AI Assistant (Cline) cần đọc kỹ các yêu cầu này trước khi bắt đầu implement hoặc refactor code.

## 1. Yêu cầu Kiến trúc Tổng thể
- **Loại hình:** Out-of-tree Loadable Kernel Module (LKM).
- **Mục tiêu:** Giảm độ trễ khi hệ thống thức dậy (wake up) bằng cách gửi tín hiệu resume đến nhiều thiết bị USB cùng lúc thay vì xử lý tuần tự. Đặc biệt hữu ích để tiết kiệm năng lượng và tăng tốc độ phản hồi trên các bo mạch nhúng như Orange Pi, Raspberry Pi hay Luckfox.
- **Kỹ thuật Concurrency:** Bắt buộc sử dụng Workqueue với cờ `WQ_UNBOUND` để đẩy các tác vụ resume sang các CPU core khác nhau xử lý song song.
- **Tính tương thích:** Code phải có các macro tiền xử lý (`#if LINUX_VERSION_CODE`) để tương thích chéo giữa các phiên bản Kernel cũ (như 5.4) và mới (như 6.12).

## 2. Chi tiết Triển khai (How to write code)

### 2.1. Quản lý Workqueue (Thiết lập môi trường song song)
- **Yêu cầu:** Không sử dụng global system workqueue. Phải tạo một workqueue riêng biệt để dễ dàng kiểm soát và flush khi unload module.
- **Cách viết:**
  ```c
  // Cấp phát trong init_module
  // WQ_UNBOUND: Cho phép kernel tự do xếp lịch worker thread trên bất kỳ CPU nào.
  // WQ_HIGHPRI: Ưu tiên cao để giảm độ trễ resume.
  usb_resume_wq = alloc_workqueue("usb_resume_wq", WQ_UNBOUND | WQ_HIGHPRI, 0);
  
  // Hủy trong exit_module
  flush_workqueue(usb_resume_wq); // Bắt buộc đợi các tác vụ hoàn thành
  destroy_workqueue(usb_resume_wq);
  

```
### 2.2. Tìm kiếm và Duyệt thiết bị USB (Device Matching)
 * **Yêu cầu:** Không được can thiệp vào các root hub mà chỉ target các thiết bị ngoại vi cụ thể hoặc duyệt qua toàn bộ bus.
 * **Cách viết:** Sử dụng API usb_for_each_dev an toàn để duyệt.
   ```c
   static int match_and_dispatch_device(struct usb_device *udev, void *data) {
       // Bỏ qua root hub
       if (!udev->parent) return 0;
   
       // Có thể thêm logic filter theo Vendor ID / Product ID tại đây
       // if (le16_to_cpu(udev->descriptor.idVendor) == TARGET_VID) { ... }
   
       dispatch_usb_resume_async(udev);
       return 0; // Trả về 0 để tiếp tục duyệt các thiết bị khác
   }
   
   // Chạy lệnh quét
   usb_for_each_dev(NULL, match_and_dispatch_device);
   
   
   ```
```

### 2.3. Logic Resume Bất đồng bộ & Tính tương thích Kernel (Porting)
- **Yêu cầu:** Đây là phần quan trọng nhất. Context của work item phải chứa con trỏ `usb_device`. Phải quản lý reference count của USB device để tránh Kernel Oops nếu thiết bị bị rút ra (hot-unplug) trong lúc đang resume.
- **Cách viết:**
  ```c
  // Khi dispatch:
  usb_get_dev(udev); // Tăng reference count trước khi đưa vào workqueue
  
  // Trong hàm worker thread (parallel_resume_work_func):
  usb_lock_device(udev); // Lock thiết bị để đảm bảo an toàn trạng thái
  
  #if LINUX_VERSION_CODE >= KERNEL_VERSION(6, 12, 0)
      // Sử dụng API của kernel 6.x
      // Cần kiểm tra xem usb_resume_device có được EXPORT_SYMBOL_GPL không.
      // Nếu không, cần tìm symbol động (kallsyms_lookup_name).
      usb_resume_device(udev, PMSG_RESUME); 
  #elif LINUX_VERSION_CODE >= KERNEL_VERSION(5, 4, 0)
      // Sử dụng API / logic tương đương của kernel 5.4
      // Trong 5.4, quá trình runtime PM thường sử dụng:
      pm_runtime_get_sync(&udev->dev);
      // ... thực hiện thao tác ...
      pm_runtime_put_autosuspend(&udev->dev);
  #else
      #error "Unsupported kernel version"
  #endif

  usb_unlock_device(udev);
  usb_put_dev(udev); // Giảm reference count khi xong việc
  

```
### 2.4. Cơ chế Kích hoạt (Trigger Mechanism)
 * **Yêu cầu:** Khởi tạo quá trình này tự động khi hệ thống wake up từ trạng thái suspend to RAM.
 * **Cách viết:** Đăng ký pm_notifier. Cần cực kỳ cẩn thận với ngữ cảnh (context) khi hàm callback này được gọi, tuyệt đối không được gọi hàm sleep (msleep, v.v.) bên trong callback này.
   ```c
   static int pm_notify_callback(struct notifier_block *nb, unsigned long action, void *ptr) {
       if (action == PM_POST_SUSPEND) {
           // Lên lịch duyệt và dispatch ngay lập tức
           trigger_parallel_resume_all();
       }
       return NOTIFY_OK;
   }
   
   
   ```
```

## 3. Tiêu chuẩn Mã nguồn & An toàn Memory
1.  **Memory Allocation:** Khi cấp phát memory cho context struct (`usb_resume_work_ctx`), phải kiểm tra kỹ context. Nếu cấp phát bên trong hàm ngắt (interrupt) hoặc spinlock, BẮT BUỘC dùng `GFP_ATOMIC`. Nếu cấp phát ở ngữ cảnh process thông thường, dùng `GFP_KERNEL`.
2.  **Concurrency Safety:** Không gọi trực tiếp các API USB có khả năng block/sleep (như gửi control message đồng bộ) nếu chưa ở trong context của workqueue.
3.  **Logging:** Sử dụng `dev_dbg(&udev->dev, ...)` để log thông tin riêng biệt cho từng thiết bị thay vì dùng `pr_info` chung chung, giúp việc phân tích log dễ dàng hơn khi có nhiều luồng chạy song song.

```
Với tài liệu này, bạn định sẽ phân biệt các thiết bị USB cần resume bằng cách nào (ví dụ: quét toàn bộ các thiết bị ngoại vi trên bus, hay chỉ lọc ra các Vendor ID / Product ID cụ thể để tối ưu thêm)?
```

```

