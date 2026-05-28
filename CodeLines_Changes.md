# CodeLines 分支修改记录

## 概述

CodeLines 分支基于 `80b6538`（Rearrange buttons on Toolbar）从 origin/master 分叉，新增了**代码行数统计与显示**功能，允许用户将文件树视图从传统的文件大小显示切换为代码行数显示。

### 分支信息
- **分叉点**: `80b6538` (Rearrange buttons on Toolbar)
- **CodeLines 独有提交数**: 13
- **origin/master 新增提交数**: ~400+（大量重构和新功能）

---

## CodeLines 独有提交列表

| # | 提交哈希 | 提交信息 | 说明 |
|---|---------|---------|------|
| 1 | `67e847f` | Change file statistics from size to line count | 核心功能：将文件统计从大小改为行数 |
| 2 | `281851e` | Extend supported file types for line counting to include CG and HLSL shader files | 扩展行数统计支持的文件类型 |
| 3 | `3fa1bf9` | Add UI option to switch between line count and file size display | 添加UI切换选项 |
| 4 | `cc7023d` | Fix resource duplication issue and add missing translations | 修复资源重复和翻译缺失 |
| 5 | `96ce508` | Fix empty STRINGTABLE compilation error | 修复空字符串表编译错误 |
| 6 | `86bca6f` | Add missing translations for all languages | 为所有语言添加缺失的翻译 |
| 7 | `0060f3d` | Fix duplicate resource definition causing CVT1100 error | 修复重复资源定义 |
| 8 | `869dc86` | Fix compile | 修复编译问题 |
| 9 | `def4cc3` | Fix symbol definition conflicts causing C2059 syntax error | 修复符号定义冲突 |
| 10 | `c73033a` | Build on latest version | 基于最新版本构建 |
| 11 | `878e6f8` | View Code lines | 查看代码行数 |
| 12 | `ce40ad1` | Fix sort function | 修复排序函数 |
| 13 | `0b3aa43` | Avoid 0 size file | 避免0大小文件 |

---

## 功能修改详情

### 1. 核心数据层 - 行数统计 (Line Count)

#### Finder.h
- **新增虚函数**: `virtual inline ULONGLONG GetFileLineCount() const = 0;`
- 所有 Finder 子类必须实现行数统计接口

#### FinderBasic.h / FinderBasic.cpp
- **新增成员**: `ULONGLONG m_LineCount = 1;`
- **新增方法**: `GetFileLineCount()` 返回 `m_LineCount`
- **修改逻辑**: `FindNext()` 中扫描文件时调用 `CountFileLines()` 计算行数

#### FinderNtfs.h / FinderNtfs.cpp
- **新增方法**: `GetFileLineCount()` 实现
- **新增包含**: `#include "GlobalHelpers.h"`, `#include "SmartPointer.h"`
- **特殊处理**: 
  - 目录返回 0
  - 当 `ShowLineCountInsteadOfSize` 启用且行数为 0 时返回 1（避免 TreeMap 中消失）

#### GlobalHelpers.h / GlobalHelpers.cpp
- **新增函数**: `ULONGLONG CountFileLines(const std::wstring& filePath)`
- **功能**: 
  - 按扩展名判断是否为文本文件（支持 .txt, .cpp, .h, .py, .js, .ts 等 40+ 种扩展名）
  - 使用内存映射文件 (MapViewOfFile) 高效读取
  - 100MB 大小限制，避免性能问题
  - 统计 `\n` 和 `\r\n` 换行符
  - 非文本文件返回 0

### 2. 数据模型层 - CItem 扩展

#### Item.h
- **新增枚举**: `COL_LINE_COUNT`（在 `COL_OPTIONAL_START` 之后、`COL_SIZE_PHYSICAL` 之前）
- **新增成员**: `std::atomic<ULONGLONG> m_LineCount = 0;`
- **新增方法**:
  - `ULONGLONG GetLineCount() const;`
  - `void SetLineCount(ULONGLONG lineCount);`
  - `void UpwardAddLineCount(ULONGLONG lines);`
  - `void UpwardSubtractLineCount(ULONGLONG lines);`
  - `void UpwardAddTreeMapSize(ULONGLONG bytes);` (目前为空实现)
  - `void SortItemsByLineCount() const;`
- **修改 TmiGetSize()**: 当 `ShowLineCountInsteadOfSize` 启用时返回行数

