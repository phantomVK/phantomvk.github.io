---
layout:     post
title:      "init启动zygote进程"
date:       2019-01-01
author:     "phantomVK"
header-img: "img/bg/post_bg.jpg"
catalog:    false
tags:
    - Android Framework
---

## init启动

>system/core/init/init.cpp

```cpp
int main(int argc, char** argv) {
    if (!strcmp(basename(argv[0]), "ueventd")) {
        return ueventd_main(argc, argv);
    }

    if (!strcmp(basename(argv[0]), "watchdogd")) {
        return watchdogd_main(argc, argv);
    }

    if (REBOOT_BOOTLOADER_ON_PANIC) {
        install_reboot_signal_handlers();
    }

    add_environment("PATH", _PATH_DEFPATH);

    bool is_first_stage = (getenv("INIT_SECOND_STAGE") == nullptr);

    if (is_first_stage) {
        boot_clock::time_point start_time = boot_clock::now();

        // 清除umask.
        umask(0);

        // Get the basic filesystem setup we need put together in the initramdisk
        // on / and then we'll let the rc file figure out the rest.
        mount("tmpfs", "/dev", "tmpfs", MS_NOSUID, "mode=0755");
        mkdir("/dev/pts", 0755);
        mkdir("/dev/socket", 0755);
        mount("devpts", "/dev/pts", "devpts", 0, NULL);
        #define MAKE_STR(x) __STRING(x)
        mount("proc", "/proc", "proc", 0, "hidepid=2,gid=" MAKE_STR(AID_READPROC));
        // Don't expose the raw commandline to unprivileged processes.
        chmod("/proc/cmdline", 0440);
        gid_t groups[] = { AID_READPROC };
        setgroups(arraysize(groups), groups);
        mount("sysfs", "/sys", "sysfs", 0, NULL);
        mount("selinuxfs", "/sys/fs/selinux", "selinuxfs", 0, NULL);
        mknod("/dev/kmsg", S_IFCHR | 0600, makedev(1, 11));
        mknod("/dev/random", S_IFCHR | 0666, makedev(1, 8));
        mknod("/dev/urandom", S_IFCHR | 0666, makedev(1, 9));

        // Now that tmpfs is mounted on /dev and we have /dev/kmsg, we can actually
        // talk to the outside world...
        InitKernelLogging(argv);

        LOG(INFO) << "init first stage started!";

        if (!DoFirstStageMount()) {
            LOG(ERROR) << "Failed to mount required partitions early ...";
            panic();
        }

        SetInitAvbVersionInRecovery();

        // Set up SELinux, loading the SELinux policy.
        selinux_initialize(true);

        // We're in the kernel domain, so re-exec init to transition to the init domain now
        // that the SELinux policy has been loaded.
        if (restorecon("/init") == -1) {
            PLOG(ERROR) << "restorecon failed";
            security_failure();
        }

        setenv("INIT_SECOND_STAGE", "true", 1);

        static constexpr uint32_t kNanosecondsPerMillisecond = 1e6;
        uint64_t start_ms = start_time.time_since_epoch().count() / kNanosecondsPerMillisecond;
        setenv("INIT_STARTED_AT", StringPrintf("%" PRIu64, start_ms).c_str(), 1);

        char* path = argv[0];
        char* args[] = { path, nullptr };
        execv(path, args);

        // execv() only returns if an error happened, in which case we
        // panic and never fall through this conditional.
        PLOG(ERROR) << "execv(\"" << path << "\") failed";
        security_failure();
    }

    // At this point we're in the second stage of init.
    InitKernelLogging(argv);
    LOG(INFO) << "init second stage started!";

    // Set up a session keyring that all processes will have access to. It
    // will hold things like FBE encryption keys. No process should override
    // its session keyring.
    keyctl(KEYCTL_GET_KEYRING_ID, KEY_SPEC_SESSION_KEYRING, 1);

    // Indicate that booting is in progress to background fw loaders, etc.
    close(open("/dev/.booting", O_WRONLY | O_CREAT | O_CLOEXEC, 0000));

    // 初始化属性服务
    property_init();

    // If arguments are passed both on the command line and in DT,
    // properties set in DT always have priority over the command-line ones.
    process_kernel_dt();
    process_kernel_cmdline();

    // Propagate the kernel variables to internal variables
    // used by init as well as the current required properties.
    export_kernel_boot_props();

    // Make the time that init started available for bootstat to log.
    property_set("ro.boottime.init", getenv("INIT_STARTED_AT"));
    property_set("ro.boottime.init.selinux", getenv("INIT_SELINUX_TOOK"));

    // Set libavb version for Framework-only OTA match in Treble build.
    const char* avb_version = getenv("INIT_AVB_VERSION");
    if (avb_version) property_set("ro.boot.avb_version", avb_version);

    // Clean up our environment.
    unsetenv("INIT_SECOND_STAGE");
    unsetenv("INIT_STARTED_AT");
    unsetenv("INIT_SELINUX_TOOK");
    unsetenv("INIT_AVB_VERSION");

    // Now set up SELinux for second stage.
    selinux_initialize(false);
    selinux_restore_context();

    // 创建epoll
    epoll_fd = epoll_create1(EPOLL_CLOEXEC);
    if (epoll_fd == -1) {
        PLOG(ERROR) << "epoll_create1 failed";
        exit(1);
    }

    signal_handler_init();

    property_load_boot_defaults();
    export_oem_lock_status();
    // 启动属性服务
    start_property_service();
    set_usb_controller();

    const BuiltinFunctionMap function_map;
    Action::set_function_map(&function_map);

    Parser& parser = Parser::GetInstance();
    parser.AddSectionParser("service",std::make_unique<ServiceParser>());
    parser.AddSectionParser("on", std::make_unique<ActionParser>());
    parser.AddSectionParser("import", std::make_unique<ImportParser>());
    std::string bootscript = GetProperty("ro.boot.init_rc", "");
    if (bootscript.empty()) {
        parser.ParseConfig("/init.rc");
        parser.set_is_system_etc_init_loaded(
                parser.ParseConfig("/system/etc/init"));
        parser.set_is_vendor_etc_init_loaded(
                parser.ParseConfig("/vendor/etc/init"));
        parser.set_is_odm_etc_init_loaded(parser.ParseConfig("/odm/etc/init"));
    } else {
        parser.ParseConfig(bootscript);
        parser.set_is_system_etc_init_loaded(true);
        parser.set_is_vendor_etc_init_loaded(true);
        parser.set_is_odm_etc_init_loaded(true);
    }

    // Turning this on and letting the INFO logging be discarded adds 0.2s to
    // Nexus 9 boot time, so it's disabled by default.
    if (false) parser.DumpState();

    ActionManager& am = ActionManager::GetInstance();

    am.QueueEventTrigger("early-init");

    // Queue an action that waits for coldboot done so we know ueventd has set up all of /dev...
    am.QueueBuiltinAction(wait_for_coldboot_done_action, "wait_for_coldboot_done");
    // ... so that we can start queuing up actions that require stuff from /dev.
    am.QueueBuiltinAction(mix_hwrng_into_linux_rng_action, "mix_hwrng_into_linux_rng");
    am.QueueBuiltinAction(set_mmap_rnd_bits_action, "set_mmap_rnd_bits");
    am.QueueBuiltinAction(set_kptr_restrict_action, "set_kptr_restrict");
    am.QueueBuiltinAction(keychord_init_action, "keychord_init");
    am.QueueBuiltinAction(console_init_action, "console_init");

    // Trigger all the boot actions to get us started.
    am.QueueEventTrigger("init");

    // Repeat mix_hwrng_into_linux_rng in case /dev/hw_random or /dev/random
    // wasn't ready immediately after wait_for_coldboot_done
    am.QueueBuiltinAction(mix_hwrng_into_linux_rng_action, "mix_hwrng_into_linux_rng");

    // Don't mount filesystems or start core system services in charger mode.
    std::string bootmode = GetProperty("ro.bootmode", "");
    if (bootmode == "charger") {
        am.QueueEventTrigger("charger");
    } else {
        am.QueueEventTrigger("late-init");
    }

    // Run all property triggers based on current state of the properties.
    am.QueueBuiltinAction(queue_property_triggers_action, "queue_property_triggers");

    while (true) {
        // By default, sleep until something happens.
        int epoll_timeout_ms = -1;

        if (!(waiting_for_prop || ServiceManager::GetInstance().IsWaitingForExec())) {
            am.ExecuteOneCommand();
        }
        if (!(waiting_for_prop || ServiceManager::GetInstance().IsWaitingForExec())) {
            restart_processes();

            // If there's a process that needs restarting, wake up in time for that.
            if (process_needs_restart_at != 0) {
                epoll_timeout_ms = (process_needs_restart_at - time(nullptr)) * 1000;
                if (epoll_timeout_ms < 0) epoll_timeout_ms = 0;
            }

            // If there's more work to do, wake up again immediately.
            if (am.HasMoreCommands()) epoll_timeout_ms = 0;
        }

        epoll_event ev;
        int nr = TEMP_FAILURE_RETRY(epoll_wait(epoll_fd, &ev, 1, epoll_timeout_ms));
        if (nr == -1) {
            PLOG(ERROR) << "epoll_wait failed";
        } else if (nr == 1) {
            ((void (*)()) ev.data.ptr)();
        }
    }

    return 0;
}
```

