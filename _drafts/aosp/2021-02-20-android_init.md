---
layout:     post
title:      "Android系统源码-init进程启动"
date:       2021-02-21
author:     "phantomVK"
header-img: "img/bg/post_bg.jpg"
catalog:    true
tags:
    - AOSP
---

> 源码基于Android-11.0.0_r3分支进行解读。最新系统和旧版差异较大，且由于本人水平有限，编写文章或有错漏之处，请与文章后的评论区直接指出，感谢支持。

## 一、启动流程

**init进程**，是Linux内核启动完成后，在用户空间开始的第一个进程，其进程号固定为1，而其工作职责有：

- 挂载系统文件目录；
- 解析并加载init.rc文件，生成相应的设备驱动节点；
- 处理子进程的终止；
- 提供属性服务能力；

### 1.1 init入口

init进程启动的入口是 **system/core/init/main.cpp** 的 **int main(int argc, char\*\* argv)** 函数。

**main()** 函数非常简单，主要包含以下调用：

1. **FirstStageMain**：启动先执行此函数，负责第一阶段初始化工作；
2. **SetupSelinux**：第一阶段初始化即将结束时，会通过Selinux设置访问控制安全策略；
3. **SecondStageMain**：第二阶段初始化工作，由SetupSelinux结束时调用；

```c++
// system/core/init/main.cpp
int main(int argc, char** argv) {
#if __has_feature(address_sanitizer)
    __asan_set_error_report_callback(AsanReportCallback);
#endif

    if (!strcmp(basename(argv[0]), "ueventd")) {
        return ueventd_main(argc, argv);
    }

    // 根据argc第二个参数的内容决定执行的逻辑分支
    if (argc > 1) {
        if (!strcmp(argv[1], "subcontext")) {
            android::base::InitLogging(argv, &android::base::KernelLogger);
            const BuiltinFunctionMap& function_map = GetBuiltinFunctionMap();

            return SubcontextMain(argc, argv, &function_map);
        }

        // 第一阶段即将结束时开始selinux_setup，执行完毕再进入第二阶段初始化
        if (!strcmp(argv[1], "selinux_setup")) {
            return SetupSelinux(argv);
        }

        // 执行第二阶段初始化
        if (!strcmp(argv[1], "second_stage")) {
            return SecondStageMain(argc, argv);
        }
    }

    // 执行第一阶段初始化
    return FirstStageMain(argc, argv);
}
```



### 1.2 FirstStageMain

第一阶段初始化主要完成系统路径的挂载，在结束初始化函数前拉起 **selinux_setup**，把初始化工作交给 **selinux_setup** 继续进行。

