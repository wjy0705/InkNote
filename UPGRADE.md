# InkNote1.2.html 升级记录

## 升级概览

本次修复了 3 个用户体验问题，涉及新建笔记、保存流程和关闭文件三个核心交互。

---

## 1. 新建笔记编辑器未激活

**问题：** 用户点击"＋"新建笔记后，页面显示的是预览模式，编辑器没有自动打开，用户无法直接输入内容。

**原因：** `createNewFile()` 先调用 `openFile()` 渲染 Markdown 预览，紧接着调用 `enterEditMode()` 替换为编辑器。这个"先渲染预览再替换编辑器"的时序导致编辑器未正确激活。

**修复：** `createNewFile()` 跳过 `openFile()` 的预览渲染，直接设置 `state.currentFile` + 高亮侧边栏 + 调用 `enterEditMode()`。

**改动位置：** 第 1508-1528 行

```javascript
// 修复前
function createNewFile() {
  // ...
  openFile(file.path)    // 先渲染预览
  enterEditMode()         // 再替换为编辑器（时序问题）
}

// 修复后
function createNewFile() {
  // ...
  state.currentFile = file
  // 直接高亮 tree，跳过 openFile 的预览渲染
  enterEditMode()         // 直接进入编辑模式
}
```

---

## 2. 保存流程顺序优化

**问题：** 保存新建的 MD 文件时，流程是"弹框改文件名 → 确认 → 打开目录选择器"。用户期望先选目录再改文件名，或者一步完成。

**原因：** 原实现用 `showDirectoryPicker` 只能选目录，不能改文件名，所以需要先弹自定义 modal 让用户输入文件名。

**修复：** 改用 `showSaveFilePicker` API（原生"另存为"对话框），一个界面同时完成选目录和改文件名。不支持该 API 的浏览器走 fallback（自定义 modal + 下载）。

**改动位置：** 第 1531-1613 行

**流程对比：**
```
旧：点保存 → 弹modal改名 → 确认 → showDirectoryPicker → 写入
新：点保存 → showSaveFilePicker（选目录+改名一步到位）→ 写入
```

---

## 3. 关闭文件后侧边栏未清理

**问题：** 点击关闭文件按钮后，页面内容清除了（回到拖拽区），但侧边栏文件树仍然显示已关闭的文件。

**原因：** `closeFile()` 只清除了 `state.currentFile` 和内容区的 active 高亮，没有从 `state.files` 移除未保存文件，也没有调用 `refreshTree()` 重建侧边栏。

**修复：** `closeFile()` 在关闭前检查文件是否未保存（`_unsaved`），如果是则从 `state.files` 中移除，然后调用 `refreshTree()` 重建侧边栏。

**改动位置：** 第 1749-1761 行

---

## 自测验证

| 测试 | 验证内容 | 状态 |
|------|---------|------|
| Test 15 | `createNewFile` 后 `state._editing === true` | ✓ |
| Test 15 | `createNewFile` 后 `editorTextarea` 存在 | ✓ |
| Test 16 | 有 `showSaveFilePicker` 时直接调用，不弹 modal | ✓ |
| Test 17 | `closeFile` 后未保存文件从侧边栏移除 | ✓ |

## 兼容性

| 特性 | Chromium | Firefox | Safari |
|------|----------|---------|--------|
| `showSaveFilePicker` | ✓ | ✗ (fallback) | ✗ (fallback) |
| fallback（modal + 下载） | ✓ | ✓ | ✓ |
