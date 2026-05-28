# CodeLines 功能修改审查报告

## 一、设计意图

在 WinDirStat 现有的文件大小统计功能基础上，新增一路**代码行数统计**功能：
- 扫描时对文本/代码文件计算行数
- 在文件树视图中显示行数列
- 可选择用行数代替文件大小来渲染 TreeMap
- 设置界面提供开关选项

---

## 二、修改总览

| 分类 | 文件 | 修改内容 |
|------|------|----------|
| **核心计数** | HelpersTasks.h/cpp | `CountFileLines()` 函数 |
| **扫描器接口** | Finder.h | 虚函数 `GetFileLineCount()` |
| **扫描器实现** | FinderBasic.h/cpp | 存储并返回 `m_lineCount` |
| **扫描器实现** | FinderNtfs.h/cpp | 实时计算行数 |
| **数据模型** | Item.h | `m_lineCount` 成员 + 列枚举 + 方法声明 |
| **数据模型** | Item.cpp | 行数的增/删/排序实现 |
| **数据显示** | Item.Extended.cpp | GetText/CompareSibling/TmiGetSize |
| **配置项** | Options.h/cpp | `ShowColumnLineCount` / `ShowLineCountInsteadOfSize` |
| **设置页面** | PageGeneral.h/cpp | TreeMap 行数替代大小 的复选框 |
| **设置页面** | PageFileTree.h/cpp | 行数列显示 的复选框 |
| **视图** | Views/FileTreeView.cpp | 插入行数列 |
| **资源** | resource.h | 控件 ID 定义 |
| **资源** | windirstat.rc | 对话框控件 |
| **语言** | LangStrings.h | `IDS_COL_LINE_COUNT` / `IDS_PAGE_GENERAL_SHOW_LINES` |
| **语言** | 23 个 lang_*.txt | 翻译字符串 |

---

## 三、逐项审查

### 3.1 `CountFileLines()` — HelpersTasks.h/cpp

**当前代码：**
```cpp
ULONGLONG CountFileLines(const std::wstring& filePath)
{
    const std::wstring extension = filePath.substr(filePath.find_last_of(L'.'));
    const std::vector<std::wstring> textExtensions = { ... 41 种扩展名 ... };

    if (std::find(...) == textExtensions.end()) return 0;

    // 打开文件 → CreateFileMapping → MapViewOfFile
    // 遍历字节数 \n/\r\n 计数
    // 限制 100MB
}
```

**问题：**

| # | 严重性 | 问题 | 建议 |
|---|--------|------|------|
| 1 | ⚠️ 中 | `find_last_of(L'.')` 对无扩展名文件返回 `npos`，`substr(npos)` 抛异常 | 需加 `npos` 检查 |
| 2 | ⚠️ 中 | `textExtensions` 是 `vector<wstring>`，每次调用都构造，在扫描线程热路径上性能差 | 改为 `static const unordered_set<wstring_view>` |
| 3 | ⚡ 低 | 扩展名比较区分大小写，但文件系统可能为 `.CPP` | 统一转小写后比较 |
| 4 | ⚡ 低 | 逐字节扫描效率偏低 | 可用 SIMD 或 `strchr` 优化，但对 100MB 限制可接受 |

**建议修改：**
```cpp
ULONGLONG CountFileLines(const std::wstring& filePath)
{
    const auto dotPos = filePath.find_last_of(L'.');
    if (dotPos == std::wstring::npos) return 0;

    static const std::unordered_set<std::wstring> textExtensions = { ... };
    std::wstring extension = filePath.substr(dotPos);
    _wcslwr_s(extension.data(), extension.size() + 1);
    if (!textExtensions.contains(extension)) return 0;

    // ... 其余不变 ...
}
```

---

### 3.2 Finder 系列修改

#### 3.2.1 `Finder.h` — 纯虚函数

```diff
+ virtual inline ULONGLONG GetFileLineCount() const = 0;
```

✅ 合理。在基类添加纯虚函数，确保所有 Finder 子类必须实现。

#### 3.2.2 `FinderBasic.h` — 成员与方法

```diff
+ ULONGLONG m_lineCount = 1;
+ inline ULONGLONG GetFileLineCount() const override { return m_lineCount; }
```

**问题：**

| # | 严重性 | 问题 | 建议 |
|---|--------|------|------|
| 5 | ⚠️ 中 | 初始值 `= 1` 有歧义：目录行数为 0，但文件初始值 1 会导致未扫描前返回 1 | 初始值应为 0，由 `FindNext()` 赋予正确值 |

#### 3.2.3 `FinderBasic.cpp` — 扫描时赋值