#### Item.cpp
- **GetText()**: 新增 `COL_LINE_COUNT` 列，显示格式化行数
- **CompareSibling()**: 新增 `COL_LINE_COUNT` 排序分支
- **UpdateStatsFromDisk()**: 添加 `UpwardSubtractLineCount` / `UpwardAddLineCount` 调用
- **AddChild()**: 添加 `UpwardAddLineCount` 调用
- **AddFile()**: 添加 `SetLineCount(finder.GetFileLineCount())` 调用
- **SetDone()**: 修改为调用 `SortItemsByLineCount()` 替代原来的 `SortItemsBySizePhysical/Logical()`
- **新增方法实现**:
  - `UpwardAddLineCount()` - 向上累加行数
  - `UpwardSubtractLineCount()` - 向上减少行数
  - `UpwardAddTreeMapSize()` - 空实现（注释中说明用于 TreeMap 一致性）
  - `SortItemsByLineCount()` - 按行数排序，与 `TmiGetSize()` 逻辑保持一致
    - 根据 `ShowLineCountInsteadOfSize` 选项选择行数或物理大小排序
    - 降序排列，等大小时按类型和名称排序

### 3. 选项层 - 新增设置

#### Options.h / Options.cpp
- **新增选项**:
  - `Setting<bool> COptions::ShowColumnLineCount` - 是否显示行数列（默认 true）
  - `Setting<bool> COptions::ShowLineCountInsteadOfSize` - 是否用行数替代大小显示（默认 true）

### 4. UI 层 - 设置页面

#### PageGeneral.h / PageGeneral.cpp
- **新增成员**: `BOOL m_ShowLineCountInsteadOfSize = FALSE;`
- **新增控件绑定**: `DDX_Check(pDX, IDC_SHOW_LINE_COUNT_INSTEAD_OF_SIZE, m_ShowLineCountInsteadOfSize)`
- **新增消息映射**: `ON_BN_CLICKED(IDC_SHOW_LINE_COUNT_INSTEAD_OF_SIZE, OnBnClickedSetModified)`
- **初始化**: 从 `COptions::ShowLineCountInsteadOfSize` 读取
- **保存**: 写入 `COptions::ShowLineCountInsteadOfSize`

#### PageFileTree.h / PageFileTree.cpp
- **新增成员**: `BOOL m_ShowColumnLineCount = FALSE;`
- **新增控件绑定**: `DDX_Check(pDX, IDC_TREECOL_LINE_COUNT, m_ShowColumnLineCount)`
- **初始化**: 从 `COptions::ShowColumnLineCount` 读取
- **保存**: 写入 `COptions::ShowColumnLineCount`

### 5. 视图层 - 列显示

#### FileTreeView.cpp
- **新增列**: 在 `CreateColumns()` 中，如果 `ShowColumnLineCount` 启用，插入行数列
  ```cpp
  if (COptions::ShowColumnLineCount)
      m_Control.InsertColumn(CHAR_MAX, Localization::Lookup(IDS_COL_LINE_COUNT).c_str(), LVCFMT_RIGHT, 90, COL_LINE_COUNT);
  ```

### 6. 资源文件

#### resource.h
- **新增 ID**: 
  - `#define IDC_TREECOL_LINE_COUNT 1031`（与 `IDC_FULL_ROW_SELECTION` 冲突！）
  - `#define IDC_SHOW_LINE_COUNT_INSTEAD_OF_SIZE 1243`（与 `IDC_SEARCH_REGEX` 冲突！）

#### LangStrings.h
- **新增字符串**:
  - `IDS_COL_LINE_COUNT = L"IDS_COL_LINE_COUNT"`
  - `IDS_PAGE_GENERAL_SHOW_LINES = L"IDS_PAGE_GENERAL_SHOW_LINES"`

#### windirstat.rc
- **IDD_PAGE_TREELIST**: 在可选列区域新增行数列复选框，调整后续控件 Y 坐标
- **IDD_PAGE_GENERAL**: 新增 "Show line count instead of file size" 复选框

### 7. 语言文件 (19个)

所有语言文件新增两行：
- `IDS_COL_LINE_COUNT=<翻译>` - 行数列标题
- `IDS_PAGE_GENERAL_SHOW_LINES=<翻译>` - 选项文本

