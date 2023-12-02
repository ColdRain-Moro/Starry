# 阶段3 lab3 实验报告

## 思考题

### 1.1

registry/src/index.crates.io-6f17d22bba15001f/log-0.4.20/

### 1.2

如果已经修改了这些代码不clean的话 下次build就会使用修改过的代码编译

### 3.1

不会，因为有些系统调用执行到中途进程就挂起了

### 3.2

同上，需要考虑sys_yield这类的syscall

## 3.1

先把有问题的shell语句的 syscall 拎出来分析。

```rust
/// -- busybox mv abc bin/
[100.455317 0:6 syscall_entry::syscall:37] [syscall] id = SET_TID_ADDRESS, args = [1457172, 248, 0, 18446744073708088664, 1, 1], entry
[100.470836 0:6 syscall_entry::syscall:46] [syscall] id = 96, args = [1457172, 248, 0, 18446744073708088664, 1, 1], return 6
[100.489472 0:6 syscall_entry::syscall:37] [syscall] id = GETUID, args = [1073741394, 47, 1073741394, 98, 1073741394, 1073741394], entry
[100.508118 0:6 syscall_entry::syscall:46] [syscall] id = 174, args = [1073741394, 47, 1073741394, 98, 1073741394, 1073741394], return 0
[100.539582 0:6 syscall_entry::syscall:28] [syscall] id = FSTATAT, args = [18446744073709551516, 1073741382, 1073740160, 0, 1073741382, 18446744073709551516], entry
[100.593688 0:6 fatfs::dir:140] Is a directory
[100.631215 0:6 syscall_entry::syscall:46] [syscall] id = 79, args = [18446744073709551516, 1073741382, 1073740160, 0, 1073741382, 18446744073709551516], return 0
[100.663661 0:6 syscall_entry::syscall:20] [syscall] id = BRK, args = [0, 64, 1458528, 0, 1457024, 4096], entry
[100.679831 0:6 syscall_entry::syscall:46] [syscall] id = 214, args = [0, 64, 1458528, 0, 1457024, 4096], return 1067450368
[100.696064 0:6 syscall_entry::syscall:20] [syscall] id = MMAP, args = [0, 4096, 3, 34, 18446744073709551615, 0], entry
[100.721234 0:6 syscall_entry::syscall:46] [syscall] id = 222, args = [0, 4096, 3, 34, 18446744073709551615, 0], return 4096
[100.751666 0:6 syscall_entry::syscall:28] [syscall] id = FSTATAT, args = [18446744073709551516, 4128, 1073740160, 0, 4128, 18446744073709551516], entry
[100.877189 0:6 syscall_entry::syscall:46] [syscall] id = 79, args = [18446744073709551516, 4128, 1073740160, 0, 4128, 18446744073709551516], return -64
[100.903980 0:6 syscall_entry::syscall:28] [syscall] id = WRITE, args = [2, 1073740112, 47, 0, 0, 0], entry
mv: can't stat 'bin/abc': No error information
[100.926199 0:6 syscall_entry::syscall:46] [syscall] id = 64, args = [2, 1073740112, 47, 0, 0, 0], return 47
[100.946685 0:6 syscall_entry::syscall:37] [syscall] id = EXIT_GROUP, args = [1, 0, 0, 1459576, 0, 1], entry
/// -- busybox mv abc bin/
```

利用 strace 简单对拍一下