## 启动Zygote

### init.rc配置

> system/core/rootdir/init.rc

```
# Copyright (C) 2012 The Android Open Source Project
#
# IMPORTANT: Do not create world writable files or directories.
# This is a common source of Android security bugs.
#

import /init.environ.rc
import /init.usb.rc
import /init.${ro.hardware}.rc
import /vendor/etc/init/hw/init.${ro.hardware}.rc
import /init.usb.configfs.rc
import /init.${ro.zygote}.rc

on early-init
    # Set init and its forked children's oom_adj.
    write /proc/1/oom_score_adj -1000

    # Disable sysrq from keyboard
    write /proc/sys/kernel/sysrq 0

    # Set the security context of /adb_keys if present.
    restorecon /adb_keys

    # Shouldn't be necessary, but sdcard won't start without it. http://b/22568628.
    mkdir /mnt 0775 root system

    # Set the security context of /postinstall if present.
    restorecon /postinstall

    start ueventd

on init
    sysclktz 0

    # Mix device-specific information into the entropy pool
    copy /proc/cmdline /dev/urandom
    copy /default.prop /dev/urandom

    # Backward compatibility.
    symlink /system/etc /etc
    symlink /sys/kernel/debug /d

    # Link /vendor to /system/vendor for devices without a vendor partition.
    symlink /system/vendor /vendor

    # Mount cgroup mount point for cpu accounting
    mount cgroup none /acct cpuacct
    mkdir /acct/uid

    # Create energy-aware scheduler tuning nodes
    mkdir /dev/stune
    mount cgroup none /dev/stune schedtune
    mkdir /dev/stune/foreground
    mkdir /dev/stune/background
    mkdir /dev/stune/top-app
    chown system system /dev/stune
    chown system system /dev/stune/foreground
    chown system system /dev/stune/background
    chown system system /dev/stune/top-app
    chown system system /dev/stune/tasks
    chown system system /dev/stune/foreground/tasks
    chown system system /dev/stune/background/tasks
    chown system system /dev/stune/top-app/tasks
    chmod 0664 /dev/stune/tasks
    chmod 0664 /dev/stune/foreground/tasks
    chmod 0664 /dev/stune/background/tasks
    chmod 0664 /dev/stune/top-app/tasks

    # Mount staging areas for devices managed by vold
    # See storage config details at http://source.android.com/tech/storage/
    mount tmpfs tmpfs /mnt mode=0755,uid=0,gid=1000
    restorecon_recursive /mnt

    mount configfs none /config
    chmod 0775 /config/sdcardfs
    chown system package_info /config/sdcardfs

    mkdir /mnt/secure 0700 root root
    mkdir /mnt/secure/asec 0700 root root
    mkdir /mnt/asec 0755 root system
    mkdir /mnt/obb 0755 root system
    mkdir /mnt/media_rw 0750 root media_rw
    mkdir /mnt/user 0755 root root
    mkdir /mnt/user/0 0755 root root
    mkdir /mnt/expand 0771 system system
    mkdir /mnt/appfuse 0711 root root

    # Storage views to support runtime permissions
    mkdir /mnt/runtime 0700 root root
    mkdir /mnt/runtime/default 0755 root root
    mkdir /mnt/runtime/default/self 0755 root root
    mkdir /mnt/runtime/read 0755 root root
    mkdir /mnt/runtime/read/self 0755 root root
    mkdir /mnt/runtime/write 0755 root root
    mkdir /mnt/runtime/write/self 0755 root root

    # Symlink to keep legacy apps working in multi-user world
    symlink /storage/self/primary /sdcard
    symlink /storage/self/primary /mnt/sdcard
    symlink /mnt/user/0/primary /mnt/runtime/default/self/primary

    # root memory control cgroup, used by lmkd
    mkdir /dev/memcg 0700 root system
    mount cgroup none /dev/memcg memory
    # app mem cgroups, used by activity manager, lmkd and zygote
    mkdir /dev/memcg/apps/ 0755 system system

    write /proc/sys/kernel/panic_on_oops 1
    write /proc/sys/kernel/hung_task_timeout_secs 0
    write /proc/cpu/alignment 4

    # scheduler tunables
    # Disable auto-scaling of scheduler tunables with hotplug. The tunables
    # will vary across devices in unpredictable ways if allowed to scale with
    # cpu cores.
    write /proc/sys/kernel/sched_tunable_scaling 0
    write /proc/sys/kernel/sched_latency_ns 10000000
    write /proc/sys/kernel/sched_wakeup_granularity_ns 2000000
    write /proc/sys/kernel/sched_child_runs_first 0

    write /proc/sys/kernel/randomize_va_space 2
    write /proc/sys/vm/mmap_min_addr 32768
    write /proc/sys/net/ipv4/ping_group_range "0 2147483647"
    write /proc/sys/net/unix/max_dgram_qlen 600
    write /proc/sys/kernel/sched_rt_runtime_us 950000
    write /proc/sys/kernel/sched_rt_period_us 1000000

    # Assign reasonable ceiling values for socket rcv/snd buffers.
    # These should almost always be overridden by the target per the
    # the corresponding technology maximums.
    write /proc/sys/net/core/rmem_max  262144
    write /proc/sys/net/core/wmem_max  262144

    # reflect fwmark from incoming packets onto generated replies
    write /proc/sys/net/ipv4/fwmark_reflect 1
    write /proc/sys/net/ipv6/fwmark_reflect 1

    # set fwmark on accepted sockets
    write /proc/sys/net/ipv4/tcp_fwmark_accept 1

    # disable icmp redirects
    write /proc/sys/net/ipv4/conf/all/accept_redirects 0
    write /proc/sys/net/ipv6/conf/all/accept_redirects 0

    # Create cgroup mount points for process groups
    mkdir /dev/cpuctl
    mount cgroup none /dev/cpuctl cpu
    chown system system /dev/cpuctl
    chown system system /dev/cpuctl/tasks
    chmod 0666 /dev/cpuctl/tasks
    write /dev/cpuctl/cpu.rt_period_us 1000000
    write /dev/cpuctl/cpu.rt_runtime_us 950000

    # sets up initial cpusets for ActivityManager
    mkdir /dev/cpuset
    mount cpuset none /dev/cpuset

    # this ensures that the cpusets are present and usable, but the device's
    # init.rc must actually set the correct cpus
    mkdir /dev/cpuset/foreground
    copy /dev/cpuset/cpus /dev/cpuset/foreground/cpus
    copy /dev/cpuset/mems /dev/cpuset/foreground/mems
    mkdir /dev/cpuset/foreground/boost
    copy /dev/cpuset/cpus /dev/cpuset/foreground/boost/cpus
    copy /dev/cpuset/mems /dev/cpuset/foreground/boost/mems
    mkdir /dev/cpuset/background
    copy /dev/cpuset/cpus /dev/cpuset/background/cpus
    copy /dev/cpuset/mems /dev/cpuset/background/mems

    # system-background is for system tasks that should only run on
    # little cores, not on bigs
    # to be used only by init, so don't change system-bg permissions
    mkdir /dev/cpuset/system-background
    copy /dev/cpuset/cpus /dev/cpuset/system-background/cpus
    copy /dev/cpuset/mems /dev/cpuset/system-background/mems

    mkdir /dev/cpuset/top-app
    copy /dev/cpuset/cpus /dev/cpuset/top-app/cpus
    copy /dev/cpuset/mems /dev/cpuset/top-app/mems

    # change permissions for all cpusets we'll touch at runtime
    chown system system /dev/cpuset
    chown system system /dev/cpuset/foreground
    chown system system /dev/cpuset/foreground/boost
    chown system system /dev/cpuset/background
    chown system system /dev/cpuset/system-background
    chown system system /dev/cpuset/top-app
    chown system system /dev/cpuset/tasks
    chown system system /dev/cpuset/foreground/tasks
    chown system system /dev/cpuset/foreground/boost/tasks
    chown system system /dev/cpuset/background/tasks
    chown system system /dev/cpuset/system-background/tasks
    chown system system /dev/cpuset/top-app/tasks

    # set system-background to 0775 so SurfaceFlinger can touch it
    chmod 0775 /dev/cpuset/system-background

    chmod 0664 /dev/cpuset/foreground/tasks
    chmod 0664 /dev/cpuset/foreground/boost/tasks
    chmod 0664 /dev/cpuset/background/tasks
    chmod 0664 /dev/cpuset/system-background/tasks
    chmod 0664 /dev/cpuset/top-app/tasks
    chmod 0664 /dev/cpuset/tasks


    # qtaguid will limit access to specific data based on group memberships.
    #   net_bw_acct grants impersonation of socket owners.
    #   net_bw_stats grants access to other apps' detailed tagged-socket stats.
    chown root net_bw_acct /proc/net/xt_qtaguid/ctrl
    chown root net_bw_stats /proc/net/xt_qtaguid/stats

    # Allow everybody to read the xt_qtaguid resource tracking misc dev.
    # This is needed by any process that uses socket tagging.
    chmod 0644 /dev/xt_qtaguid

    # Create location for fs_mgr to store abbreviated output from filesystem
    # checker programs.
    mkdir /dev/fscklogs 0770 root system

    # pstore/ramoops previous console log
    mount pstore pstore /sys/fs/pstore
    chown system log /sys/fs/pstore/console-ramoops
    chmod 0440 /sys/fs/pstore/console-ramoops
    chown system log /sys/fs/pstore/pmsg-ramoops-0
    chmod 0440 /sys/fs/pstore/pmsg-ramoops-0

    # enable armv8_deprecated instruction hooks
    write /proc/sys/abi/swp 1

    # Linux's execveat() syscall may construct paths containing /dev/fd
    # expecting it to point to /proc/self/fd
    symlink /proc/self/fd /dev/fd

    export DOWNLOAD_CACHE /data/cache

    # set RLIMIT_NICE to allow priorities from 19 to -20
    setrlimit 13 40 40

    # This allows the ledtrig-transient properties to be created here so
    # that they can be chown'd to system:system later on boot
    write /sys/class/leds/vibrator/trigger "transient"

# Healthd can trigger a full boot from charger mode by signaling this
# property when the power button is held.
on property:sys.boot_from_charger_mode=1
    class_stop charger
    trigger late-init

on load_persist_props_action
    load_persist_props
    start logd
    start logd-reinit

# Indicate to fw loaders that the relevant mounts are up.
on firmware_mounts_complete
    rm /dev/.booting

# Mount filesystems and start core system services.
on late-init
    trigger early-fs

    # Mount fstab in init.{$device}.rc by mount_all command. Optional parameter
    # '--early' can be specified to skip entries with 'latemount'.
    # /system and /vendor must be mounted by the end of the fs stage,
    # while /data is optional.
    trigger fs
    trigger post-fs

    # Mount fstab in init.{$device}.rc by mount_all with '--late' parameter
    # to only mount entries with 'latemount'. This is needed if '--early' is
    # specified in the previous mount_all command on the fs stage.
    # With /system mounted and properties form /system + /factory available,
    # some services can be started.
    trigger late-fs

    # Now we can mount /data. File encryption requires keymaster to decrypt
    # /data, which in turn can only be loaded when system properties are present.
    trigger post-fs-data

    # Now we can start zygote for devices with file based encryption
    trigger zygote-start

    # Load persist properties and override properties (if enabled) from /data.
    trigger load_persist_props_action

    # Remove a file to wake up anything waiting for firmware.
    trigger firmware_mounts_complete

    trigger early-boot
    trigger boot

on post-fs
    # Load properties from
    #     /system/build.prop,
    #     /odm/build.prop,
    #     /vendor/build.prop and
    #     /factory/factory.prop
    load_system_props
    # start essential services
    start logd
    start servicemanager
    start hwservicemanager
    start vndservicemanager

    # once everything is setup, no need to modify /
    mount rootfs rootfs / ro remount
    # Mount shared so changes propagate into child namespaces
    mount rootfs rootfs / shared rec
    # Mount default storage into root namespace
    mount none /mnt/runtime/default /storage bind rec
    mount none none /storage slave rec

    # Make sure /sys/kernel/debug (if present) is labeled properly
    # Note that tracefs may be mounted under debug, so we need to cross filesystems
    restorecon --recursive --cross-filesystems /sys/kernel/debug
    chmod 0755 /sys/kernel/debug/tracing

    # We chown/chmod /cache again so because mount is run as root + defaults
    chown system cache /cache
    chmod 0770 /cache
    # We restorecon /cache in case the cache partition has been reset.
    restorecon_recursive /cache

    # Create /cache/recovery in case it's not there. It'll also fix the odd
    # permissions if created by the recovery system.
    mkdir /cache/recovery 0770 system cache

    # Backup/restore mechanism uses the cache partition
    mkdir /cache/backup_stage 0700 system system
    mkdir /cache/backup 0700 system system

    #change permissions on vmallocinfo so we can grab it from bugreports
    chown root log /proc/vmallocinfo
    chmod 0440 /proc/vmallocinfo

    chown root log /proc/slabinfo
    chmod 0440 /proc/slabinfo

    #change permissions on kmsg & sysrq-trigger so bugreports can grab kthread stacks
    chown root system /proc/kmsg
    chmod 0440 /proc/kmsg
    chown root system /proc/sysrq-trigger
    chmod 0220 /proc/sysrq-trigger
    chown system log /proc/last_kmsg
    chmod 0440 /proc/last_kmsg

    # make the selinux kernel policy world-readable
    chmod 0444 /sys/fs/selinux/policy

    # create the lost+found directories, so as to enforce our permissions
    mkdir /cache/lost+found 0770 root root

on late-fs
    # HALs required before storage encryption can get unlocked (FBE/FDE)
    class_start early_hal

on post-fs-data
    # We chown/chmod /data again so because mount is run as root + defaults
    chown system system /data
    chmod 0771 /data
    # We restorecon /data in case the userdata partition has been reset.
    restorecon /data

    # Make sure we have the device encryption key.
    start vold
    installkey /data

    # Start bootcharting as soon as possible after the data partition is
    # mounted to collect more data.
    mkdir /data/bootchart 0755 shell shell
    bootchart start

    # Avoid predictable entropy pool. Carry over entropy from previous boot.
    copy /data/system/entropy.dat /dev/urandom

    # create basic filesystem structure
    mkdir /data/misc 01771 system misc
    mkdir /data/misc/bluedroid 02770 bluetooth bluetooth
    # Fix the access permissions and group ownership for 'bt_config.conf'
    chmod 0660 /data/misc/bluedroid/bt_config.conf
    chown bluetooth bluetooth /data/misc/bluedroid/bt_config.conf
    mkdir /data/misc/bluetooth 0770 bluetooth bluetooth
    mkdir /data/misc/bluetooth/logs 0770 bluetooth bluetooth
    mkdir /data/misc/keystore 0700 keystore keystore
    mkdir /data/misc/gatekeeper 0700 system system
    mkdir /data/misc/keychain 0771 system system
    mkdir /data/misc/net 0750 root shell
    mkdir /data/misc/radio 0770 system radio
    mkdir /data/misc/sms 0770 system radio
    mkdir /data/misc/zoneinfo 0775 system system
    mkdir /data/misc/textclassifier 0771 system system
    mkdir /data/misc/vpn 0770 system vpn
    mkdir /data/misc/shared_relro 0771 shared_relro shared_relro
    mkdir /data/misc/systemkeys 0700 system system
    mkdir /data/misc/wifi 0770 wifi wifi
    mkdir /data/misc/wifi/sockets 0770 wifi wifi
    mkdir /data/misc/wifi/wpa_supplicant 0770 wifi wifi
    mkdir /data/misc/ethernet 0770 system system
    mkdir /data/misc/dhcp 0770 dhcp dhcp
    mkdir /data/misc/user 0771 root root
    mkdir /data/misc/perfprofd 0775 root root
    # give system access to wpa_supplicant.conf for backup and restore
    chmod 0660 /data/misc/wifi/wpa_supplicant.conf
    mkdir /data/local 0751 root root
    mkdir /data/misc/media 0700 media media
    mkdir /data/misc/audioserver 0700 audioserver audioserver
    mkdir /data/misc/cameraserver 0700 cameraserver cameraserver
    mkdir /data/misc/vold 0700 root root
    mkdir /data/misc/boottrace 0771 system shell
    mkdir /data/misc/update_engine 0700 root root
    mkdir /data/misc/trace 0700 root root
    mkdir /data/misc/reboot 0700 system system
    # profile file layout
    mkdir /data/misc/profiles 0771 system system
    mkdir /data/misc/profiles/cur 0771 system system
    mkdir /data/misc/profiles/ref 0771 system system
    mkdir /data/misc/profman 0770 system shell
    mkdir /data/misc/gcov 0770 root root

    mkdir /data/vendor 0771 root root
    mkdir /data/vendor/hardware 0771 root root

    # For security reasons, /data/local/tmp should always be empty.
    # Do not place files or directories in /data/local/tmp
    mkdir /data/local/tmp 0771 shell shell
    mkdir /data/data 0771 system system
    mkdir /data/app-private 0771 system system
    mkdir /data/app-ephemeral 0771 system system
    mkdir /data/app-asec 0700 root root
    mkdir /data/app-lib 0771 system system
    mkdir /data/app 0771 system system
    mkdir /data/property 0700 root root
    mkdir /data/tombstones 0771 system system

    # create dalvik-cache, so as to enforce our permissions
    mkdir /data/dalvik-cache 0771 root root
    # create the A/B OTA directory, so as to enforce our permissions
    mkdir /data/ota 0771 root root

    # create the OTA package directory. It will be accessed by GmsCore (cache
    # group), update_engine and update_verifier.
    mkdir /data/ota_package 0770 system cache

    # create resource-cache and double-check the perms
    mkdir /data/resource-cache 0771 system system
    chown system system /data/resource-cache
    chmod 0771 /data/resource-cache

    # create the lost+found directories, so as to enforce our permissions
    mkdir /data/lost+found 0770 root root

    # create directory for DRM plug-ins - give drm the read/write access to
    # the following directory.
    mkdir /data/drm 0770 drm drm

    # create directory for MediaDrm plug-ins - give drm the read/write access to
    # the following directory.
    mkdir /data/mediadrm 0770 mediadrm mediadrm

    mkdir /data/anr 0775 system system

    # Create all remaining /data root dirs so that they are made through init
    # and get proper encryption policy installed
    mkdir /data/backup 0700 system system
    mkdir /data/ss 0700 system system

    mkdir /data/system 0775 system system
    mkdir /data/system/heapdump 0700 system system
    mkdir /data/system/users 0775 system system

    mkdir /data/system_de 0770 system system
    mkdir /data/system_ce 0770 system system

    mkdir /data/misc_de 01771 system misc
    mkdir /data/misc_ce 01771 system misc

    mkdir /data/user 0711 system system
    mkdir /data/user_de 0711 system system
    symlink /data/data /data/user/0

    mkdir /data/media 0770 media_rw media_rw
    mkdir /data/media/obb 0770 media_rw media_rw

    mkdir /data/cache 0770 system cache
    mkdir /data/cache/recovery 0770 system cache
    mkdir /data/cache/backup_stage 0700 system system
    mkdir /data/cache/backup 0700 system system

    init_user0

    # Set SELinux security contexts on upgrade or policy update.
    restorecon --recursive --skip-ce /data

    # Check any timezone data in /data is newer than the copy in /system, delete if not.
    exec - system system -- /system/bin/tzdatacheck /system/usr/share/zoneinfo /data/misc/zoneinfo

    # If there is no post-fs-data action in the init.<device>.rc file, you
    # must uncomment this line, otherwise encrypted filesystems
    # won't work.
    # Set indication (checked by vold) that we have finished this action
    #setprop vold.post_fs_data_done 1

# It is recommended to put unnecessary data/ initialization from post-fs-data
# to start-zygote in device's init.rc to unblock zygote start.
on zygote-start && property:ro.crypto.state=unencrypted
    # A/B update verifier that marks a successful boot.
    exec_start update_verifier_nonencrypted
    start netd
    start zygote
    start zygote_secondary

on zygote-start && property:ro.crypto.state=unsupported
    # A/B update verifier that marks a successful boot.
    exec_start update_verifier_nonencrypted
    start netd
    start zygote
    start zygote_secondary

on zygote-start && property:ro.crypto.state=encrypted && property:ro.crypto.type=file
    # A/B update verifier that marks a successful boot.
    exec_start update_verifier_nonencrypted
    start netd
    start zygote
    start zygote_secondary

on boot
    # basic network init
    ifup lo
    hostname localhost
    domainname localdomain

    # Memory management.  Basic kernel parameters, and allow the high
    # level system server to be able to adjust the kernel OOM driver
    # parameters to match how it is managing things.
    write /proc/sys/vm/overcommit_memory 1
    write /proc/sys/vm/min_free_order_shift 4
    chown root system /sys/module/lowmemorykiller/parameters/adj
    chmod 0664 /sys/module/lowmemorykiller/parameters/adj
    chown root system /sys/module/lowmemorykiller/parameters/minfree
    chmod 0664 /sys/module/lowmemorykiller/parameters/minfree

    # Tweak background writeout
    write /proc/sys/vm/dirty_expire_centisecs 200
    write /proc/sys/vm/dirty_background_ratio  5

    # Permissions for System Server and daemons.
    chown radio system /sys/android_power/state
    chown radio system /sys/android_power/request_state
    chown radio system /sys/android_power/acquire_full_wake_lock
    chown radio system /sys/android_power/acquire_partial_wake_lock
    chown radio system /sys/android_power/release_wake_lock
    chown system system /sys/power/autosleep
    chown system system /sys/power/state
    chown system system /sys/power/wakeup_count
    chown radio wakelock /sys/power/wake_lock
    chown radio wakelock /sys/power/wake_unlock
    chmod 0660 /sys/power/state
    chmod 0660 /sys/power/wake_lock
    chmod 0660 /sys/power/wake_unlock

    chown system system /sys/devices/system/cpu/cpufreq/interactive/timer_rate
    chmod 0660 /sys/devices/system/cpu/cpufreq/interactive/timer_rate
    chown system system /sys/devices/system/cpu/cpufreq/interactive/timer_slack
    chmod 0660 /sys/devices/system/cpu/cpufreq/interactive/timer_slack
    chown system system /sys/devices/system/cpu/cpufreq/interactive/min_sample_time
    chmod 0660 /sys/devices/system/cpu/cpufreq/interactive/min_sample_time
    chown system system /sys/devices/system/cpu/cpufreq/interactive/hispeed_freq
    chmod 0660 /sys/devices/system/cpu/cpufreq/interactive/hispeed_freq
    chown system system /sys/devices/system/cpu/cpufreq/interactive/target_loads
    chmod 0660 /sys/devices/system/cpu/cpufreq/interactive/target_loads
    chown system system /sys/devices/system/cpu/cpufreq/interactive/go_hispeed_load
    chmod 0660 /sys/devices/system/cpu/cpufreq/interactive/go_hispeed_load
    chown system system /sys/devices/system/cpu/cpufreq/interactive/above_hispeed_delay
    chmod 0660 /sys/devices/system/cpu/cpufreq/interactive/above_hispeed_delay
    chown system system /sys/devices/system/cpu/cpufreq/interactive/boost
    chmod 0660 /sys/devices/system/cpu/cpufreq/interactive/boost
    chown system system /sys/devices/system/cpu/cpufreq/interactive/boostpulse
    chown system system /sys/devices/system/cpu/cpufreq/interactive/input_boost
    chmod 0660 /sys/devices/system/cpu/cpufreq/interactive/input_boost
    chown system system /sys/devices/system/cpu/cpufreq/interactive/boostpulse_duration
    chmod 0660 /sys/devices/system/cpu/cpufreq/interactive/boostpulse_duration
    chown system system /sys/devices/system/cpu/cpufreq/interactive/io_is_busy
    chmod 0660 /sys/devices/system/cpu/cpufreq/interactive/io_is_busy

    # Assume SMP uses shared cpufreq policy for all CPUs
    chown system system /sys/devices/system/cpu/cpu0/cpufreq/scaling_max_freq
    chmod 0660 /sys/devices/system/cpu/cpu0/cpufreq/scaling_max_freq

    chown system system /sys/class/leds/vibrator/trigger
    chown system system /sys/class/leds/vibrator/activate
    chown system system /sys/class/leds/vibrator/brightness
    chown system system /sys/class/leds/vibrator/duration
    chown system system /sys/class/leds/vibrator/state
    chown system system /sys/class/timed_output/vibrator/enable
    chown system system /sys/class/leds/keyboard-backlight/brightness
    chown system system /sys/class/leds/lcd-backlight/brightness
    chown system system /sys/class/leds/button-backlight/brightness
    chown system system /sys/class/leds/jogball-backlight/brightness
    chown system system /sys/class/leds/red/brightness
    chown system system /sys/class/leds/green/brightness
    chown system system /sys/class/leds/blue/brightness
    chown system system /sys/class/leds/red/device/grpfreq
    chown system system /sys/class/leds/red/device/grppwm
    chown system system /sys/class/leds/red/device/blink
    chown system system /sys/module/sco/parameters/disable_esco
    chown system system /sys/kernel/ipv4/tcp_wmem_min
    chown system system /sys/kernel/ipv4/tcp_wmem_def
    chown system system /sys/kernel/ipv4/tcp_wmem_max
    chown system system /sys/kernel/ipv4/tcp_rmem_min
    chown system system /sys/kernel/ipv4/tcp_rmem_def
    chown system system /sys/kernel/ipv4/tcp_rmem_max
    chown root radio /proc/cmdline

    # Define default initial receive window size in segments.
    setprop net.tcp.default_init_rwnd 60

    # Start standard binderized HAL daemons
    class_start hal

    class_start core

on nonencrypted
    class_start main
    class_start late_start

on property:sys.init_log_level=*
    loglevel ${sys.init_log_level}

on charger
    class_start charger

on property:vold.decrypt=trigger_reset_main
    class_reset main

on property:vold.decrypt=trigger_load_persist_props
    load_persist_props
    start logd
    start logd-reinit

on property:vold.decrypt=trigger_post_fs_data
    trigger post-fs-data

on property:vold.decrypt=trigger_restart_min_framework
    # A/B update verifier that marks a successful boot.
    exec_start update_verifier
    class_start main

on property:vold.decrypt=trigger_restart_framework
    # A/B update verifier that marks a successful boot.
    exec_start update_verifier
    class_start main
    class_start late_start

on property:vold.decrypt=trigger_shutdown_framework
    class_reset late_start
    class_reset main

on property:sys.boot_completed=1
    bootchart stop

# system server cannot write to /proc/sys files,
# and chown/chmod does not work for /proc/sys/ entries.
# So proxy writes through init.
on property:sys.sysctl.extra_free_kbytes=*
    write /proc/sys/vm/extra_free_kbytes ${sys.sysctl.extra_free_kbytes}

# "tcp_default_init_rwnd" Is too long!
on property:sys.sysctl.tcp_def_init_rwnd=*
    write /proc/sys/net/ipv4/tcp_default_init_rwnd ${sys.sysctl.tcp_def_init_rwnd}

on property:security.perf_harden=0
    write /proc/sys/kernel/perf_event_paranoid 1

on property:security.perf_harden=1
    write /proc/sys/kernel/perf_event_paranoid 3

## Daemon processes to be run by init.
##
service ueventd /sbin/ueventd
    class core
    critical
    seclabel u:r:ueventd:s0

service healthd /system/bin/healthd
    class core
    critical
    group root system wakelock

service console /system/bin/sh
    class core
    console
    disabled
    user shell
    group shell log readproc
    seclabel u:r:shell:s0

on property:ro.debuggable=1
    # Give writes to anyone for the trace folder on debug builds.
    # The folder is used to store method traces.
    chmod 0773 /data/misc/trace
    start console

service flash_recovery /system/bin/install-recovery.sh
    class main
    oneshot
```