```diff
+ #include "HelpersTasks.h"
  ...
+ // Calculate line count for text files
+ m_lineCount = 1;
+ if ((m_currentInfo->FileAttributes & FILE_ATTRIBUTE_DIRECTORY) == 0)
+ {
+     m_lineCount = CountFileLines(GetFilePathLong());
+ }
```

**问题：**

| # | 严重性 | 问题 | 建议 |
|---|--------|------|------|
| 6 | ⚠️ 中 | 非 text 文件 `m_lineCount = 1`，但非文本文件不应有行数（应返回 0），行数列显示 1 令人困惑 | 目录 → 0，非文本文件 → 0，文本文件 → 实际行数 |
| 7 | ⚡ 低 | `CountFileLines()` 在每次 `FindNext()` 中调用，对非文本文件也会先判断扩展名，性能可接受 | 无需修改 |

#### 3.2.4 `FinderNtfs.h/cpp` — NTFS 扫描行数

```diff
+ ULONGLONG GetFileLineCount() const override;  // FinderNtfs.h

  // FinderNtfs.cpp
+ ULONGLONG FinderNtfs::GetFileLineCount() const
+ {
+     if ((m_currentRecord->Attributes & FILE_ATTRIBUTE_DIRECTORY) != 0)
+         return 0;
+     ULONGLONG lineCount = CountFileLines(GetFilePath());
+     if (COptions::ShowLineCountInsteadOfSize && lineCount == 0)
+         return 1;
+     return lineCount;
+ }
```

**问题：**

| # | 严重性 | 问题 | 建议 |
|---|--------|------|------|
| 8 | 🔴 高 | **Finder 不应依赖 UI 设置**。`GetFileLineCount()` 是数据层方法，在扫描阶段调用时用户可能尚未勾选 `ShowLineCountInsteadOfSize`。行数应该是客观数据，不应受 UI 选项影响 | 移除 `COptions::ShowLineCountInsteadOfSize` 判断。二进制/非文本文件行数为 0 即可 |
| 9 | ⚠️ 中 | NTFS 快速扫描路径上调用 `CountFileLines()` 需要打开文件句柄做内存映射，与 MFT 原生快速扫描的设计理念矛盾，会大幅拖慢 NTFS 扫描速度 | 考虑在 NTFS 扫描完成后异步/延迟计算行数，或仅对文本文件才调用 |

---

### 3.3 Item 数据模型

#### 3.3.1 `Item.h` — 列枚举、成员、方法声明

```diff
  enum ITEMCOLUMNS : std::uint8_t
  {
      COL_NAME,
      COL_SIZE_PROPORTION,
      COL_PERCENTAGE,
      COL_OPTIONAL_START,
+     COL_LINE_COUNT,       // ← 在 COL_OPTIONAL_START 之后
      COL_SIZE_PHYSICAL,
      ...
  };

+ ULONGLONG GetLineCount() const noexcept;
+ void SetLineCount(ULONGLONG lineCount) noexcept;
+ void UpwardAddLineCount(ULONGLONG lines) noexcept;
+ void UpwardSubtractLineCount(ULONGLONG lines) noexcept;
+ void SortItemsByLineCount() const;

+ std::atomic<ULONGLONG> m_lineCount = 0;  // 成员变量
```

✅ 合理。列枚举位置正确（可选列之首），方法签名与现有 `SizePhysical`/`SizeLogical` 一致。`atomic` 保证了线程安全。

#### 3.3.2 `Item.cpp` — 方法实现与调用点

**AddChild() 中累加行数：**
```diff
  void CItem::AddChild(CItem* child, const bool addOnly)
  {
      if (!addOnly)
      {
          UpwardAddSizePhysical(child->GetSizePhysical());
          UpwardAddSizeLogical(child->GetSizeLogical());
+         UpwardAddLineCount(child->GetLineCount());
          UpwardUpdateLastChange(child->GetLastChange());
```

✅ 合理。与 `UpwardAddSizePhysical`/`UpwardAddSizeLogical` 对称。

**AddFile() 中设置行数：**
```diff
  child->SetLineCount(finder.GetFileLineCount());
```

✅ 合理。

**UpdateStatsFromDisk() 中更新行数：**
```diff
  UpwardSubtractLineCount(GetLineCount());
  UpwardAddLineCount(finder.GetFileLineCount());
```

✅ 合理。先减旧值再加新值。

**SetDone() 排序逻辑：**
```diff
- COptions::TreeMapUseLogical ? SortItemsBySizeLogical() : SortItemsBySizePhysical();
+ COptions::ShowLineCountInsteadOfSize ? SortItemsByLineCount() :
+     COptions::TreeMapUseLogical ? SortItemsBySizeLogical() : SortItemsBySizePhysical();
```

