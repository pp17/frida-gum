# Stalker SIGSEGV 修复说明

## 问题描述

在初次修改后，Stalker 功能出现了 `SIGSEGV` 错误：
```
Fatal signal 11 (SIGSEGV), code 1 (SEGV_MAPERR), fault addr 0x7b64cfc000
```

## 根本原因

初次修改强制 Stalker 在 Android 上使用 RW 权限分配内存，但没有正确处理权限切换：

1. **错误的方法**：直接强制分配 RW 内存
   ```c
   // 错误：内存是 RW，但从未切换为 RX
   base = gum_memory_allocate(..., GUM_PAGE_RW);
   ```

2. **问题**：Stalker 使用 `gum_stalker_freeze()` 来切换权限，但该函数检查 `is_rwx_supported` 标志：
   ```c
   void gum_stalker_freeze(GumStalker * self, gpointer code, gsize size) {
     if (!self->is_rwx_supported)
       gum_memory_mark_code(code, size);  // 切换到 RX
   }
   ```

3. **失败原因**：如果 `is_rwx_supported` 仍然为 true，`freeze()` 不会切换权限，代码保持 RW，无法执行。

## 正确的解决方案

在 Android 上强制禁用 RWX 支持，让 Stalker 使用内置的 RW/RX 切换机制：

```c
// 在 gum_stalker_init() 中
#ifdef HAVE_ANDROID
  /*
   * On Android, force disable RWX support to ensure proper RW/RX switching.
   * This is safer and avoids detection by security mechanisms.
   */
  self->is_rwx_supported = FALSE;
#else
  self->is_rwx_supported = gum_query_rwx_support () != GUM_RWX_NONE;
#endif
```

## 工作流程

设置 `is_rwx_supported = FALSE` 后，Stalker 自动执行以下流程：

### 1. 内存分配
```c
// 因为 is_rwx_supported = FALSE，分配 RW 内存
base = gum_memory_allocate(NULL, stalker->ctx_size, stalker->page_size,
    stalker->is_rwx_supported ? GUM_PAGE_RWX : GUM_PAGE_RW);
// 结果: RW 内存
```

### 2. 代码生成阶段
```c
// 需要写入代码，确保是 RW
gum_stalker_thaw(stalker, code, size);
// -> gum_mprotect(code, size, GUM_PAGE_RW)

// 写入代码...
gum_arm64_writer_put_xxx(...);
```

### 3. 代码执行阶段
```c
// 完成写入，切换为可执行
gum_stalker_freeze(stalker, code, size);
// -> gum_memory_mark_code(code, size)
// -> gum_try_mprotect(code, size, GUM_PAGE_RX)
// 结果: RX 内存（可执行但不可写）
```

### 4. 修改已存在的代码
```c
// 需要修改，切回 RW
gum_stalker_thaw(stalker, code, size);
// -> gum_mprotect(code, size, GUM_PAGE_RW)

// 修改代码...

// 完成修改，再次切换为 RX
gum_stalker_freeze(stalker, code, size);
// -> gum_memory_mark_code(code, size)
```

## 关键优势

1. **自动权限管理**：Stalker 内部已经有完整的 thaw/freeze 机制，我们只需要启用它

2. **无需修改分配代码**：通过设置标志，所有现有的逻辑都能正确工作

3. **统一处理**：ARM、ARM64、x86 所有架构使用相同的方法

4. **安全性**：永远不会有 RWX 页面，代码要么可写（RW）要么可执行（RX）

## 修改的文件

1. `gum/backend-arm64/gumstalker-arm64.c`
2. `gum/backend-arm/gumstalker-arm.c`
3. `gum/backend-x86/gumstalker-x86.c`

所有文件在 `gum_stalker_init()` 函数中添加相同的修改。

## 验证方法

修复后，你可以通过以下方式验证：

```bash
# 1. 检查进程内存映射，不应该有 rwx 页面
adb shell cat /proc/[pid]/maps | grep rwx
# 应该没有输出

# 2. 检查 Stalker 代码页面
adb shell cat /proc/[pid]/maps | grep 'r-xp.*\[anon:.*gum\]'
# 应该看到 r-xp（可读可执行）的匿名映射

# 3. 运行 Stalker 代码
// 应该不再出现 SIGSEGV 错误
```

## 性能影响

- RW/RX 切换有轻微开销，但比 RWX 安全得多
- Stalker 使用缓存减少不必要的切换
- 实际性能影响通常可忽略不计

## 总结

这个修复采用了"最小干预"原则：
- ✅ 不修改内存分配代码
- ✅ 不修改 thaw/freeze 逻辑
- ✅ 只设置一个标志，让现有机制正常工作
- ✅ 所有架构统一处理
- ✅ 安全性和功能性兼得