### Zygote配置

> system/core/rootdir/init.zygote64.rc

**init进程** 要启动名为 **zygote** 的新进程，新进程执行程序是 **/system/bin/app_process64**，后面的都是传给 **app_process64** 的参数

```
service zygote /system/bin/app_process64 -Xzygote /system/bin --zygote --start-system-server
    class main
    priority -20
    user root
    group root readproc
    socket zygote stream 660 root system
    onrestart write /sys/android_power/request_state wake
    onrestart write /sys/power/state on
    onrestart restart audioserver
    onrestart restart cameraserver
    onrestart restart media
    onrestart restart netd
    onrestart restart wificond
    writepid /dev/cpuset/foreground/tasks
```

### 解析头

> system/core/init/service.cpp

```cpp
bool ServiceParser::ParseSection(const std::vector<std::string>& args,
                                 std::string* err) {
    // 检查至少需要传入3个参数，例如："service zygote /system/bin/app_process64"
    if (args.size() < 3) {
        *err = "services must have a name and a program";
        return false;
    }

    // name的值为"zygote"
    const std::string& name = args[1];
    // 验证服务名字有效性
    if (!IsValidName(name)) {
        *err = StringPrintf("invalid service name '%s'", name.c_str());
        return false;
    }

    // 处理参数列表，分割每个参数并保存为vector
    std::vector<std::string> str_args(args.begin() + 2, args.end());
    // 用服务名和服务参数创建Service
    service_ = std::make_unique<Service>(name, str_args);
    return true;
}

// 除了解析头部，还需要逐行解析头部以下参数配置
bool ServiceParser::ParseLineSection(const std::vector<std::string>& args,
                                     const std::string& filename, int line,
                                     std::string* err) const {
    return service_ ? service_->ParseLine(args, err) : false;
}

bool Service::ParseLine(const std::vector<std::string>& args, std::string* err) {
    if (args.empty()) {
        *err = "option needed, but not provided";
        return false;
    }

    static const OptionParserMap parser_map;
    auto parser = parser_map.FindFunction(args[0], args.size() - 1, err);

    if (!parser) {
        return false;
    }

    return (this->*parser)(args, err);
}

// 服务已创建且参数全部读取完毕
void ServiceParser::EndSection() {
    if (service_) {
        // 把该服务添加到ServiceManager服务列表中
        ServiceManager::GetInstance().AddService(std::move(service_));
    }
}

// 把该服务添加到ServiceManager服务列表中
void ServiceManager::AddService(std::unique_ptr<Service> service) {
    // 根据传入服务名查找是否有相同服务存在
    Service* old_service = FindServiceByName(service->name());
    if (old_service) {
        // 同名服务已存在，新服务不会添加
        LOG(ERROR) << "ignored duplicate definition of service '" << service->name() << "'";
        return;
    }
    // 没有同名服务，用传入服务名注册该服务
    services_.emplace_back(std::move(service));
}
```