```c++
// system/core/init/first_stage_init.cpp
int FirstStageMain(int argc, char** argv) {
    if (REBOOT_BOOTLOADER_ON_PANIC) {
        InstallRebootSignalHandlers();
    }

    // 记录第一阶段启动的时间戳
    boot_clock::time_point start_time = boot_clock::now();

    std::vector<std::pair<std::string, int>> errors;
#define CHECKCALL(x) \
    if ((x) != 0) errors.emplace_back(#x " failed", errno);

    // Clear the umask.
    umask(0);

    // 挂载必须的系统文件路径
    CHECKCALL(clearenv());
    CHECKCALL(setenv("PATH", _PATH_DEFPATH, 1));
    // Get the basic filesystem setup we need put together in the initramdisk
    // on / and then we'll let the rc file figure out the rest.
    CHECKCALL(mount("tmpfs", "/dev", "tmpfs", MS_NOSUID, "mode=0755"));
    CHECKCALL(mkdir("/dev/pts", 0755));
    CHECKCALL(mkdir("/dev/socket", 0755));
    CHECKCALL(mount("devpts", "/dev/pts", "devpts", 0, NULL));
#define MAKE_STR(x) __STRING(x)
    CHECKCALL(mount("proc", "/proc", "proc", 0, "hidepid=2,gid=" MAKE_STR(AID_READPROC)));
#undef MAKE_STR
    // Don't expose the raw commandline to unprivileged processes.
    CHECKCALL(chmod("/proc/cmdline", 0440));
    std::string cmdline;
    android::base::ReadFileToString("/proc/cmdline", &cmdline);
    gid_t groups[] = {AID_READPROC};
    CHECKCALL(setgroups(arraysize(groups), groups));
    CHECKCALL(mount("sysfs", "/sys", "sysfs", 0, NULL));
    CHECKCALL(mount("selinuxfs", "/sys/fs/selinux", "selinuxfs", 0, NULL));

    // 创建kernel的Log设备节点，以后可用来读取内核的日志
    CHECKCALL(mknod("/dev/kmsg", S_IFCHR | 0600, makedev(1, 11)));

    if constexpr (WORLD_WRITABLE_KMSG) {
        CHECKCALL(mknod("/dev/kmsg_debug", S_IFCHR | 0622, makedev(1, 11)));
    }

    CHECKCALL(mknod("/dev/random", S_IFCHR | 0666, makedev(1, 8)));
    CHECKCALL(mknod("/dev/urandom", S_IFCHR | 0666, makedev(1, 9)));

    // This is needed for log wrapper, which gets called before ueventd runs.
    CHECKCALL(mknod("/dev/ptmx", S_IFCHR | 0666, makedev(5, 2)));
    CHECKCALL(mknod("/dev/null", S_IFCHR | 0666, makedev(1, 3)));

    // 下述挂载操作在第一阶段初始化完成，所以第一阶段可以挂载子目录，例如/mnt/{vendor,product}
    // 其他非第一阶段需要的挂载点，则在rc文件完成
    // Mount staging areas for devices managed by vold
    // See storage config details at http://source.android.com/devices/storage/
    CHECKCALL(mount("tmpfs", "/mnt", "tmpfs", MS_NOEXEC | MS_NOSUID | MS_NODEV,
                    "mode=0755,uid=0,gid=1000"));
    // /mnt/vendor is used to mount vendor-specific partitions that can not be
    // part of the vendor partition, e.g. because they are mounted read-write.
    CHECKCALL(mkdir("/mnt/vendor", 0755));
    // /mnt/product is used to mount product-specific partitions that can not be
    // part of the product partition, e.g. because they are mounted read-write.
    CHECKCALL(mkdir("/mnt/product", 0755));

    // /debug_ramdisk is used to preserve additional files from the debug ramdisk
    CHECKCALL(mount("tmpfs", "/debug_ramdisk", "tmpfs", MS_NOEXEC | MS_NOSUID | MS_NODEV,
                    "mode=0755,uid=0,gid=0"));
#undef CHECKCALL

    // 重定向标准输入输出输出到/dev/null
    SetStdioToDevNull(argv);
    // tmpfs已经挂载到/dev，且/dev/kmsg也存在，已经可以和外界交流
    // 初始化Kernel的日志系统
    InitKernelLogging(argv);

    // 如果上述初始化流程有错误，则打印日志到系统Log
    if (!errors.empty()) {
        for (const auto& [error_string, error_errno] : errors) {
            LOG(ERROR) << error_string << " " << strerror(error_errno);
        }
        LOG(FATAL) << "Init encountered errors starting first stage, aborting";
    }

    // 到这里第一阶段初始化工作都完成
    LOG(INFO) << "init first stage started!";

    auto old_root_dir = std::unique_ptr<DIR, decltype(&closedir)>{opendir("/"), closedir};
    if (!old_root_dir) {
        PLOG(ERROR) << "Could not opendir(\"/\"), not freeing ramdisk";
    }

    struct stat old_root_info;
    if (stat("/", &old_root_info) != 0) {
        PLOG(ERROR) << "Could not stat(\"/\"), not freeing ramdisk";
        old_root_dir.reset();
    }

    auto want_console = ALLOW_FIRST_STAGE_CONSOLE ? FirstStageConsole(cmdline) : 0;

    if (!LoadKernelModules(IsRecoveryMode() && !ForceNormalBoot(cmdline), want_console)) {
        if (want_console != FirstStageConsoleParam::DISABLED) {
            LOG(ERROR) << "Failed to load kernel modules, starting console";
        } else {
            LOG(FATAL) << "Failed to load kernel modules";
        }
    }

    if (want_console == FirstStageConsoleParam::CONSOLE_ON_FAILURE) {
        StartConsole();
    }

    if (ForceNormalBoot(cmdline)) {
        mkdir("/first_stage_ramdisk", 0755);
        // SwitchRoot() must be called with a mount point as the target, so we bind mount the
        // target directory to itself here.
        if (mount("/first_stage_ramdisk", "/first_stage_ramdisk", nullptr, MS_BIND, nullptr) != 0) {
            LOG(FATAL) << "Could not bind mount /first_stage_ramdisk to itself";
        }
        SwitchRoot("/first_stage_ramdisk");
    }

    // If this file is present, the second-stage init will use a userdebug sepolicy
    // and load adb_debug.prop to allow adb root, if the device is unlocked.
    if (access("/force_debuggable", F_OK) == 0) {
        std::error_code ec;  // to invoke the overloaded copy_file() that won't throw.
        if (!fs::copy_file("/adb_debug.prop", kDebugRamdiskProp, ec) ||
            !fs::copy_file("/userdebug_plat_sepolicy.cil", kDebugRamdiskSEPolicy, ec)) {
            LOG(ERROR) << "Failed to setup debug ramdisk";
        } else {
            // setenv for second-stage init to read above kDebugRamdisk* files.
            setenv("INIT_FORCE_DEBUGGABLE", "true", 1);
        }
    }

    if (!DoFirstStageMount()) {
        LOG(FATAL) << "Failed to mount required partitions early ...";
    }

    struct stat new_root_info;
    if (stat("/", &new_root_info) != 0) {
        PLOG(ERROR) << "Could not stat(\"/\"), not freeing ramdisk";
        old_root_dir.reset();
    }

    if (old_root_dir && old_root_info.st_dev != new_root_info.st_dev) {
        FreeRamdisk(old_root_dir.get(), old_root_info.st_dev);
    }

    SetInitAvbVersionInRecovery();

    // 记录第一阶段启动耗时
    setenv(kEnvFirstStageStartedAt, std::to_string(start_time.time_since_epoch().count()).c_str(),
           1);

    // 在第一阶段初始化函数结束前，拉起selinux_setup的初始化，以下是拉起的参数
    const char* path = "/system/bin/init";
    const char* args[] = {path, "selinux_setup", nullptr};
    auto fd = open("/dev/kmsg", O_WRONLY | O_CLOEXEC);
    dup2(fd, STDOUT_FILENO);
    dup2(fd, STDERR_FILENO);
    close(fd);
    execv(path, const_cast<char**>(args));

    // execv() only returns if an error happened, in which case we
    // panic and never fall through this conditional.
    PLOG(FATAL) << "execv(\"" << path << "\") failed";

    return 1;
}
```



### 1.3 SetupSelinux

安全增强式Linux是一个Linux内核的安全模块，其提供了访问控制安全策略机制，包括了强制访问控制。 SELinux是一组内核修改和用户空间工具，已经被添加到各种Linux发行版中。

其软件架构力图将安全决策的执行与安全策略分离，并简化涉及执行安全策略的软件的数量。

```c++
// system/core/init/selinux.cpp
int SetupSelinux(char** argv) {
    SetStdioToDevNull(argv);
    InitKernelLogging(argv);

    if (REBOOT_BOOTLOADER_ON_PANIC) {
        InstallRebootSignalHandlers();
    }

    boot_clock::time_point start_time = boot_clock::now();

    // 挂载缺失的系统分区
    MountMissingSystemPartitions();

    // Set up SELinux, loading the SELinux policy.
    SelinuxSetupKernelLogging();
    // 执行SELinux初始化工作
    SelinuxInitialize();

    // We're in the kernel domain and want to transition to the init domain.  File systems that
    // store SELabels in their xattrs, such as ext4 do not need an explicit restorecon here,
    // but other file systems do.  In particular, this is needed for ramdisks such as the
    // recovery image for A/B devices.
    if (selinux_android_restorecon("/system/bin/init", 0) == -1) {
        PLOG(FATAL) << "restorecon failed of /system/bin/init failed";
    }

    setenv(kEnvSelinuxStartedAt, std::to_string(start_time.time_since_epoch().count()).c_str(), 1);

    // SELinux初始化完毕后，把初始化工作交给第二阶段
    const char* path = "/system/bin/init";
    const char* args[] = {path, "second_stage", nullptr};
    execv(path, const_cast<char**>(args));

    // execv() only returns if an error happened, in which case we
    // panic and never return from this function.
    PLOG(FATAL) << "execv(\"" << path << "\") failed";

    return 1;
}
```