✅ 合理。三元优先级：行数 > 逻辑大小 > 物理大小。

**SortItemsByLineCount() 实现：**
```cpp
void CItem::SortItemsByLineCount() const
{
    if (IsLeaf()) return;
    m_folderInfo->m_children.shrink_to_fit();
    std::ranges::sort(m_folderInfo->m_children, [](const CItem* item1, const CItem* item2)
    {
        ULONGLONG size1, size2;
        if (COptions::ShowLineCountInsteadOfSize)
        {
            size1 = item1->GetLineCount();
            size2 = item2->GetLineCount();
        }
        else
        {
            size1 = item1->GetSizePhysical();
            size2 = item2->GetSizePhysical();
        }
        if (size1 != size2) return size1 > size2;
        if (item1->IsTypeOrFlag(IT_FILE) != item2->IsTypeOrFlag(IT_FILE))
            return item1->IsTypeOrFlag(IT_FILE);
        return item1->GetName() < item2->GetName();
    });
}
```

**问题：**

| # | 严重性 | 问题 | 建议 |
|---|--------|------|------|
| 10 | 🔴 高 | **逻辑冗余**。`SortItemsByLineCount()` 内部又检查 `ShowLineCountInsteadOfSize`，在 `ShowLineCountInsteadOfSize=false` 时按 size 排序。但此函数只在 `SetDone()` 中当 `ShowLineCountInsteadOfSize=true` 时才调用，所以 else 分支永远不会执行 | 移除 else 分支，此函数应始终按 `GetLineCount()` 排序 |
| 11 | ⚡ 低 | 排序 tie-breaker 中 `item1->IsTypeOrFlag(IT_FILE)` 返回 bool，隐式转为 int(0/1) 做比较，逻辑正确但可读性差 | 建议改为 `return !item1->IsTypeOrFlag(IT_FILE);`（文件夹排前面） |

#### 3.3.3 `Item.Extended.cpp` — 显示与比较

**GetText()：**
```diff
+ case COL_LINE_COUNT:
+     return FormatCount(GetLineCount());
```

✅ 合理。使用 `FormatCount` 格式化数字。

**CompareSibling()：**
```diff
+ case COL_LINE_COUNT:
+ {
+     return usignum(GetLineCount(), other->GetLineCount());
+ }
```

✅ 合理。使用 `usignum` 做无符号整数比较。

**TmiGetSize()：**
```diff
- return COptions::TreeMapUseLogical ? GetSizeLogical() : GetSizePhysical();
+ if (COptions::ShowLineCountInsteadOfSize)
+ {
+     return GetLineCount();
+ }
+ return COptions::TreeMapUseLogical ? GetSizeLogical() : GetSizePhysical();
```

✅ 合理。TreeMap 用行数替代大小来决定矩形面积。

---

### 3.4 配置项 — Options.h/cpp

```diff
  // Options.h
+ static Setting<bool> ShowColumnLineCount;       // 控制列显隐
+ static Setting<bool> ShowLineCountInsteadOfSize; // 控制 TreeMap 模式

  // Options.cpp
+ Setting<bool> COptions::ShowColumnLineCount(OptionsFileTree, L"ShowColumnLineCount", true);
+ Setting<bool> COptions::ShowLineCountInsteadOfSize(OptionsGeneral, L"ShowLineCountInsteadOfSize", true);
```

**问题：**

| # | 严重性 | 问题 | 建议 |
|---|--------|------|------|
| 12 | ⚠️ 中 | `ShowLineCountInsteadOfSize` 默认值 `true` — 用户首次启动就会看到 TreeMap 用行数渲染，对不了解此功能的用户造成困惑 | 默认值改为 `false`，让用户主动开启 |

---

### 3.5 设置页面

#### PageGeneral — "Show line count instead of size"
```diff
+ DDX_Check(pDX, IDC_SHOW_LINE_COUNT_INSTEAD_OF_SIZE, m_showLineCountInsteadOfSize);
+ ON_BN_CLICKED(IDC_SHOW_LINE_COUNT_INSTEAD_OF_SIZE, OnBnClickedSetModified)
+ m_showLineCountInsteadOfSize = COptions::ShowLineCountInsteadOfSize;
+ COptions::ShowLineCountInsteadOfSize = (FALSE != m_showLineCountInsteadOfSize);
```

✅ 合理。与其他 checkbox 设置一致。

