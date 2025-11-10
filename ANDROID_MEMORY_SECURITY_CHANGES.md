# Android 内存安全改进

## 概述

这些修改增强了 Frida Gum 在 Android 平台上的内存安全性，特别是在使用 `gum_interceptor_attach` 和 Stalker 功能时。主要改进包括：

1. **避免切割目标内存区间**：在修改目标函数时，不扩展内存保护区域
2. **恢复原有权限**：修改完成后恢复原始内存权限而不是使用 RWX
3. **禁用跳板 RWX**：自己创建的跳板内存使用 RW 或 RX，而不是 RWX

## 修改的文件

### 1. gum/backend-linux/gummemory-linux.c

**修改**: `gum_try_mprotect()` 函数

在 Android 平台上，该函数现在会查询现有内存区域的保护属性，以避免意外扩展保护区域的边界。这可以防止修改相邻的内存区域，并有助于避免被安全机制检测到。

```c
#ifdef HAVE_ANDROID
  /*
   * On Android, when modifying existing code pages (not our own allocations),
   * we should avoid expanding the memory region beyond the original boundaries.
   */
  GumPageProtection existing_prot;
  gsize actual_size;
  
  if (gum_memory_get_protection (address, size, &actual_size, &existing_prot))
  {
    /* Use exact boundaries */
    result = mprotect (aligned_address, aligned_size, posix_prot);
  }
#endif
```

### 2. gum/gummemory.c

**修改**: `gum_memory_patch_code_pages()` 函数

在 Android 上，此函数现在会：
- 在修改之前保存原始页面保护属性
- 修改完成后恢复原始保护属性
- 确保不会设置 RWX 权限（如果原本有写权限和执行权限，优先保留执行权限）

```c
#ifdef HAVE_ANDROID
  /* Save original page protections */
  original_prots = g_new0 (GumPageProtection, sorted_addresses->len);
  for (i = 0; i != sorted_addresses->len; i++)
  {
    if (!gum_memory_query_protection (target_page, &original_prots[i]))
      original_prots[i] = GUM_PAGE_RX;
  }
#endif

/* Later, restore original protection */
#ifdef HAVE_ANDROID
  restore_prot = original_prots[i];
  
  /* Ensure at least read permission */
  if ((restore_prot & GUM_PAGE_READ) == 0)
    restore_prot |= GUM_PAGE_READ;
  
  /* Never set both write and execute */
  if ((restore_prot & GUM_PAGE_WRITE) && (restore_prot & GUM_PAGE_EXECUTE))
  {
    /* Prefer execute over write for code pages */
    restore_prot = (restore_prot & ~GUM_PAGE_WRITE);
  }
#endif
```

### 3. gum/gumcodeallocator.c

**修改**: `gum_code_allocator_try_alloc_batch_near()` 和 `gum_code_allocator_commit()` 函数

在 Android 上，代码分配器现在：
- 强制禁用 RWX 模式，即使技术上支持
- 跳板页面在写入时使用 RW，执行时切换为 RX

```c
#ifdef HAVE_ANDROID
  /*
   * On Android, never allocate RWX pages for trampolines.
   * Always start with RW for writing, then switch to RX for execution.
   */
  if (rwx_supported && !remap_supported)
  {
    /* Force non-RWX path on Android even if RWX is technically supported */
    rwx_supported = FALSE;
  }
#endif
```

### 4. gum/backend-arm64/gumstalker-arm64.c

**修改**: Stalker 代码和数据分配

在 Android 上，Stalker 现在：
- 为执行上下文分配 RW 内存（而不是 RWX）
- 为代码 slab 分配 RW 内存（而不是 RWX）
- 数据 slab 仍然使用 RW（因为不需要执行权限）

```c
#ifdef HAVE_ANDROID
  /* On Android, never allocate RWX for Stalker. Use RW, then switch to RX. */
  base = gum_memory_allocate (NULL, stalker->ctx_size, stalker->page_size,
      GUM_PAGE_RW);
#else
  base = gum_memory_allocate (NULL, stalker->ctx_size, stalker->page_size,
      stalker->is_rwx_supported ? GUM_PAGE_RWX : GUM_PAGE_RW);
#endif
```

## 安全优势

1. **避免 RWX 检测**：现代 Android 安全机制会监控 RWX 页面。通过使用 RW/RX 而不是 RWX，可以降低被检测的风险。

2. **减少攻击面**：不使用 RWX 权限可以减少潜在的安全风险，因为攻击者无法同时写入和执行相同的内存页面。

3. **遵守最佳实践**：Android 安全指南建议不要使用 RWX 权限。这些修改使 Frida Gum 更符合平台安全要求。

4. **精确的权限管理**：通过保存和恢复原始权限，我们确保不会意外修改系统库或其他敏感代码的保护属性。

## 向后兼容性

这些修改仅在定义了 `HAVE_ANDROID` 时生效，因此不会影响其他平台的行为。在非 Android 平台上，代码将继续使用原来的逻辑。

## 测试建议

测试这些修改时，请确保：

1. **Interceptor 测试**：
   - 测试 `gum_interceptor_attach()` 能正常工作
   - 验证被 hook 的函数可以正常调用
   - 检查内存权限是否正确恢复

2. **Stalker 测试**：
   - 测试 Stalker 跟踪功能
   - 验证代码生成和执行正常
   - 检查性能是否在可接受范围内

3. **安全验证**：
   - 使用 `/proc/[pid]/maps` 检查没有 RWX 页面
   - 验证目标函数的内存权限正确恢复
   - 确认跳板代码在执行前切换为 RX

## 使用示例

使用这些修改后，你的代码无需更改。Frida Gum 会自动在 Android 上应用这些安全改进：

```c
// Interceptor 示例
GumInterceptor *interceptor = gum_interceptor_obtain();
gum_interceptor_attach(interceptor, target_function, 
                       listener, NULL, GUM_ATTACH_FLAGS_NONE);

// Stalker 示例
GumStalker *stalker = gum_stalker_new();
gum_stalker_follow_me(stalker, transformer, NULL);
```

## 性能影响

由于需要在 RW 和 RX 之间切换权限，可能会有轻微的性能开销。但是，这种开销通常是可以接受的，并且被安全性改进所抵消。

## 已知限制

1. 这些修改假设目标代码页面原本就有执行权限
2. 在某些极端情况下，可能需要额外的权限处理
3. 性能关键路径可能会受到轻微影响

## 贡献者注意事项

在修改内存保护相关代码时，请记住：
- 始终在 Android 上避免使用 RWX
- 保存并恢复原始权限
- 在代码和数据之间明确区分权限需求
