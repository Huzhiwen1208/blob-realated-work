# 运行Libos形态的zCore
```shell
git clone git@github.com:elliott10/zCore.git --recursive
make rootfs
cd zCore
# 这个命令和下面那个命令等同
cargo run --release --features "linux libos" -- /bin/busybox [args]
# make run LIBOS=1 LINUX=1 LOG=warn
```

# 用户态libOS模式的设计实现分析
## 源码分析
```rust
zCore/zCore/platform/libos.rs:main
    zCore/zCore/main.rs:primary_main
        log初始化、mem初始化、kernel_hal:primary_init_early(kernel_hal/src/libos/boot.rs)
            device.add(uart),注册监听事件：不断读取数据至UART中并等待中断处理。
        构建BootOptions{log_level: warn, root_proc: /bin/busybox}
        页帧分配器：kernel_hal: [4K----1GB)
        kernel_hal: primary_init()  空实现
        STARTED = true
        args = /bin/busybox, envs = "PATH=/usr/sbin:/usr/bin:/sbin:/bin"
        zcore_loader::linux::run(args, envs, rootfs)
```
在zCore启动过程中，会初始化物理页帧分配器、堆分配器、线程调度器等各个组成部分。并委托 zircon-­loader 进行内核对象的初始化创建过程，然后进入用户态的启动过程开始执行。每当用户态触发系统调用进入内核态，系统调用处理例程将会通过已实现的内核对象的功能来对服务请求进行处理；而对应的内核对象的内部实现所需要的各种底层操作，则是通过 HAL 层接口由各个内核组件负责提供。其中，VDSO（Virtual dynamic shared object）是一个映射到用户空间的 so 文件，可以在不陷入内核的情况下执行一些简单的系统调用。在设计中，所有中断都需要经过 VDSO 拦截进行处理，因此重写 VDSO 便可以实现自定义的对下层系统调用（syscall）的支持。Executor 是 zCore 中基于 Rust 的 async 机制的协程调度器。在HAL接口层的设计上，还借助了 Rust 的能够指定函数链接过程的特性。即，在 kernel-­hal 中规定了所有可供 zircon­-object 库及 zircon-­syscall 库调用的虚拟硬件接口，以函数 API 的形式给出，但是内部均为未实现状态，并设置函数为弱引用链接状态。在 kernel­-hal 中才给出裸机环境下的硬件接口具体实现（libos或者bare），编译 zCore 项目时、链接的过程中将会替换/覆盖 kernel-­hal 中未实现的同名接口，从而达到能够在编译时灵活选择 HAL 层的效果。其能得到的相关结论：
1. 根据zCore/zCore/main.rs中的#![cfg_attr(not(feature = "libos"), no_std)]我们可以看到，libos版的zCore是依赖于宿主机操作系统提供的标准库的。
2. HAL层提供了一个统一的弱链接实现的空接口集，由libos实现，其实现依赖于宿主机操作系统。如kernel-hal/src/libos/timer.rs中的timer_now方法。
3. libos和bare模式的实现是通过条件编译控制的，判断feature=libos即可控制，除了HAL层代码和入口代码，其他所有功能代码均与bare模式共用，实现代码复用。
4. libos的入口并不是_start，而是main()函数，原因是libos跑在用户态，只需提供main入口即可。

## TODO
1. HAL层的内存分配管理，现阶段观察到虚存从0x800000000开始映射。
2. HAL层的其它接口实现，如mmap, 看起来源码中libos没有实现这些？