### 1.4 SecondStageMain

从 **SELinux** 结束后进入第二阶段初始化，主要执行了以下工作：

1. 设置 **SIGPIPE** 信号的处理器；
2. 加载调试相关的参数以允许adb root；
3. 系统属性的初始化、挂载额外的文件系统；
4. 为第二阶段初始化设置SELinux；
5. 创建并打开Epoll接口；
6. 初始化子进程退出信号处理函数、安装初始化通知器、启动属性服务；
7. 创建ActionManager和ServiceList实例，加载并配置rc脚本；
8. 给ActionManager添加内置操作，并添加事件触发器；
9. 在Epoll上开始死循环来处理将来事件；



action执行阶段：先执行**early-init**，然后执行**init**、最后执行**charger或late-init**。

```c++
// system/core/init/init.cpp
int SecondStageMain(int argc, char** argv) {
    if (REBOOT_BOOTLOADER_ON_PANIC) {
        InstallRebootSignalHandlers();
    }

    boot_clock::time_point start_time = boot_clock::now();

    trigger_shutdown = [](const std::string& command) { shutdown_state.TriggerShutdown(command); };

    SetStdioToDevNull(argv);
    InitKernelLogging(argv);
    LOG(INFO) << "init second stage started!";

    // Init should not crash because of a dependence on any other process, therefore we ignore
    // SIGPIPE and handle EPIPE at the call site directly.  Note that setting a signal to SIG_IGN
    // is inherited across exec, but custom signal handlers are not.  Since we do not want to
    // ignore SIGPIPE for child processes, we set a no-op function for the signal handler instead.
    {
        struct sigaction action = {.sa_flags = SA_RESTART};
        action.sa_handler = [](int) {};
        sigaction(SIGPIPE, &action, nullptr);
    }

    // Set init and its forked children's oom_adj.
    if (auto result =
                WriteFile("/proc/1/oom_score_adj", StringPrintf("%d", DEFAULT_OOM_SCORE_ADJUST));
        !result.ok()) {
        LOG(ERROR) << "Unable to write " << DEFAULT_OOM_SCORE_ADJUST
                   << " to /proc/1/oom_score_adj: " << result.error();
    }

    // Set up a session keyring that all processes will have access to. It
    // will hold things like FBE encryption keys. No process should override
    // its session keyring.
    keyctl_get_keyring_ID(KEY_SPEC_SESSION_KEYRING, 1);

    // Indicate that booting is in progress to background fw loaders, etc.
    close(open("/dev/.booting", O_WRONLY | O_CREAT | O_CLOEXEC, 0000));

    // 如果手机已解锁，则需要加载调试相关的参数以允许adb root
    const char* force_debuggable_env = getenv("INIT_FORCE_DEBUGGABLE");
    bool load_debug_prop = false;
    if (force_debuggable_env && AvbHandle::IsDeviceUnlocked()) {
        load_debug_prop = "true"s == force_debuggable_env;
    }
    unsetenv("INIT_FORCE_DEBUGGABLE");

    // 当属性服务不打算从debug ramdisk读取.prop文件时，卸载debug ramdisk
    if (!load_debug_prop) {
        UmountDebugRamdisk();
    }

    // 开始系统属性的初始化
    PropertyInit();

    // 当属性服务已经从debug ramdisk读取.prop文件后，卸载debug ramdisk
    if (load_debug_prop) {
        UmountDebugRamdisk();
    }

    // 挂载额外的文件系统
    MountExtraFilesystems();

    // 为第二阶段初始化设置SELinux
    SelinuxSetupKernelLogging();
    SelabelInitialize();
    SelinuxRestoreContext();

    // 创建并打开Epoll
    Epoll epoll;
    if (auto result = epoll.Open(); !result.ok()) {
        PLOG(FATAL) << result.error();
    }

    // 初始化子进程退出信号处理函数
    InstallSignalFdHandler(&epoll);
    // 安装初始化通知器
    InstallInitNotifier(&epoll);
    // 启动属性服务
    StartPropertyService(&property_fd);

    // Make the time that init stages started available for bootstat to log.
    RecordStageBoottimes(start_time);

    // 设置libavb版本号
    if (const char* avb_version = getenv("INIT_AVB_VERSION"); avb_version != nullptr) {
        SetProperty("ro.boot.avb_version", avb_version);
    }
    unsetenv("INIT_AVB_VERSION");

    fs_mgr_vendor_overlay_mount_all();
    export_oem_lock_status();
    MountHandler mount_handler(&epoll);
    // 设置USB控制器
    SetUsbController();

    const BuiltinFunctionMap& function_map = GetBuiltinFunctionMap();
    Action::set_function_map(&function_map);

    if (!SetupMountNamespaces()) {
        PLOG(FATAL) << "SetupMountNamespaces failed";
    }

    // 初始化Subcontext
    InitializeSubcontext();

    // ActionManager和ServiceList实例，用来解析rc文件的action、service操作
    ActionManager& am = ActionManager::GetInstance();
    ServiceList& sm = ServiceList::GetInstance();

    // 加载并配置rc脚本，传入ActionManager和ServiceList实例
    LoadBootScripts(am, sm);

    // Turning this on and letting the INFO logging be discarded adds 0.2s to
    // Nexus 9 boot time, so it's disabled by default.
    if (false) DumpState();

    // GSI相关初始化
    auto is_running = android::gsi::IsGsiRunning() ? "1" : "0";
    SetProperty(gsi::kGsiBootedProp, is_running);
    auto is_installed = android::gsi::IsGsiInstalled() ? "1" : "0";
    SetProperty(gsi::kGsiInstalledProp, is_installed);

    // ActionManager
    am.QueueBuiltinAction(SetupCgroupsAction, "SetupCgroups");
    am.QueueBuiltinAction(SetKptrRestrictAction, "SetKptrRestrict");
    am.QueueBuiltinAction(TestPerfEventSelinuxAction, "TestPerfEventSelinux");
    // 触发early-init阶段
    am.QueueEventTrigger("early-init");

    // 添加一个action来检查wait_for_coldboot_done，以便确认ueventd已经完成配置所有/dev
    am.QueueBuiltinAction(wait_for_coldboot_done_action, "wait_for_coldboot_done");
    // 并继续执行那些依赖/dev的action任务，下面添加三个action任务
    am.QueueBuiltinAction(MixHwrngIntoLinuxRngAction, "MixHwrngIntoLinuxRng");
    am.QueueBuiltinAction(SetMmapRndBitsAction, "SetMmapRndBits");
    Keychords keychords;
    am.QueueBuiltinAction(
            [&epoll, &keychords](const BuiltinArguments& args) -> Result<void> {
                for (const auto& svc : ServiceList::GetInstance()) {
                    keychords.Register(svc->keycodes());
                }
                keychords.Start(&epoll, HandleKeychord);
                return {};
            },
            "KeychordInit");

    // 触发init阶段所有引导的action开始执行
    am.QueueEventTrigger("init");

    // 在wait_for_coldboot_done流程后，若/dev/hw_random或/dev/random没有准备完成
    // 则重复执行mix_hwrng_into_linux_rng操作
    am.QueueBuiltinAction(MixHwrngIntoLinuxRngAction, "MixHwrngIntoLinuxRng");

    // 在charger模式不挂载文件系统，也不启动核心系统服务
    std::string bootmode = GetProperty("ro.bootmode", "");
    if (bootmode == "charger") {
        am.QueueEventTrigger("charger");
    } else {
        am.QueueEventTrigger("late-init");
    }

    // 基于当前状态运行所有的属性触发器
    am.QueueBuiltinAction(queue_property_triggers_action, "queue_property_triggers");

    // 开始死循环，处理上述添加的事件。如果对应子进程异常退出，则重启该子进程
    while (true) {
        // 如果没有任务，就继续休眠
        auto epoll_timeout = std::optional<std::chrono::milliseconds>{};

        // 接受到关机指令则进行关机操作
        auto shutdown_command = shutdown_state.CheckShutdown();
        if (shutdown_command) {
            HandlePowerctlMessage(*shutdown_command);
        }

        if (!(prop_waiter_state.MightBeWaiting() || Service::is_exec_service_running())) {
            am.ExecuteOneCommand();
        }
        if (!IsShuttingDown()) {
            auto next_process_action_time = HandleProcessActions();

            // 及时唤起去重启线程
            if (next_process_action_time) {
                epoll_timeout = std::chrono::ceil<std::chrono::milliseconds>(
                        *next_process_action_time - boot_clock::now());
                if (*epoll_timeout < 0ms) epoll_timeout = 0ms;
            }
        }

        if (!(prop_waiter_state.MightBeWaiting() || Service::is_exec_service_running())) {
            // 有工作则直接唤醒去执行
            if (am.HasMoreCommands()) epoll_timeout = 0ms;
        }

        // 没有工作则等待
        auto pending_functions = epoll.Wait(epoll_timeout);
        if (!pending_functions.ok()) {
            LOG(ERROR) << pending_functions.error();
        } else if (!pending_functions->empty()) {
            // We always reap children before responding to the other pending functions. This is to
            // prevent a race where other daemons see that a service has exited and ask init to
            // start it again via ctl.start before init has reaped it.
            ReapAnyOutstandingChildren();
            for (const auto& function : *pending_functions) {
                (*function)();
            }
        }
        if (!IsShuttingDown()) {
            HandleControlMessages();
            SetUsbController();
        }
    }

    return 0;
}
```