#### PageFileTree — "Line Count column"
```diff
+ DDX_Check(pDX, IDC_TREECOL_LINE_COUNT, m_showColumnLineCount);
+ m_showColumnLineCount = COptions::ShowColumnLineCount;
+ COptions::ShowColumnLineCount = (FALSE != m_showColumnLineCount);
```

✅ 合理。

---

### 3.6 视图与资源

#### FileTreeView.cpp
```diff
+ if (COptions::ShowColumnLineCount) InsertCol(IDS_COL_LINE_COUNT, LVCFMT_RIGHT, 90, COL_LINE_COUNT);
```

✅ 合理。列宽 90 与其他列一致。

#### resource.h
```diff
+ #define IDC_TREECOL_LINE_COUNT          1123
+ #define IDC_SHOW_LINE_COUNT_INSTEAD_OF_SIZE 1125
```

✅ 合理。ID 不与现有控件冲突。

#### windirstat.rc — 对话框控件
```diff
+ CONTROL "IDS_COL_LINE_COUNT",IDC_TREECOL_LINE_COUNT,"Button",BS_AUTOCHECKBOX | WS_TABSTOP,14,79,111,12
+ CONTROL "IDS_PAGE_GENERAL_SHOW_LINES",IDC_SHOW_LINE_COUNT_INSTEAD_OF_SIZE,... "Button",BS_AUTOCHECKBOX | WS_TABSTOP
```

✅ 合理。

#### LangStrings.h
```diff
+ constexpr std::wstring_view IDS_COL_LINE_COUNT = L"IDS_COL_LINE_COUNT";
+ constexpr std::wstring_view IDS_PAGE_GENERAL_SHOW_LINES = L"IDS_PAGE_GENERAL_SHOW_LINES";
```

✅ 合理。

#### 23 个语言文件
每个文件新增两行翻译。✅ 合理。

---

## 四、问题汇总与优先级

| # | 严重性 | 文件 | 问题 | 修复方案 |
|---|--------|------|------|----------|
| 1 | ⚠️ 中 | HelpersTasks.cpp | `find_last_of(L'.')` 对无扩展名文件返回 npos 导致异常 | 加 npos 检查 |
| 2 | ⚠️ 中 | HelpersTasks.cpp | textExtensions 每次调用构造 vector，热路径性能差 | 改为 static const unordered_set |
| 3 | ⚡ 低 | HelpersTasks.cpp | 扩展名比较区分大小写 | 统一转小写 |
| 5 | ⚠️ 中 | FinderBasic.h | `m_lineCount = 1` 初始值语义不清 | 改为 `= 0` |
| 6 | ⚠️ 中 | FinderBasic.cpp | 非文本文件 `m_lineCount = 1` 不合理 | 改为 0 |
| 8 | 🔴 高 | FinderNtfs.cpp | `GetFileLineCount()` 依赖 UI 选项 `ShowLineCountInsteadOfSize` | 移除 UI 依赖，行数是客观数据 |
| 9 | ⚠️ 中 | FinderNtfs.cpp | NTFS 扫描中调用 CountFileLines() 拖慢速度 | 可考虑异步计算或仅对文本文件调用 |
| 10 | 🔴 高 | Item.cpp | `SortItemsByLineCount()` 内部冗余检查 `ShowLineCountInsteadOfSize` | 移除 else 分支 |
| 11 | ⚡ 低 | Item.cpp | tie-breaker 可读性差 | 可选优化 |
| 12 | ⚠️ 中 | Options.cpp | `ShowLineCountInsteadOfSize` 默认值 true 令新用户困惑 | 改为 false |

---

## 五、推荐修改方案

### 修改 1: CountFileLines 安全性与性能 (问题 #1 #2 #3)

```cpp
// HelpersTasks.cpp
ULONGLONG CountFileLines(const std::wstring& filePath)
{
    const auto dotPos = filePath.find_last_of(L'.');
    if (dotPos == std::wstring::npos || dotPos + 1 >= filePath.size())
        return 0;

    static const std::unordered_set<std::wstring> textExtensions = {
        L".txt", L".cpp", L".h", L".c", L".hpp", L".cxx", L".cc", L".cs", L".java",
        L".py", L".js", L".ts", L".html", L".htm", L".css", L".xml", L".json",
        L".sql", L".php", L".rb", L".pl", L".sh", L".bat", L".cmd", L".ini",
        L".cfg", L".conf", L".log", L".md", L".rst", L".csv", L".tex", L".yml", L".yaml",
        L".cg", L".hlsl", L".glsl", L".vert", L".frag", L".geom", L".tesc", L".tese", L".comp"
    };

    std::wstring extension = filePath.substr(dotPos);
    _wcslwr_s(extension.data(), extension.size() + 1);
    if (!textExtensions.contains(extension))
        return 0;

    // ... 其余不变 ...
}
```