### 启动服务

上面只是把服务用对应名称保存到 **ServiceManager**，下面开始逐个启动服务

> system/core/init/builtins.cpp

```cpp
static int do_class_start(const std::vector<std::string>& args) {
    /* Starting a class does not start services
     * which are explicitly disabled.  They must
     * be started individually.
     */
    // 根据传入的Service参数从列表找到service
    // 并用StartIfNotDisabled()检查服务是否允许启动
    ServiceManager::GetInstance().
        ForEachServiceInClass(args[1], [] (Service* s) { s->StartIfNotDisabled(); });
    return 0;
}
```

### 检查服务可否启动

> system/core/init/service.cpp

```cpp
bool Service::StartIfNotDisabled() {
    if (!(flags_ & SVC_DISABLED)) {
        // 服务没有SVC_DISABLED标志则启动服务
        return Start();
    } else {
        flags_ |= SVC_DISABLED_START;
    }
    return true;
}
```

### 启动服务

当 **zygote** 的 **Service.start()** 被调用时，进入以下方法，即 **init** 实际启动服务的步骤

```cpp
bool Service::Start() {
    // Starting a service removes it from the disabled or reset state and
    // immediately takes it out of the restarting state if it was in there.
    flags_ &= (~(SVC_DISABLED|SVC_RESTARTING|SVC_RESET|SVC_RESTART|SVC_DISABLED_START));

    // Running processes require no additional work --- if they're in the
    // process of exiting, we've ensured that they will immediately restart
    // on exit, unless they are ONESHOT.
    if (flags_ & SVC_RUNNING) {
        return false;
    }

    bool needs_console = (flags_ & SVC_CONSOLE);
    if (needs_console) {
        if (console_.empty()) {
            console_ = default_console;
        }

        // Make sure that open call succeeds to ensure a console driver is
        // properly registered for the device node
        int console_fd = open(console_.c_str(), O_RDWR | O_CLOEXEC);
        if (console_fd < 0) {
            PLOG(ERROR) << "service '" << name_ << "' couldn't open console '" << console_ << "'";
            flags_ |= SVC_DISABLED;
            return false;
        }
        close(console_fd);
    }

    struct stat sb;
    if (stat(args_[0].c_str(), &sb) == -1) {
        PLOG(ERROR) << "cannot find '" << args_[0] << "', disabling '" << name_ << "'";
        flags_ |= SVC_DISABLED;
        return false;
    }

    std::string scon;
    if (!seclabel_.empty()) {
        scon = seclabel_;
    } else {
        LOG(INFO) << "computing context for service '" << name_ << "'";
        scon = ComputeContextFromExecutable(name_, args_[0]);
        if (scon == "") {
            return false;
        }
    }

    LOG(INFO) << "starting service '" << name_ << "'...";

    pid_t pid = -1;
    if (namespace_flags_) {
        pid = clone(nullptr, nullptr, namespace_flags_ | SIGCHLD, nullptr);
    } else {
        pid = fork();
    }

    if (pid == 0) {
        umask(077);

        if (namespace_flags_ & CLONE_NEWPID) {
            // This will fork again to run an init process inside the PID
            // namespace.
            SetUpPidNamespace(name_);
        }

        for (const auto& ei : envvars_) {
            add_environment(ei.name.c_str(), ei.value.c_str());
        }

        std::for_each(descriptors_.begin(), descriptors_.end(),
                      std::bind(&DescriptorInfo::CreateAndPublish, std::placeholders::_1, scon));

        // See if there were "writepid" instructions to write to files under /dev/cpuset/.
        auto cpuset_predicate = [](const std::string& path) {
            return android::base::StartsWith(path, "/dev/cpuset/");
        };
        auto iter = std::find_if(writepid_files_.begin(), writepid_files_.end(), cpuset_predicate);
        if (iter == writepid_files_.end()) {
            // There were no "writepid" instructions for cpusets, check if the system default
            // cpuset is specified to be used for the process.
            std::string default_cpuset = android::base::GetProperty("ro.cpuset.default", "");
            if (!default_cpuset.empty()) {
                // Make sure the cpuset name starts and ends with '/'.
                // A single '/' means the 'root' cpuset.
                if (default_cpuset.front() != '/') {
                    default_cpuset.insert(0, 1, '/');
                }
                if (default_cpuset.back() != '/') {
                    default_cpuset.push_back('/');
                }
                writepid_files_.push_back(
                    StringPrintf("/dev/cpuset%stasks", default_cpuset.c_str()));
            }
        }
        std::string pid_str = StringPrintf("%d", getpid());
        for (const auto& file : writepid_files_) {
            if (!WriteStringToFile(pid_str, file)) {
                PLOG(ERROR) << "couldn't write " << pid_str << " to " << file;
            }
        }

        if (ioprio_class_ != IoSchedClass_NONE) {
            if (android_set_ioprio(getpid(), ioprio_class_, ioprio_pri_)) {
                PLOG(ERROR) << "failed to set pid " << getpid()
                            << " ioprio=" << ioprio_class_ << "," << ioprio_pri_;
            }
        }

        if (needs_console) {
            setsid();
            OpenConsole();
        } else {
            ZapStdio();
        }

        // As requested, set our gid, supplemental gids, uid, context, and
        // priority. Aborts on failure.
        SetProcessAttributes();

        std::vector<char*> strs;
        ExpandArgs(args_, &strs);
        if (execve(strs[0], (char**) &strs[0], (char**) ENV) < 0) {
            PLOG(ERROR) << "cannot execve('" << strs[0] << "')";
        }

        _exit(127);
    }

    if (pid < 0) {
        PLOG(ERROR) << "failed to fork for '" << name_ << "'";
        pid_ = 0;
        return false;
    }

    if (oom_score_adjust_ != -1000) {
        std::string oom_str = StringPrintf("%d", oom_score_adjust_);
        std::string oom_file = StringPrintf("/proc/%d/oom_score_adj", pid);
        if (!WriteStringToFile(oom_str, oom_file)) {
            PLOG(ERROR) << "couldn't write oom_score_adj: " << strerror(errno);
        }
    }

    time_started_ = boot_clock::now();
    pid_ = pid;
    flags_ |= SVC_RUNNING;

    errno = -createProcessGroup(uid_, pid_);
    if (errno != 0) {
        PLOG(ERROR) << "createProcessGroup(" << uid_ << ", " << pid_ << ") failed for service '"
                    << name_ << "'";
    }

    if ((flags_ & SVC_EXEC) != 0) {
        LOG(INFO) << android::base::StringPrintf(
            "SVC_EXEC pid %d (uid %d gid %d+%zu context %s) started; waiting...", pid_, uid_, gid_,
            supp_gids_.size(), !seclabel_.empty() ? seclabel_.c_str() : "default");
    }

    NotifyStateChange("running");
    return true;
}
```