init进程初始化完成，循环等待epoll.Wait并在事件到来时处理。



## 二、系统属性

### 2.1 属性初始化

在第二阶段初始化过程中，调用了 **PropertyInit()** 函数进行属性的初始化工作。

```c++
// system/core/init/property_service.cpp
void PropertyInit() {
    selinux_callback cb;
    cb.func_audit = PropertyAuditCallback;
    selinux_set_callback(SELINUX_CB_AUDIT, cb);

    // 创建属性目录，即/dev/__properties__
    mkdir("/dev/__properties__", S_IRWXU | S_IXGRP | S_IXOTH);
    // 创建序列化的属性信息
    CreateSerializedPropertyInfo();
    // 系统属性区域初始化
    if (__system_property_area_init()) {
        LOG(FATAL) << "Failed to initialize property area";
    }
    if (!property_info_area.LoadDefaultPath()) {
        LOG(FATAL) << "Failed to load serialized property info file";
    }

    // If arguments are passed both on the command line and in DT,
    // properties set in DT always have priority over the command-line ones.
    ProcessKernelDt();
    ProcessKernelCmdline();

    // Propagate the kernel variables to internal variables
    // used by init as well as the current required properties.
    ExportKernelBootProps();

    PropertyLoadBootDefaults();
}
```



### 2.2 读取系统属性

创建序列化的属性信息：

- 把需要序列化的属性从文件中读取出来，添加到名为 **property_infos** 的 **vector**；
- 把 **property_infos** 通过 **BuildTrie** 方法构造为字典树，序列化为字符串写入到 **serialized_contexts**；
- 最后把 **serialized_contexts** 字符串写入到 **/dev/\_\_properties\_\_/property\_info**；