| 语言 | IDS_COL_LINE_COUNT | IDS_PAGE_GENERAL_SHOW_LINES |
|------|-------------------|---------------------------|
| en | Lines | Show line count instead of file size |
| zh | 行数 | 显示行数而不是文件大小 |
| de | Zeilen | Zeilenanzahl statt Dateigröße anzeigen |
| ... | ... | ... |

**注意**: `IDS_COL_LASTCHANGE` 在 CodeLines 中保留旧名（无下划线），而 origin/master 改为 `IDS_COL_LAST_CHANGE`（有下划线）

---

## 与 origin/master 的冲突分析

### 主要冲突类型

1. **命名规范变更**（占冲突大部分）
   - CodeLines 使用 PascalCase: `m_Name`, `m_SizePhysical`, `m_FolderInfo`
   - origin/master 改为 camelCase: `m_name`, `m_sizePhysical`, `m_folderInfo`
   - origin/master 还将 `m_Name` 从 `std::wstring` 改为 `std::unique_ptr<wchar_t[]>`（内存优化）

2. **文件拆分重构**
   - origin/master 将 `GlobalHelpers.cpp/.h` 拆分为 `HelpersInterface.cpp/.h` 和 `HelpersTasks.cpp/.h`
   - CodeLines 的 `CountFileLines()` 函数在 `GlobalHelpers.cpp` 中，需要迁移

3. **资源 ID 冲突**
   - CodeLines 新增的 `IDC_TREECOL_LINE_COUNT` (1031) 与 origin/master 的 `IDC_FULL_ROW_SELECTION` (1031) 冲突
   - CodeLines 新增的 `IDC_SHOW_LINE_COUNT_INSTEAD_OF_SIZE` (1243) 与 origin/master 的 `IDC_SEARCH_REGEX` (1243) 冲突

4. **新增功能共存**
   - origin/master 新增了 Dark Mode、Context Menu、Auto-elevation 等
   - CodeLines 新增的 Line Count 功能需要与这些新功能共存

5. **语言字符串键名变更**
   - `IDS_COL_LASTCHANGE` → `IDS_COL_LAST_CHANGE`（master 加了下划线）

### 冲突文件清单

| 文件 | 冲突数 | 主要冲突类型 |
|------|--------|------------|
| FinderBasic.h | 1 | 命名规范 + 新增字段/方法 |
| FinderNtfs.cpp | 1 | 包含文件 |
| GlobalHelpers.cpp | 0 (deleted by them) | 文件被删除，需迁移 CountFileLines |
| GlobalHelpers.h | 0 (deleted by them) | 文件被删除，需迁移声明 |
| Item.cpp | 6 | 命名规范 + 新增方法 + 行数逻辑 |
| Item.h | 3 | 命名规范 + 新增成员/方法 + TmiGetSize |
| PageFileTree.cpp | 3 | 命名规范 + 行数列选项 |
| PageFileTree.h | 1 | 命名规范 + 新增成员 |
| PageGeneral.cpp | 4 | 命名规范 + 行数显示选项 |
| PageGeneral.h | 1 | 命名规范 + 新增成员 |
| FileTreeView.cpp | 1 | 列创建逻辑 |
| LangStrings.h | 1 | 字符串键名变更 + 新增 |
| resource.h | 1 | ID 冲突 + 新增 ID |
| windirstat.rc | 2 | 对话框布局 |
| 19个 lang_*.txt | 各1-2 | LASTCHANGE→LAST_CHANGE + 新增行数翻译 |

---

## 解决策略

### 总体原则
- 采用 origin/master 的代码结构（camelCase 命名、文件拆分、新功能）
- 将 CodeLines 的行数功能适配到 master 的代码框架中
- 不丢失 CodeLines 的任何功能逻辑

### 具体方案

1. **命名规范**: 全面采用 origin/master 的 camelCase 命名
2. **GlobalHelpers → HelpersTasks**: 将 `CountFileLines()` 迁移到 `HelpersTasks.cpp/.h`
3. **资源 ID**: 为 CodeLines 新增的控件分配不冲突的 ID
4. **语言字符串**: 使用 master 的 `IDS_COL_LAST_CHANGE` 键名，保留 `IDS_COL_LINE_COUNT`
5. **Item.h 成员变量**: 采用 master 的内存优化结构，新增 `m_lineCount`
6. **TmiGetSize()**: 在 master 的实现基础上加入行数判断逻辑
7. **SortItemsByLineCount()**: 新增该方法，但 `SetDone()` 中应保持 master 原有排序调用（行数排序作为额外选项）