```rust
ubuntu@VM-16-15-ubuntu:~/Starry/testcases/sdcard$ strace busybox mv abc bin/
execve("/usr/bin/busybox", ["busybox", "mv", "abc", "bin/"], 0x7ffd9edd53f8 /* 31 vars */) = 0
arch_prctl(0x3001 /* ARCH_??? */, 0x7ffd014850c0) = -1 EINVAL (Invalid argument)
brk(NULL)                               = 0x810000
brk(0x810dc0)                           = 0x810dc0
arch_prctl(ARCH_SET_FS, 0x8103c0)       = 0
set_tid_address(0x810690)               = 2032618
set_robust_list(0x8106a0, 24)           = 0
rseq(0x810d60, 0x20, 0, 0x53053053)     = 0
uname({sysname="Linux", nodename="VM-16-15-ubuntu", ...}) = 0
prlimit64(0, RLIMIT_STACK, NULL, {rlim_cur=8192*1024, rlim_max=RLIM64_INFINITY}) = 0
readlink("/proc/self/exe", "/usr/bin/busybox", 4096) = 16
getrandom("\x15\x7b\x47\x2f\xda\x26\x32\x9a", 8, GRND_NONBLOCK) = 8
brk(0x831dc0)                           = 0x831dc0
brk(0x832000)                           = 0x832000
mprotect(0x60c000, 28672, PROT_READ)    = 0
prctl(PR_SET_NAME, "busybox")           = 0
getuid()                                = 1000
newfstatat(AT_FDCWD, "/etc/busybox.conf", 0x7ffd01484d78, 0) = -1 ENOENT (No such file or directory)
getgid()                                = 1000
setgid(1000)                            = 0
setuid(1000)                            = 0
newfstatat(AT_FDCWD, "bin/", {st_mode=S_IFDIR|0775, st_size=4096, ...}, 0) = 0
newfstatat(AT_FDCWD, "bin/abc", 0x7ffd01484d18, 0) = -1 ENOENT (No such file or directory)
rename("abc", "bin/abc")                = 0
exit_group(0)                           = ?
+++ exited with 0 +++
```

发现 starry 的输出并没有进行 rename 操作，仔细一看 `mv: can't stat 'bin/abc'` 。我们对一下  strace 的输出

```rust
newfstatat(AT_FDCWD, "bin/", {st_mode=S_IFDIR|0775, st_size=4096, ...}, 0) = 0
newfstatat(AT_FDCWD, "bin/abc", 0x7ffd01484d18, 0) = -1 ENOENT (No such file or directory)
```

根据 mv 的功能，推测这里 mv 先查看了 bin/ 的文件类型，发现是一个文件夹，然后查询 bin/abc 是否存在，如果不存在的话就直接在 bin/abc 创建。

对拍时发现 starry 的 fstatat 返回值是 -64（ENONET），很奇怪，查了一下发现代表的含义是设备未联网，怎么想都不可能是因为没有联网导致的问题。然后对了一下 strace 的返回值，猜测应该是把 `ENOENT` 写错了：