```c++
// system/core/init/property_service.cpp
void CreateSerializedPropertyInfo() {
    auto property_infos = std::vector<PropertyInfoEntry>();
    // 从文件中读取属性信息，添加到property_infos
    if (access("/system/etc/selinux/plat_property_contexts", R_OK) != -1) {
        if (!LoadPropertyInfoFromFile("/system/etc/selinux/plat_property_contexts",
                                      &property_infos)) {
            return;
        }
        // Don't check for failure here, so we always have a sane list of properties.
        // E.g. In case of recovery, the vendor partition will not have mounted and we
        // still need the system / platform properties to function.
        if (access("/system_ext/etc/selinux/system_ext_property_contexts", R_OK) != -1) {
            LoadPropertyInfoFromFile("/system_ext/etc/selinux/system_ext_property_contexts",
                                     &property_infos);
        }
        if (!LoadPropertyInfoFromFile("/vendor/etc/selinux/vendor_property_contexts",
                                      &property_infos)) {
            // Fallback to nonplat_* if vendor_* doesn't exist.
            LoadPropertyInfoFromFile("/vendor/etc/selinux/nonplat_property_contexts",
                                     &property_infos);
        }
        if (access("/product/etc/selinux/product_property_contexts", R_OK) != -1) {
            LoadPropertyInfoFromFile("/product/etc/selinux/product_property_contexts",
                                     &property_infos);
        }
        if (access("/odm/etc/selinux/odm_property_contexts", R_OK) != -1) {
            LoadPropertyInfoFromFile("/odm/etc/selinux/odm_property_contexts", &property_infos);
        }
    } else {
        if (!LoadPropertyInfoFromFile("/plat_property_contexts", &property_infos)) {
            return;
        }
        LoadPropertyInfoFromFile("/system_ext_property_contexts", &property_infos);
        if (!LoadPropertyInfoFromFile("/vendor_property_contexts", &property_infos)) {
            // Fallback to nonplat_* if vendor_* doesn't exist.
            LoadPropertyInfoFromFile("/nonplat_property_contexts", &property_infos);
        }
        LoadPropertyInfoFromFile("/product_property_contexts", &property_infos);
        LoadPropertyInfoFromFile("/odm_property_contexts", &property_infos);
    }
  
    // 所有属性已经文件读取并写入到property_infos
    // 这里创建名为serialized_contexts的字符串，写入property_infos序列化后的信息
    auto serialized_contexts = std::string();
    auto error = std::string();
    // 已前缀树的形式把属性信息构造为字符串，写入到serialized_contexts
    if (!BuildTrie(property_infos, "u:object_r:default_prop:s0", "string", &serialized_contexts,
                   &error)) {
        LOG(ERROR) << "Unable to serialize property contexts: " << error;
        return;
    }

    // 把serialized_contexts写入到/dev/__properties__/property_info文件
    constexpr static const char kPropertyInfosPath[] = "/dev/__properties__/property_info";
    if (!WriteStringToFile(serialized_contexts, kPropertyInfosPath, 0444, 0, 0, false)) {
        PLOG(ERROR) << "Unable to write serialized property infos to file";
    }
    // 重新设置selinux权限
    selinux_android_restorecon(kPropertyInfosPath, 0);
}
```



### 2.3 系统属性区初始化

上述流程已把属性写入 **/dev/\_\_properties\_\_/property\_info** 文件中，现在则对 **/dev/\_\_properties\_\_** 进行初始化。

```c++
// bionic/libc/bionic/system_property_api.cpp
__BIONIC_WEAK_FOR_NATIVE_BRIDGE
int __system_property_area_init() {
  bool fsetxattr_failed = false;
  return system_properties.AreaInit(PROP_FILENAME, &fsetxattr_failed) && !fsetxattr_failed ? 0 : -1;
}
```

**PROP_FILENAME** 定义如下

```c++
// bionic/tools/versioner/current/sys/_system_properties.h
#define PROP_FILENAME "/dev/__properties__"
```





### 2.4 导出内核引导属性

根据 **prop_map** 的原属性和默认值，从 **GetProperty()** 函数读取结果属性，并初始化属性集合。

```c++
static void ExportKernelBootProps() {
    constexpr const char* UNSET = "";
    struct {
        const char* src_prop;
        const char* dst_prop;
        const char* default_value;
    } prop_map[] = {
            // clang-format off
        { "ro.boot.serialno",   "ro.serialno",   UNSET, },
        { "ro.boot.mode",       "ro.bootmode",   "unknown", },
        { "ro.boot.baseband",   "ro.baseband",   "unknown", },
        { "ro.boot.bootloader", "ro.bootloader", "unknown", },
        { "ro.boot.hardware",   "ro.hardware",   "unknown", },
        { "ro.boot.revision",   "ro.revision",   "0", },
            // clang-format on
    };
    for (const auto& prop : prop_map) {
        std::string value = GetProperty(prop.src_prop, prop.default_value);
        if (value != UNSET) InitPropertySet(prop.dst_prop, value);
    }
}
```



### 2.5 加载引导属性默认值

```c++
void PropertyLoadBootDefaults() {
    // TODO(b/117892318): merge prop.default and build.prop files into one
    // We read the properties and their values into a map, in order to always allow properties
    // loaded in the later property files to override the properties in loaded in the earlier
    // property files, regardless of if they are "ro." properties or not.
    // 构建名为properties的哈希表
    std::map<std::string, std::string> properties;

    // 把属性从文件中读取出来，写入到properties
    if (!load_properties_from_file("/system/etc/prop.default", nullptr, &properties)) {
        // Try recovery path
        if (!load_properties_from_file("/prop.default", nullptr, &properties)) {
            // Try legacy path
            load_properties_from_file("/default.prop", nullptr, &properties);
        }
    }
    load_properties_from_file("/system/build.prop", nullptr, &properties);
    load_properties_from_file("/system_ext/build.prop", nullptr, &properties);
    load_properties_from_file("/vendor/default.prop", nullptr, &properties);
    load_properties_from_file("/vendor/build.prop", nullptr, &properties);
    if (SelinuxGetVendorAndroidVersion() >= __ANDROID_API_Q__) {
        load_properties_from_file("/odm/etc/build.prop", nullptr, &properties);
    } else {
        load_properties_from_file("/odm/default.prop", nullptr, &properties);
        load_properties_from_file("/odm/build.prop", nullptr, &properties);
    }
    load_properties_from_file("/product/build.prop", nullptr, &properties);
    load_properties_from_file("/factory/factory.prop", "ro.*", &properties);

    if (access(kDebugRamdiskProp, R_OK) == 0) {
        LOG(INFO) << "Loading " << kDebugRamdiskProp;
        load_properties_from_file(kDebugRamdiskProp, nullptr, &properties);
    }

    // 设置属性
    for (const auto& [name, value] : properties) {
        std::string error;
        if (PropertySet(name, value, &error) != PROP_SUCCESS) {
            LOG(ERROR) << "Could not set '" << name << "' to '" << value
                       << "' while loading .prop files" << error;
        }
    }

    property_initialize_ro_product_props();
    property_derive_build_fingerprint();

    update_sys_usb_config();
}
```