### 修改 2: FinderBasic 初始值 (问题 #5 #6)

```cpp
// FinderBasic.h
- ULONGLONG m_lineCount = 1;
+ ULONGLONG m_lineCount = 0;

// FinderBasic.cpp
- m_lineCount = 1;
- if ((m_currentInfo->FileAttributes & FILE_ATTRIBUTE_DIRECTORY) == 0)
- {
-     m_lineCount = CountFileLines(GetFilePathLong());
- }
+ m_lineCount = 0;
+ if ((m_currentInfo->FileAttributes & FILE_ATTRIBUTE_DIRECTORY) == 0)
+ {
+     m_lineCount = CountFileLines(GetFilePathLong());
+ }
```

### 修改 3: FinderNtfs 移除 UI 依赖 (问题 #8)

```cpp
// FinderNtfs.cpp
  ULONGLONG FinderNtfs::GetFileLineCount() const
  {
      if ((m_currentRecord->Attributes & FILE_ATTRIBUTE_DIRECTORY) != 0)
          return 0;

-     ULONGLONG lineCount = CountFileLines(GetFilePath());
-     if (COptions::ShowLineCountInsteadOfSize && lineCount == 0)
-     {
-         return 1;
-     }
-     return lineCount;
+     return CountFileLines(GetFilePath());
  }
```

### 修改 4: SortItemsByLineCount 移除冗余分支 (问题 #10)

```cpp
// Item.cpp
  void CItem::SortItemsByLineCount() const
  {
      if (IsLeaf()) return;

      m_folderInfo->m_children.shrink_to_fit();
      std::ranges::sort(m_folderInfo->m_children, [](const CItem* item1, const CItem* item2)
      {
-         ULONGLONG size1, size2;
-         if (COptions::ShowLineCountInsteadOfSize)
-         {
-             size1 = item1->GetLineCount();
-             size2 = item2->GetLineCount();
-         }
-         else
-         {
-             size1 = item1->GetSizePhysical();
-             size2 = item2->GetSizePhysical();
-         }
+         const ULONGLONG count1 = item1->GetLineCount();
+         const ULONGLONG count2 = item2->GetLineCount();

-         if (size1 != size2)
-             return size1 > size2;
+         if (count1 != count2)
+             return count1 > count2;

          if (item1->IsTypeOrFlag(IT_FILE) != item2->IsTypeOrFlag(IT_FILE))
              return item1->IsTypeOrFlag(IT_FILE);

          return item1->GetName() < item2->GetName();
      });
  }
```

### 修改 5: ShowLineCountInsteadOfSize 默认值 (问题 #12)

```cpp
// Options.cpp
- Setting<bool> COptions::ShowLineCountInsteadOfSize(OptionsGeneral, L"ShowLineCountInsteadOfSize", true);
+ Setting<bool> COptions::ShowLineCountInsteadOfSize(OptionsGeneral, L"ShowLineCountInsteadOfSize", false);
```

---

## 六、可选优化（非必须）

| 优化 | 说明 |
|------|------|
| NTFS 异步行数计算 | NTFS 快速扫描路径调用 `CountFileLines()` 需要打开文件句柄，可考虑在扫描完成后用线程池异步计算 |
| SIMD 行计数 | 对大文件（接近 100MB）可用 SIMD 指令加速 `\n` 搜索，但对小文件收益有限 |
| 扩展名列表可配置 | 允许用户在设置页面自定义要统计行数的文件扩展名 |
| 行数列右对齐格式 | 当前 `FormatCount` 与其他数值列一致，如需可添加千分位分隔 |

---

## 七、已修复的合并冲突问题（本次会话）

| 文件 | 问题 | 状态 |
|------|------|------|
| FinderBasic.h | 重复的 `m_lineCount` 成员和 `GetFileLineCount()` 方法 | ✅ 已修复 |
| Item.Extended.cpp | 重复 `case COL_SIZE_PHYSICAL:` 和未闭合花括号 | ✅ 已修复 |
| HelpersTasks.cpp | `CopyAllDriveMappings()` 被 `CountFileLines()` 截断 | ✅ 已修复 |
| HelpersTasks.cpp | `SmartPointer<HANDLE>` 显式模板参数导致编译错误 | ✅ 已修复 |
| LangStrings.h | 缺少 `IDS_COL_LINE_COUNT` 和 `IDS_PAGE_GENERAL_SHOW_LINES` | ✅ 已修复 |
| 23 个语言文件 | 缺少行数相关翻译字符串 | ✅ 已修复 |