### 服务启动流程

> frameworks/base/cmds/app_process/app_main.cpp

```cpp
int main(int argc, char* const argv[])
{
    if (!LOG_NDEBUG) {
      String8 argv_String;
      for (int i = 0; i < argc; ++i) {
        argv_String.append("\"");
        argv_String.append(argv[i]);
        argv_String.append("\" ");
      }
      ALOGV("app_process main with argv: %s", argv_String.string());
    }

    AppRuntime runtime(argv[0], computeArgBlockSize(argc, argv));
    // Process command line arguments
    // ignore argv[0]
    argc--;
    argv++;

    // Everything up to '--' or first non '-' arg goes to the vm.
    //
    // The first argument after the VM args is the "parent dir", which
    // is currently unused.
    //
    // After the parent dir, we expect one or more the following internal
    // arguments :
    //
    // --zygote : Start in zygote mode
    // --start-system-server : Start the system server.
    // --application : Start in application (stand alone, non zygote) mode.
    // --nice-name : The nice name for this process.
    //
    // For non zygote starts, these arguments will be followed by
    // the main class name. All remaining arguments are passed to
    // the main method of this class.
    //
    // For zygote starts, all remaining arguments are passed to the zygote.
    // main function.
    //
    // Note that we must copy argument string values since we will rewrite the
    // entire argument block when we apply the nice name to argv0.
    //
    // As an exception to the above rule, anything in "spaced commands"
    // goes to the vm even though it has a space in it.
    const char* spaced_commands[] = { "-cp", "-classpath" };
    // Allow "spaced commands" to be succeeded by exactly 1 argument (regardless of -s).
    bool known_command = false;

    int i;
    for (i = 0; i < argc; i++) {
        if (known_command == true) {
          runtime.addOption(strdup(argv[i]));
          ALOGV("app_process main add known option '%s'", argv[i]);
          known_command = false;
          continue;
        }

        for (int j = 0;
             j < static_cast<int>(sizeof(spaced_commands) / sizeof(spaced_commands[0]));
             ++j) {
          if (strcmp(argv[i], spaced_commands[j]) == 0) {
            known_command = true;
            ALOGV("app_process main found known command '%s'", argv[i]);
          }
        }

        if (argv[i][0] != '-') {
            break;
        }
        if (argv[i][1] == '-' && argv[i][2] == 0) {
            ++i; // Skip --.
            break;
        }

        runtime.addOption(strdup(argv[i]));
        ALOGV("app_process main add option '%s'", argv[i]);
    }

    // Parse runtime arguments.  Stop at first unrecognized option.
    bool zygote = false;
    bool startSystemServer = false;
    bool application = false;
    String8 niceName;
    String8 className;

    ++i;  // Skip unused "parent dir" argument.
    while (i < argc) {
        const char* arg = argv[i++];
        if (strcmp(arg, "--zygote") == 0) {
            zygote = true;
            niceName = ZYGOTE_NICE_NAME;
        } else if (strcmp(arg, "--start-system-server") == 0) {
            startSystemServer = true;
        } else if (strcmp(arg, "--application") == 0) {
            application = true;
        } else if (strncmp(arg, "--nice-name=", 12) == 0) {
            niceName.setTo(arg + 12);
        } else if (strncmp(arg, "--", 2) != 0) {
            className.setTo(arg);
            break;
        } else {
            --i;
            break;
        }
    }

    Vector<String8> args;
    if (!className.isEmpty()) {
        // We're not in zygote mode, the only argument we need to pass
        // to RuntimeInit is the application argument.
        //
        // The Remainder of args get passed to startup class main(). Make
        // copies of them before we overwrite them with the process name.
        args.add(application ? String8("application") : String8("tool"));
        runtime.setClassNameAndArgs(className, argc - i, argv + i);

        if (!LOG_NDEBUG) {
          String8 restOfArgs;
          char* const* argv_new = argv + i;
          int argc_new = argc - i;
          for (int k = 0; k < argc_new; ++k) {
            restOfArgs.append("\"");
            restOfArgs.append(argv_new[k]);
            restOfArgs.append("\" ");
          }
          ALOGV("Class name = %s, args = %s", className.string(), restOfArgs.string());
        }
    } else {
        // We're in zygote mode.
        maybeCreateDalvikCache();

        if (startSystemServer) {
            args.add(String8("start-system-server"));
        }

        char prop[PROP_VALUE_MAX];
        if (property_get(ABI_LIST_PROPERTY, prop, NULL) == 0) {
            LOG_ALWAYS_FATAL("app_process: Unable to determine ABI list from property %s.",
                ABI_LIST_PROPERTY);
            return 11;
        }

        String8 abiFlag("--abi-list=");
        abiFlag.append(prop);
        args.add(abiFlag);

        // In zygote mode, pass all remaining arguments to the zygote
        // main() method.
        for (; i < argc; ++i) {
            args.add(String8(argv[i]));
        }
    }

    if (!niceName.isEmpty()) {
        runtime.setArgv0(niceName.string(), true /* setProcName */);
    }

    if (zygote) {
        // AppRuntime启动ZygoteInit
        runtime.start("com.android.internal.os.ZygoteInit", args, zygote);
    } else if (className) {
        runtime.start("com.android.internal.os.RuntimeInit", args, zygote);
    } else {
        fprintf(stderr, "Error: no class name or --zygote supplied.\n");
        app_usage();
        LOG_ALWAYS_FATAL("app_process: no class name or --zygote supplied.");
    }
}
```