## 三、Epoll

> 维基百科：epoll是Linux内核的可扩展I/O事件通知机制。于Linux 2.5.44首度登场，它设计目的旨在取代既有POSIX select与poll系统函数，让需要大量操作文件描述符的程序得以发挥更优异的性能。epoll 实现的功能与poll 类似，都是监听多个文件描述符上的事件。



### 3.1 Epoll定义

```c++
// system/core/init/epoll.h
class Epoll {
  public:
    Epoll();

    Result<void> Open();
    Result<void> RegisterHandler(int fd, std::function<void()> handler, uint32_t events = EPOLLIN);
    Result<void> UnregisterHandler(int fd);
    Result<std::vector<std::function<void()>*>> Wait(
            std::optional<std::chrono::milliseconds> timeout);

  private:
    android::base::unique_fd epoll_fd_;
    std::map<int, std::function<void()>> epoll_handlers_;
};
```

在第二阶段初始化中，有两个与Epoll相关的设置

```c++
InstallSignalFdHandler(&epoll);
InstallInitNotifier(&epoll);
```



### 3.2 InstallSignalFdHandler

注册子进程的 **SIGCHLD** 信号监听子进程运行状态。若子进程出现挂断，则移除或重启该子进程。

```c++
// system/core/init/init.cpp
static void InstallSignalFdHandler(Epoll* epoll) {
    // Applying SA_NOCLDSTOP to a defaulted SIGCHLD handler prevents the signalfd from receiving
    // SIGCHLD when a child process stops or continues (b/77867680#comment9).
    const struct sigaction act { .sa_handler = SIG_DFL, .sa_flags = SA_NOCLDSTOP };
    sigaction(SIGCHLD, &act, nullptr);

    sigset_t mask;
    sigemptyset(&mask);
    sigaddset(&mask, SIGCHLD);

    if (!IsRebootCapable()) {
        // If init does not have the CAP_SYS_BOOT capability, it is running in a container.
        // In that case, receiving SIGTERM will cause the system to shut down.
        sigaddset(&mask, SIGTERM);
    }

    if (sigprocmask(SIG_BLOCK, &mask, nullptr) == -1) {
        PLOG(FATAL) << "failed to block signals";
    }

    // Register a handler to unblock signals in the child processes.
    const int result = pthread_atfork(nullptr, nullptr, &UnblockSignals);
    if (result != 0) {
        LOG(FATAL) << "Failed to register a fork handler: " << strerror(result);
    }

    // 创建signal_fd
    signal_fd = signalfd(-1, &mask, SFD_CLOEXEC);
    if (signal_fd == -1) {
        PLOG(FATAL) << "failed to create signalfd";
    }

    // 注册HandleSignalFd函数到signal_fd
    if (auto result = epoll->RegisterHandler(signal_fd, HandleSignalFd); !result.ok()) {
        LOG(FATAL) << result.error();
    }
}
```



### 3.2.1 HandleSignalFd

```c++
// system/core/init/init_test.cpp
static void HandleSignalFd() {
    signalfd_siginfo siginfo;
    ssize_t bytes_read = TEMP_FAILURE_RETRY(read(signal_fd, &siginfo, sizeof(siginfo)));
    if (bytes_read != sizeof(siginfo)) {
        PLOG(ERROR) << "Failed to read siginfo from signal_fd";
        return;
    }

    // 根据子进程不同信号执行对应操作
    switch (siginfo.ssi_signo) {
        case SIGCHLD:
            // 进入waitpid来处理子进程是否退出的情况
            ReapAnyOutstandingChildren();
            break;
        case SIGTERM:
            HandleSigtermSignal(siginfo);
            break;
        default:
            PLOG(ERROR) << "signal_fd: received unexpected signal " << siginfo.ssi_signo;
            break;
    }
}
```



### 3.2.2 ReapAnyOutstandingChildren

```c++
// core/init/sigchld_handler.cpp
void ReapAnyOutstandingChildren() {
    while (ReapOneProcess() != 0) {
    }
}

static pid_t ReapOneProcess() {
    siginfo_t siginfo = {};
    // This returns a zombie pid or informs us that there are no zombies left to be reaped.
    // It does NOT reap the pid; that is done below.
    if (TEMP_FAILURE_RETRY(waitid(P_ALL, 0, &siginfo, WEXITED | WNOHANG | WNOWAIT)) != 0) {
        PLOG(ERROR) << "waitid failed";
        return 0;
    }

    auto pid = siginfo.si_pid;
    if (pid == 0) return 0;

    // At this point we know we have a zombie pid, so we use this scopeguard to reap the pid
    // whenever the function returns from this point forward.
    // We do NOT want to reap the zombie earlier as in Service::Reap(), we kill(-pid, ...) and we
    // want the pid to remain valid throughout that (and potentially future) usages.
    auto reaper = make_scope_guard([pid] { TEMP_FAILURE_RETRY(waitpid(pid, nullptr, WNOHANG)); });

    std::string name;
    std::string wait_string;
    Service* service = nullptr;

    if (SubcontextChildReap(pid)) {
        name = "Subcontext";
    } else {
        service = ServiceList::GetInstance().FindService(pid, &Service::pid);

        if (service) {
            name = StringPrintf("Service '%s' (pid %d)", service->name().c_str(), pid);
            if (service->flags() & SVC_EXEC) {
                auto exec_duration = boot_clock::now() - service->time_started();
                auto exec_duration_ms =
                    std::chrono::duration_cast<std::chrono::milliseconds>(exec_duration).count();
                wait_string = StringPrintf(" waiting took %f seconds", exec_duration_ms / 1000.0f);
            } else if (service->flags() & SVC_ONESHOT) {
                auto exec_duration = boot_clock::now() - service->time_started();
                auto exec_duration_ms =
                        std::chrono::duration_cast<std::chrono::milliseconds>(exec_duration)
                                .count();
                wait_string = StringPrintf(" oneshot service took %f seconds in background",
                                           exec_duration_ms / 1000.0f);
            }
        } else {
            name = StringPrintf("Untracked pid %d", pid);
        }
    }

    if (siginfo.si_code == CLD_EXITED) {
        LOG(INFO) << name << " exited with status " << siginfo.si_status << wait_string;
    } else {
        LOG(INFO) << name << " received signal " << siginfo.si_status << wait_string;
    }

    if (!service) return pid;

    service->Reap(siginfo);

    if (service->flags() & SVC_TEMPORARY) {
        ServiceList::GetInstance().RemoveService(*service);
    }

    return pid;
}
```