![Untitled](https://persecution-1301196908.cos.ap-chongqing.myqcloud.com/image_bed/Untitled.png)

改回 `ENOENT` 即可，然后压力就来到了 rename 这边

```rust
mv: can't rename 'abc': Operation not permitted
```

提示已经给得非常到位了，直接照着写即可。只为通过测例的话，加一行就可以结束战斗

```rust
rename(old_path.path(), new_path.path()).unwrap();
```

## 3.2

先对个拍

```rust
[ 57.395849 0:6 syscall_entry::syscall:37] [syscall] id = SET_TID_ADDRESS, args = [1457172, 248, 0, 18446744073708088664, 1, 1], entry
[ 57.421868 0:6 syscall_entry::syscall:46] [syscall] id = 96, args = [1457172, 248, 0, 18446744073708088664, 1, 1], return 6
[ 57.443425 0:6 syscall_entry::syscall:37] [syscall] id = GETUID, args = [1073741394, 47, 1073741394, 98, 1073741394, 1073741394], entry
[ 57.460874 0:6 syscall_entry::syscall:46] [syscall] id = 174, args = [1073741394, 47, 1073741394, 98, 1073741394, 1073741394], return 0
[ 57.487189 0:6 syscall_entry::syscall:28] [syscall] id = UTIMENSAT, args = [18446744073709551516, 1073741384, 0, 0, 0, 0], entry
[ 59.444953 0:6 syscall_entry::syscall:46] [syscall] id = 88, args = [18446744073709551516, 1073741384, 0, 0, 0, 0], return 0
[ 59.461913 0:6 syscall_entry::syscall:37] [syscall] id = EXIT_GROUP, args = [0, 0, 0, 0, 0, 0], entry
[ 77.106936 0:6 syscall_entry::syscall:37] [syscall] id = SET_TID_ADDRESS, args = [1457172, 248, 0, 18446744073708088664, 1, 1], entry
[ 77.122957 0:6 syscall_entry::syscall:46] [syscall] id = 96, args = [1457172, 248, 0, 18446744073708088664, 1, 1], return 6
[ 77.141284 0:6 syscall_entry::syscall:37] [syscall] id = GETUID, args = [1073741394, 47, 1073741394, 98, 1073741394, 1073741394], entry
[ 77.156505 0:6 syscall_entry::syscall:46] [syscall] id = 174, args = [1073741394, 47, 1073741394, 98, 1073741394, 1073741394], return 0
[ 77.183134 0:6 syscall_entry::syscall:28] [syscall] id = FSTATAT, args = [18446744073709551516, 1073741383, 1073740160, 0, 1073741383, 18446744073709551516], entry
[ 77.226453 0:6 fatfs::dir:140] Is a directory
[ 77.259945 0:6 syscall_entry::syscall:46] [syscall] id = 79, args = [18446744073709551516, 1073741383, 1073740160, 0, 1073741383, 18446744073709551516], return 0
[ 77.288514 0:6 syscall_entry::syscall:28] [syscall] id = FACCESSAT, args = [18446744073709551516, 1073741383, 2, 0, 16384, 1073741383], entry
[ 77.329071 0:6 fatfs::dir:140] Is a directory
[ 77.358741 0:6 syscall_entry::syscall:46] [syscall] id = 48, args = [18446744073709551516, 1073741383, 2, 0, 16384, 1073741383], return 0
[ 77.375370 0:6 syscall_entry::syscall:28] [syscall] id = RENAMEAT2, args = [18446744073709551516, 1073741387, 18446744073709551516, 1073741383, 0, 1073741387], entry
[ 77.421839 0:6 fatfs::dir:140] Is a directory
[ 77.450974 0:6 axfs::root:320] dst file already exist, now remove it
[ 77.484255 0:6 fatfs::dir:140] Is a directory
[ 77.511693 0:6 axfs::root:253] [AxError::IsADirectory]
[ 77.524079 0:6 axruntime::lang_items:5] panicked at ulib/axstarry/syscall_fs/src/imp/ctl.rs:260:46:
called `Result::unwrap()` on an `Err` value: IsADirectory
```

```rust
ubuntu@VM-16-15-ubuntu:~/Starry/testcases/sdcard$ strace busybox mv def bin
execve("/usr/bin/busybox", ["busybox", "mv", "def", "bin"], 0x7ffcf708cea8 /* 31 vars */) = 0
arch_prctl(0x3001 /* ARCH_??? */, 0x7ffd6abfbed0) = -1 EINVAL (Invalid argument)
brk(NULL)                               = 0x18ca000
brk(0x18cadc0)                          = 0x18cadc0
arch_prctl(ARCH_SET_FS, 0x18ca3c0)      = 0
set_tid_address(0x18ca690)              = 2161885
set_robust_list(0x18ca6a0, 24)          = 0
rseq(0x18cad60, 0x20, 0, 0x53053053)    = 0
uname({sysname="Linux", nodename="VM-16-15-ubuntu", ...}) = 0
prlimit64(0, RLIMIT_STACK, NULL, {rlim_cur=8192*1024, rlim_max=RLIM64_INFINITY}) = 0
readlink("/proc/self/exe", "/usr/bin/busybox", 4096) = 16
getrandom("\xe6\x2d\x88\xd5\x46\xaa\xd5\xaa", 8, GRND_NONBLOCK) = 8
brk(0x18ebdc0)                          = 0x18ebdc0
brk(0x18ec000)                          = 0x18ec000
mprotect(0x60c000, 28672, PROT_READ)    = 0
prctl(PR_SET_NAME, "busybox")           = 0
getuid()                                = 1000
newfstatat(AT_FDCWD, "/etc/busybox.conf", 0x7ffd6abfbb88, 0) = -1 ENOENT (No such file or directory)
getgid()                                = 1000
setgid(1000)                            = 0
setuid(1000)                            = 0
newfstatat(AT_FDCWD, "bin", {st_mode=S_IFDIR|0775, st_size=4096, ...}, 0) = 0
newfstatat(AT_FDCWD, "bin/def", 0x7ffd6abfbb28, 0) = -1 ENOENT (No such file or directory)
rename("def", "bin/def")                = 0
exit_group(0)                           = ?
+++ exited with 0 +++
```

可以锁定是 fstatat 的问题，没有能区分文件和文件夹。发现 starry 只针对路径的写法做了简单的区分：

![Untitled 1](https://persecution-1301196908.cos.ap-chongqing.myqcloud.com/image_bed/Untitled%201.png)

稍微修改一下这里的实现即可，写法取了点巧，比指导书说的还少加两行（

![Untitled 2](https://persecution-1301196908.cos.ap-chongqing.myqcloud.com/image_bed/Untitled%202.png)