> frameworks/base/core/jni/AndroidRuntime.cpp

```cpp
/*
 * Start the Android runtime.  This involves starting the virtual machine
 * and calling the "static void main(String[] args)" method in the class
 * named by "className".
 *
 * Passes the main function two arguments, the class name and the specified
 * options string.
 */
void AndroidRuntime::start(const char* className, const Vector<String8>& options, bool zygote)
{
    ALOGD(">>>>>> START %s uid %d <<<<<<\n",
            className != NULL ? className : "(unknown)", getuid());

    static const String8 startSystemServer("start-system-server");

    /*
     * 'startSystemServer == true' means runtime is obsolete and not run from
     * init.rc anymore, so we print out the boot start event here.
     */
    for (size_t i = 0; i < options.size(); ++i) {
        if (options[i] == startSystemServer) {
           /* track our progress through the boot sequence */
           const int LOG_BOOT_PROGRESS_START = 3000;
           LOG_EVENT_LONG(LOG_BOOT_PROGRESS_START,  ns2ms(systemTime(SYSTEM_TIME_MONOTONIC)));
        }
    }

    const char* rootDir = getenv("ANDROID_ROOT");
    if (rootDir == NULL) {
        rootDir = "/system";
        if (!hasDir("/system")) {
            LOG_FATAL("No root directory specified, and /android does not exist.");
            return;
        }
        setenv("ANDROID_ROOT", rootDir, 1);
    }

    //const char* kernelHack = getenv("LD_ASSUME_KERNEL");
    //ALOGD("Found LD_ASSUME_KERNEL='%s'\n", kernelHack);

    /* start the virtual machine */
    JniInvocation jni_invocation;
    jni_invocation.Init(NULL);
    JNIEnv* env;
    if (startVm(&mJavaVM, &env, zygote) != 0) {
        return;
    }
    onVmCreated(env);

    /*
     * Register android functions.
     */
    if (startReg(env) < 0) {
        ALOGE("Unable to register all android natives\n");
        return;
    }

    /*
     * We want to call main() with a String array with arguments in it.
     * At present we have two arguments, the class name and an option string.
     * Create an array to hold them.
     */
    jclass stringClass;
    jobjectArray strArray;
    jstring classNameStr;

    stringClass = env->FindClass("java/lang/String");
    assert(stringClass != NULL);
    strArray = env->NewObjectArray(options.size() + 1, stringClass, NULL);
    assert(strArray != NULL);
    classNameStr = env->NewStringUTF(className);
    assert(classNameStr != NULL);
    env->SetObjectArrayElement(strArray, 0, classNameStr);

    for (size_t i = 0; i < options.size(); ++i) {
        jstring optionsStr = env->NewStringUTF(options.itemAt(i).string());
        assert(optionsStr != NULL);
        env->SetObjectArrayElement(strArray, i + 1, optionsStr);
    }

    /*
     * Start VM.  This thread becomes the main thread of the VM, and will
     * not return until the VM exits.
     */
    char* slashClassName = toSlashClassName(className);
    jclass startClass = env->FindClass(slashClassName);
    if (startClass == NULL) {
        ALOGE("JavaVM unable to locate class '%s'\n", slashClassName);
        /* keep going */
    } else {
        jmethodID startMeth = env->GetStaticMethodID(startClass, "main",
            "([Ljava/lang/String;)V");
        if (startMeth == NULL) {
            ALOGE("JavaVM unable to find main() in '%s'\n", className);
            /* keep going */
        } else {
            env->CallStaticVoidMethod(startClass, startMeth, strArray);

#if 0
            if (env->ExceptionCheck())
                threadExitUncaughtException(env);
#endif
        }
    }
    free(slashClassName);

    ALOGD("Shutting down VM\n");
    if (mJavaVM->DetachCurrentThread() != JNI_OK)
        ALOGW("Warning: unable to detach main thread\n");
    if (mJavaVM->DestroyJavaVM() != 0)
        ALOGW("Warning: VM did not shut down cleanly\n");
}
```