### 3.2.3 HandleSigtermSignal

```c++
static void HandleSigtermSignal(const signalfd_siginfo& siginfo) {
    if (siginfo.ssi_pid != 0) {
        // Drop any userspace SIGTERM requests.
        LOG(DEBUG) << "Ignoring SIGTERM from pid " << siginfo.ssi_pid;
        return;
    }

    HandlePowerctlMessage("shutdown,container");
}
```

这里主要看 **HandlePowerctlMessage** 函数的逻辑

```c++
// core/init/reboot.cpp
void HandlePowerctlMessage(const std::string& command) {
    unsigned int cmd = 0;
    std::vector<std::string> cmd_params = Split(command, ",");
    std::string reboot_target = "";
    bool run_fsck = false;
    bool command_invalid = false;
    bool userspace_reboot = false;

    if (cmd_params[0] == "shutdown") {
        cmd = ANDROID_RB_POWEROFF;
        if (cmd_params.size() >= 2) {
            if (cmd_params[1] == "userrequested") {
                // The shutdown reason is PowerManager.SHUTDOWN_USER_REQUESTED.
                // Run fsck once the file system is remounted in read-only mode.
                run_fsck = true;
            } else if (cmd_params[1] == "thermal") {
                // Turn off sources of heat immediately.
                TurnOffBacklight();
                // run_fsck is false to avoid delay
                cmd = ANDROID_RB_THERMOFF;
            }
        }
    } else if (cmd_params[0] == "reboot") {
        cmd = ANDROID_RB_RESTART2;
        if (cmd_params.size() >= 2) {
            reboot_target = cmd_params[1];
            if (reboot_target == "userspace") {
                LOG(INFO) << "Userspace reboot requested";
                userspace_reboot = true;
            }
            // adb reboot fastboot should boot into bootloader for devices not
            // supporting logical partitions.
            if (reboot_target == "fastboot" &&
                !android::base::GetBoolProperty("ro.boot.dynamic_partitions", false)) {
                reboot_target = "bootloader";
            }
            // When rebooting to the bootloader notify the bootloader writing
            // also the BCB.
            if (reboot_target == "bootloader") {
                std::string err;
                if (!write_reboot_bootloader(&err)) {
                    LOG(ERROR) << "reboot-bootloader: Error writing "
                                  "bootloader_message: "
                               << err;
                }
            } else if (reboot_target == "recovery") {
                bootloader_message boot = {};
                if (std::string err; !read_bootloader_message(&boot, &err)) {
                    LOG(ERROR) << "Failed to read bootloader message: " << err;
                }
                // Update the boot command field if it's empty, and preserve
                // the other arguments in the bootloader message.
                if (!CommandIsPresent(&boot)) {
                    strlcpy(boot.command, "boot-recovery", sizeof(boot.command));
                    if (std::string err; !write_bootloader_message(boot, &err)) {
                        LOG(ERROR) << "Failed to set bootloader message: " << err;
                        return;
                    }
                }
            } else if (reboot_target == "sideload" || reboot_target == "sideload-auto-reboot" ||
                       reboot_target == "fastboot") {
                std::string arg = reboot_target == "sideload-auto-reboot" ? "sideload_auto_reboot"
                                                                          : reboot_target;
                const std::vector<std::string> options = {
                        "--" + arg,
                };
                std::string err;
                if (!write_bootloader_message(options, &err)) {
                    LOG(ERROR) << "Failed to set bootloader message: " << err;
                    return;
                }
                reboot_target = "recovery";
            }

            // If there are additional parameter, pass them along
            for (size_t i = 2; (cmd_params.size() > i) && cmd_params[i].size(); ++i) {
                reboot_target += "," + cmd_params[i];
            }
        }
    } else {
        command_invalid = true;
    }
    if (command_invalid) {
        LOG(ERROR) << "powerctl: unrecognized command '" << command << "'";
        return;
    }

    // We do not want to process any messages (queue'ing triggers, shutdown messages, control
    // messages, etc) from properties during reboot.
    StopSendingMessages();

    if (userspace_reboot) {
        HandleUserspaceReboot();
        return;
    }

    LOG(INFO) << "Clear action queue and start shutdown trigger";
    ActionManager::GetInstance().ClearQueue();
    // Queue shutdown trigger first
    ActionManager::GetInstance().QueueEventTrigger("shutdown");
    // Queue built-in shutdown_done
    auto shutdown_handler = [cmd, command, reboot_target, run_fsck](const BuiltinArguments&) {
        DoReboot(cmd, command, reboot_target, run_fsck);
        return Result<void>{};
    };
    ActionManager::GetInstance().QueueBuiltinAction(shutdown_handler, "shutdown_done");

    EnterShutdown();
}
```



### 3.3 InstallInitNotifier

注册 **wake_main_thread_fd** 事件的监听

```c++
// system/core/init/init.cpp

// Init epolls various FDs to wait for various inputs.  It previously waited on property changes
// with a blocking socket that contained the information related to the change, however, it was easy
// to fill that socket and deadlock the system.  Now we use locks to handle the property changes
// directly in the property thread, however we still must wake the epoll to inform init that there
// is a change to process, so we use this FD.  It is non-blocking, since we do not care how many
// times WakeMainInitThread() is called, only that the epoll will wake.
static int wake_main_thread_fd = -1;
static void InstallInitNotifier(Epoll* epoll) {
    wake_main_thread_fd = eventfd(0, EFD_CLOEXEC);
    if (wake_main_thread_fd == -1) {
        PLOG(FATAL) << "Failed to create eventfd for waking init";
    }
    auto clear_eventfd = [] {
        uint64_t counter;
        TEMP_FAILURE_RETRY(read(wake_main_thread_fd, &counter, sizeof(counter)));
    };

    if (auto result = epoll->RegisterHandler(wake_main_thread_fd, clear_eventfd); !result.ok()) {
        LOG(FATAL) << result.error();
    }
}
```



## 四、引导脚本

