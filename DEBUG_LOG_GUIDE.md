# 调试日志收集指南

## 如何收集日志

编译并运行你的应用后，使用以下命令收集完整日志：

```bash
# 方法1: 使用 logcat 过滤 GUM 相关日志
adb logcat -s "System.out:I" | grep "\[GUM\]"

# 方法2: 收集所有日志到文件
adb logcat > /tmp/gum_debug.log

# 然后过滤 GUM 日志
grep "\[GUM\]" /tmp/gum_debug.log
```

## 关键日志信息

日志会显示以下关键信息：

### 1. Stalker 初始化
```
[GUM] Stalker init: Android detected, forcing is_rwx_supported=FALSE
```
- 这应该在 Android 上显示

### 2. 内存分配
```
[GUM] Allocating exec ctx: size=XXX, prot=RW (is_rwx_supported=0)
[GUM] Allocated exec ctx at: 0xXXXXXXXX
```
- 应该显示 `prot=RW` (而不是 RWX)
- 应该显示 `is_rwx_supported=0` (FALSE)

```
[GUM] Allocating code slab: size=XXX, prot=RW (is_rwx_supported=0)
```
- 同样应该是 RW

### 3. 权限切换（thaw/freeze）
```
[GUM] Stalker thaw: code=0xXXXXXXXX, size=XXX, setting RW
[GUM] mprotect: addr=0xXXXXXXXX, size=XXX, aligned_addr=0xXXXXXXXX, aligned_size=XXX, prot=RW
[GUM] mprotect: existing prot=RW, actual_size=XXX
[GUM] mprotect SUCCESS
```

```
[GUM] Stalker freeze: code=0xXXXXXXXX, size=XXX, setting RX
[GUM] mark_code: addr=0xXXXXXXXX, size=XXX
[GUM] mark_code: using mprotect RX
[GUM] mprotect: addr=0xXXXXXXXX, size=XXX, aligned_addr=0xXXXXXXXX, aligned_size=XXX, prot=RX
[GUM] mprotect SUCCESS
[GUM] mark_code: success=1
```

### 4. 如果出现错误
```
[GUM] mprotect FAILED: errno=XX (error message)
```

## 重要：查找崩溃地址

你的错误信息：
```
Fatal signal 11 (SIGSEGV), code 1 (SEGV_MAPERR), fault addr 0x7b64cfc000
```

在日志中搜索这个地址 `0x7b64cfc000`，看看：
1. 这个地址是否被分配过？
2. 这个地址的权限是什么？
3. 在崩溃前最后一次操作这个地址的是什么？

## 完整的日志收集命令

```bash
# 清除之前的日志
adb logcat -c

# 启动日志收集
adb logcat -v time System.out:I *:S > gum_debug_$(date +%Y%m%d_%H%M%S).log &

# 运行你的应用
# (启动你的 app 并触发 Stalker)

# 等待崩溃发生后，停止日志收集
kill %1

# 查看 GUM 相关日志
grep "\[GUM\]" gum_debug_*.log
```

## 检查内存映射

在崩溃前或崩溃后，也可以检查进程的内存映射：

```bash
# 获取进程 PID
adb shell "ps | grep com.litatom.app"

# 查看内存映射
adb shell "cat /proc/[PID]/maps"

# 查找崩溃地址附近的映射
adb shell "cat /proc/[PID]/maps | grep -A 5 -B 5 7b64cfc"
```

## 需要发给我的信息

请提供：

1. **完整的 GUM 日志**（从启动到崩溃）
   ```bash
   grep "\[GUM\]" gum_debug.log
   ```

2. **崩溃时的完整 logcat**（包括堆栈跟踪）
   ```bash
   adb logcat -d > crash_log.txt
   ```

3. **内存映射信息**（如果能获取到）
   ```bash
   adb shell "cat /proc/[PID]/maps" > maps.txt
   ```

4. **特别注意**：找出崩溃地址 `0x7b64cfc000` 相关的日志

## 快速检查脚本

我创建了一个快速检查脚本：

```bash
#!/bin/bash
# save as check_gum_logs.sh

echo "=== 检查 Stalker 初始化 ==="
grep "Stalker init" gum_debug.log

echo -e "\n=== 检查内存分配 ==="
grep "Allocating" gum_debug.log

echo -e "\n=== 检查权限切换 ==="
grep -E "(thaw|freeze)" gum_debug.log | head -20

echo -e "\n=== 检查失败的 mprotect ==="
grep "FAILED" gum_debug.log

echo -e "\n=== 查找崩溃地址 ==="
grep -i "7b64cfc" gum_debug.log
```

运行：
```bash
chmod +x check_gum_logs.sh
./check_gum_logs.sh
```