## 属性

### 属性入口

功能是初始化属性区域的初始化，实际工作由 **__system_property_area_init()** 完成

> system/core/init/property_service.cpp

```cpp
void property_init() {
    if (__system_property_area_init()) {
        LOG(ERROR) << "Failed to initialize property area";
        exit(1);
    }
}
```

### 初始化属性空间

> bionic/libc/bionic/system_properties.cpp

```cpp
int __system_property_area_init() {
  free_and_unmap_contexts();
  mkdir(property_filename, S_IRWXU | S_IXGRP | S_IXOTH);
  if (!initialize_properties()) {
    return -1;
  }
  bool open_failed = false;
  bool fsetxattr_failed = false;
  list_foreach(contexts, [&fsetxattr_failed, &open_failed](context_node* l) {
    if (!l->open(true, &fsetxattr_failed)) {
      open_failed = true;
    }
  });
  if (open_failed || !map_system_property_area(true, &fsetxattr_failed)) {
    free_and_unmap_contexts();
    return -1;
  }
  initialized = true;
  ret
```

### 初始化属性值

> system/core/init/property_service.cpp

```cpp
void start_property_service() {
    // 设置具体 ro.property_service.version 属性
    property_set("ro.property_service.version", "2");

    // 创建属性服务的非阻塞Socket服务
    property_set_fd = create_socket(PROP_SERVICE_NAME, SOCK_STREAM | SOCK_CLOEXEC | SOCK_NONBLOCK,
                                    0666, 0, 0, NULL);
    if (property_set_fd == -1) {
        PLOG(ERROR) << "start_property_service socket creation failed";
        exit(1);
    }

    // 开始监听属性服务的Socket，最大8个客户端同时连接
    listen(property_set_fd, 8);
    // epoll监听property_set_fd，用handle_property_set_fd函数处理调用
    register_epoll_handler(property_set_fd, handle_property_set_fd);
}
```