```c++
static void LoadBootScripts(ActionManager& action_manager, ServiceList& service_list) {
    // 用ActionManager实例和ServiceList实例创建Parser
    Parser parser = CreateParser(action_manager, service_list);

    // 从ro.boot.init_rc获取引导脚本
    std::string bootscript = GetProperty("ro.boot.init_rc", "");
    if (bootscript.empty()) {
        // 如果引导脚本为空，则从以下路径解析配置信息
        parser.ParseConfig("/system/etc/init/hw/init.rc");
        if (!parser.ParseConfig("/system/etc/init")) {
            late_import_paths.emplace_back("/system/etc/init");
        }
        // late_import is available only in Q and earlier release. As we don't
        // have system_ext in those versions, skip late_import for system_ext.
        parser.ParseConfig("/system_ext/etc/init");
        if (!parser.ParseConfig("/product/etc/init")) {
            late_import_paths.emplace_back("/product/etc/init");
        }
        if (!parser.ParseConfig("/odm/etc/init")) {
            late_import_paths.emplace_back("/odm/etc/init");
        }
        if (!parser.ParseConfig("/vendor/etc/init")) {
            late_import_paths.emplace_back("/vendor/etc/init");
        }
    } else {
        // 引导脚本不为空，则读取引导脚本的信息
        parser.ParseConfig(bootscript);
    }
}
```



#### ParseConfig

解析配置

```c++
// system/core/init/parser.cpp

bool Parser::ParseConfig(const std::string& path) {
    // 根据传入path，按照目录或文件不同区分处理
    if (is_dir(path.c_str())) {
        return ParseConfigDir(path);
    }
    return ParseConfigFile(path);
}
```



#### ParseConfigFile

遇到配置文件则直接去解析

```c++
// system/core/init/parser.cpp
bool Parser::ParseConfigFile(const std::string& path) {
    LOG(INFO) << "Parsing file " << path << "...";
    android::base::Timer t;
    auto config_contents = ReadFile(path);
    if (!config_contents.ok()) {
        LOG(INFO) << "Unable to read config file '" << path << "': " << config_contents.error();
        return false;
    }

    ParseData(path, &config_contents.value());

    LOG(VERBOSE) << "(Parsing " << path << " took " << t << ".)";
    return true;
}
```



#### ParseConfigDir

解析配置目录时，把目录下所有文件记录下来，并把所有遇到的文件送去解析

```c++
// system/core/init/parser.cpp
bool Parser::ParseConfigDir(const std::string& path) {
    LOG(INFO) << "Parsing directory " << path << "...";
    std::unique_ptr<DIR, decltype(&closedir)> config_dir(opendir(path.c_str()), closedir);
    if (!config_dir) {
        PLOG(INFO) << "Could not import directory '" << path << "'";
        return false;
    }
    dirent* current_file;
    std::vector<std::string> files;
    while ((current_file = readdir(config_dir.get()))) {
        // Ignore directories and only process regular files.
        if (current_file->d_type == DT_REG) {
            std::string current_path =
                android::base::StringPrintf("%s/%s", path.c_str(), current_file->d_name);
            files.emplace_back(current_path);
        }
    }
    // Sort first so we load files in a consistent order (bug 31996208)
    std::sort(files.begin(), files.end());
    for (const auto& file : files) {
        if (!ParseConfigFile(file)) {
            LOG(ERROR) << "could not import file '" << file << "'";
        }
    }
    return true;
}
```



#### ParseData

解析数据

```c++
void Parser::ParseData(const std::string& filename, std::string* data) {
    data->push_back('\n');  // TODO: fix tokenizer
    data->push_back('\0');

    parse_state state;
    state.line = 0;
    state.ptr = data->data();
    state.nexttoken = 0;

    SectionParser* section_parser = nullptr;
    int section_start_line = -1;
    std::vector<std::string> args;

    // If we encounter a bad section start, there is no valid parser object to parse the subsequent
    // sections, so we must suppress errors until the next valid section is found.
    bool bad_section_found = false;

    auto end_section = [&] {
        bad_section_found = false;
        if (section_parser == nullptr) return;

        if (auto result = section_parser->EndSection(); !result.ok()) {
            parse_error_count_++;
            LOG(ERROR) << filename << ": " << section_start_line << ": " << result.error();
        }

        section_parser = nullptr;
        section_start_line = -1;
    };

    for (;;) {
        switch (next_token(&state)) {
            case T_EOF:
                end_section();

                for (const auto& [section_name, section_parser] : section_parsers_) {
                    section_parser->EndFile();
                }

                return;
            case T_NEWLINE: {
                state.line++;
                if (args.empty()) break;
                // If we have a line matching a prefix we recognize, call its callback and unset any
                // current section parsers.  This is meant for /sys/ and /dev/ line entries for
                // uevent.
                auto line_callback = std::find_if(
                    line_callbacks_.begin(), line_callbacks_.end(),
                    [&args](const auto& c) { return android::base::StartsWith(args[0], c.first); });
                if (line_callback != line_callbacks_.end()) {
                    end_section();

                    if (auto result = line_callback->second(std::move(args)); !result.ok()) {
                        parse_error_count_++;
                        LOG(ERROR) << filename << ": " << state.line << ": " << result.error();
                    }
                } else if (section_parsers_.count(args[0])) {
                    end_section();
                    section_parser = section_parsers_[args[0]].get();
                    section_start_line = state.line;
                    if (auto result =
                                section_parser->ParseSection(std::move(args), filename, state.line);
                        !result.ok()) {
                        parse_error_count_++;
                        LOG(ERROR) << filename << ": " << state.line << ": " << result.error();
                        section_parser = nullptr;
                        bad_section_found = true;
                    }
                } else if (section_parser) {
                    if (auto result = section_parser->ParseLineSection(std::move(args), state.line);
                        !result.ok()) {
                        parse_error_count_++;
                        LOG(ERROR) << filename << ": " << state.line << ": " << result.error();
                    }
                } else if (!bad_section_found) {
                    parse_error_count_++;
                    LOG(ERROR) << filename << ": " << state.line
                               << ": Invalid section keyword found";
                }
                args.clear();
                break;
            }
            case T_TEXT:
                args.emplace_back(state.text);
                break;
        }
    }
}
```