### 处理属性值

> system/core/init/property_service.cpp

```cpp
static void handle_property_set_fd() {
    static constexpr uint32_t kDefaultSocketTimeout = 2000; /* ms */

    int s = accept4(property_set_fd, nullptr, nullptr, SOCK_CLOEXEC);
    if (s == -1) {
        return;
    }

    struct ucred cr;
    socklen_t cr_size = sizeof(cr);
    if (getsockopt(s, SOL_SOCKET, SO_PEERCRED, &cr, &cr_size) < 0) {
        close(s);
        PLOG(ERROR) << "sys_prop: unable to get SO_PEERCRED";
        return;
    }

    SocketConnection socket(s, cr);
    uint32_t timeout_ms = kDefaultSocketTimeout;

    uint32_t cmd = 0;
    if (!socket.RecvUint32(&cmd, &timeout_ms)) {
        PLOG(ERROR) << "sys_prop: error while reading command from the socket";
        socket.SendUint32(PROP_ERROR_READ_CMD);
        return;
    }

    switch (cmd) {
    case PROP_MSG_SETPROP: {
        char prop_name[PROP_NAME_MAX];
        char prop_value[PROP_VALUE_MAX];

        // 无法读取属性键或属性值，直接返回
        if (!socket.RecvChars(prop_name, PROP_NAME_MAX, &timeout_ms) ||
            !socket.RecvChars(prop_value, PROP_VALUE_MAX, &timeout_ms)) {
          PLOG(ERROR) << "sys_prop(PROP_MSG_SETPROP): error while reading name/value from the socket";
          return;
        }

        prop_name[PROP_NAME_MAX-1] = 0;
        prop_value[PROP_VALUE_MAX-1] = 0;

        handle_property_set(socket, prop_value, prop_value, true);
        break;
      }

    case PROP_MSG_SETPROP2: {
        std::string name;
        std::string value;
        if (!socket.RecvString(&name, &timeout_ms) ||
            !socket.RecvString(&value, &timeout_ms)) {
          PLOG(ERROR) << "sys_prop(PROP_MSG_SETPROP2): error while reading name/value from the socket";
          socket.SendUint32(PROP_ERROR_READ_DATA);
          return;
        }

        handle_property_set(socket, name, value, false);
        break;
      }

    default:
        LOG(ERROR) << "sys_prop: invalid command " << cmd;
        socket.SendUint32(PROP_ERROR_INVALID_CMD);
        break;
    }
}
```

### 设置属性值

> system/core/init/property_service.cpp

```cpp
static void handle_property_set(SocketConnection& socket,
                                const std::string& name,
                                const std::string& value,
                                bool legacy_protocol) {
  const char* cmd_name = legacy_protocol ? "PROP_MSG_SETPROP" : "PROP_MSG_SETPROP2";
  if (!is_legal_property_name(name)) {
    LOG(ERROR) << "sys_prop(" << cmd_name << "): illegal property name \"" << name << "\"";
    socket.SendUint32(PROP_ERROR_INVALID_NAME);
    return;
  }

  struct ucred cr = socket.cred();
  char* source_ctx = nullptr;
  getpeercon(socket.socket(), &source_ctx);
    
  // "ctl."开头的控制属性
  if (android::base::StartsWith(name, "ctl.")) {
    if (check_control_mac_perms(value.c_str(), source_ctx, &cr)) {
      handle_control_message(name.c_str() + 4, value.c_str());
      if (!legacy_protocol) {
        socket.SendUint32(PROP_SUCCESS);
      }
    } else {
      LOG(ERROR) << "sys_prop(" << cmd_name << "): Unable to " << (name.c_str() + 4)
                 << " service ctl [" << value << "]"
                 << " uid:" << cr.uid
                 << " gid:" << cr.gid
                 << " pid:" << cr.pid;
      if (!legacy_protocol) {
        socket.SendUint32(PROP_ERROR_HANDLE_CONTROL_MESSAGE);
      }
    }
  } else {
    if (check_mac_perms(name, source_ctx, &cr)) {
      uint32_t result = property_set(name, value);
      if (!legacy_protocol) {
        socket.SendUint32(result);
      }
    } else {
      LOG(ERROR) << "sys_prop(" << cmd_name << "): permission denied uid:" << cr.uid << " name:" << name;
      if (!legacy_protocol) {
        socket.SendUint32(PROP_ERROR_PERMISSION_DENIED);
      }
    }
  }

  freecon(source_ctx);
}
```

```cpp
uint32_t property_set(const std::string& name, const std::string& value) {
    size_t valuelen = value.size();

    if (!is_legal_property_name(name)) {
        LOG(ERROR) << "property_set(\"" << name << "\", \"" << value << "\") failed: bad name";
        return PROP_ERROR_INVALID_NAME;
    }

    if (valuelen >= PROP_VALUE_MAX) {
        LOG(ERROR) << "property_set(\"" << name << "\", \"" << value << "\") failed: "
                   << "value too long";
        return PROP_ERROR_INVALID_VALUE;
    }

    if (name == "selinux.restorecon_recursive" && valuelen > 0) {
        if (restorecon(value.c_str(), SELINUX_ANDROID_RESTORECON_RECURSE) != 0) {
            LOG(ERROR) << "Failed to restorecon_recursive " << value;
        }
    }

    prop_info* pi = (prop_info*) __system_property_find(name.c_str());
    if (pi != nullptr) {
        // ro.* properties are actually "write-once".
        if (android::base::StartsWith(name, "ro.")) {
            LOG(ERROR) << "property_set(\"" << name << "\", \"" << value << "\") failed: "
                       << "property already set";
            return PROP_ERROR_READ_ONLY_PROPERTY;
        }

        __system_property_update(pi, value.c_str(), valuelen);
    } else {
        int rc = __system_property_add(name.c_str(), name.size(), value.c_str(), valuelen);
        if (rc < 0) {
            LOG(ERROR) << "property_set(\"" << name << "\", \"" << value << "\") failed: "
                       << "__system_property_add failed";
            return PROP_ERROR_SET_FAILED;
        }
    }

    // Don't write properties to disk until after we have read all default
    // properties to prevent them from being overwritten by default values.
    if (persistent_properties_loaded && android::base::StartsWith(name, "persist.")) {
        write_persistent_property(name.c_str(), value.c_str());
    }
    property_changed(name, value);
    return PROP_SUCCESS;
}
```

