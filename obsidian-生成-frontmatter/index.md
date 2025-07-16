# obsidian 生成 Frontmatter



官方自带的模板还是太弱了, 如果需要自动生成frontmatter，还是需要第三方插件

目前是利用这个插件，因为使用js生成内容，所以理论上什么都可以用模板来填
https://github.com/SilentVoid13/Templater

先贴一下配置
```yaml
---
title: <% tp.file.folder() %>
date: <% tp.file.creation_date("YYYY-MM-DDTHH:mm:ss.SSSZ") %> 
lastmod: <% tp.file.last_modified_date("YYYY-MM-DDTHH:mm:ss.SSSZ") %>
draft: true
tags:
series:
description: 
<%*
  const fullPath = tp.file.folder(true); // 获取路径，例如 "content/posts/A/B/C"
  const pathArray = fullPath.split("/"); // 分割路径
  const endIndex = pathArray.length-1
  const categories = pathArray.slice(2, endIndex); // 提取第3和第4级目录（"A", "B"）
  tR += "categories:\n";
  if (categories.length === 0) {
    tR += "  - None\n"; // 如果没有足够的目录
  } else {
    for (const category of categories) {
      if (category) { // 确保目录非空
        tR += `  - ${category}\n`;
      }
    }
  }%>
---

```


在 Obsidian 中使用 Templater 插件获取文件的目录和上级目录，可以通过 `tp.file` 对象的相关方法实现。以下是具体方法和代码示例：

### 1. 获取当前文件所在目录
使用 `tp.file.folder()` 方法可以获取当前文件所在的目录。

- **语法**：
  ```javascript
  <% tp.file.folder() %>  // 返回当前文件的目录名称（不含路径）
  <% tp.file.folder(true) %>  // 返回当前文件的完整路径（相对于 Vault 根目录）
  ```

- **示例**：
  假设文件路径为 `Projects/ClientA/ProjectX/Note.md`：
  ```javascript
  <% tp.file.folder() %>  // 输出：ProjectX
  <% tp.file.folder(true) %>  // 输出：Projects/ClientA/ProjectX
  ```

### 2. 获取上级目录
Templater 本身没有直接提供获取上级目录的内置方法，但可以通过处理 `tp.file.folder(true)` 返回的路径来提取上级目录。可以使用 JavaScript 字符串操作或正则表达式来实现。

- **方法**：获取完整路径后，使用 `split()` 分割路径并提取上级目录。
- **示例代码**：
  ```javascript
  <%* 
    const fullPath = tp.file.folder(true); // 获取完整路径，例如 "Projects/ClientA/ProjectX"
    const pathArray = fullPath.split("/"); // 按斜杠分割路径
    const parentFolder = pathArray.slice(0, -1).join("/"); // 去掉最后一级，拼接为上级目录
    tR += parentFolder; // 输出上级目录
  %>
  ```

  **输出**：
  - 如果 `fullPath` 是 `Projects/ClientA/ProjectX`，则 `parentFolder` 输出为 `Projects/ClientA`。
  - 如果文件在根目录（`fullPath` 为空），则 `parentFolder` 输出为空字符串。

- **更简洁的写法**（仅获取上级目录名称）：
  如果只需要上级目录的名称（而不是完整路径），可以进一步处理：
  ```javascript
  <%* 
    const fullPath = tp.file.folder(true);
    const pathArray = fullPath.split("/");
    const parentFolderName = pathArray[pathArray.length - 2] || ""; // 获取倒数第二个元素
    tR += parentFolderName;
  %>
  ```

  **输出**：
  - 如果 `fullPath` 是 `Projects/ClientA/ProjectX`，则 `parentFolderName` 输出为 `ClientA`。
  - 如果文件在根目录或只有一级目录，则输出空字符串。

### 3. 结合使用示例
假设你有一个 Templater 模板，想在文件中自动插入当前目录和上级目录的名称，可以这样写：

```javascript
---
title: <% tp.file.title %>
folder: <% tp.file.folder() %>
parent_folder: <%* const fullPath = tp.file.folder(true); const pathArray = fullPath.split("/"); tR += pathArray[pathArray.length - 2] || "None"; %>
---
# <% tp.file.title %>
当前目录: <% tp.file.folder() %>
上级目录: <%* const fullPath = tp.file.folder(true); const pathArray = fullPath.split("/"); tR += pathArray[pathArray.length - 2] || "None"; %>
```

**输出示例**（文件路径为 `Projects/ClientA/ProjectX/Note.md`）：
```yaml
---
title: Note
folder: ProjectX
parent_folder: ClientA
---
# Note
当前目录: ProjectX
上级目录: ClientA
```

### 4.时间日期

如果你希望 date 和 lastmod 基于文件属性（例如创建时间或修改时间）动态生成，可以使用 tp.file.creation_date() 和 tp.file.last_modified_date()。以下是示例代码：

```
---
<%*
  const fullPath = tp.file.folder(true);
  const pathArray = fullPath.split("/");
  const categories = pathArray.slice(2, 4);

  // 动态生成日期（ISO 8601 格式）
  const date = tp.file.creation_date("YYYY-MM-DDTHH:mm:ss.SSSZ");
  const lastmod = tp.file.last_modified_date("YYYY-MM-DDTHH:mm:ss.SSSZ");

  // 输出 YAML
  tR += `date: ${date}\n`;
  tR += `lastmod: ${lastmod}\n`;
  tR += "categories:\n";
  if (categories.length === 0) {
    tR += "  - None\n";
  } else {
    for (const category of categories) {
      if (category) {
        tR += `  - ${category}\n`;
      }
    }
  }
%>
---
```

### 4. 注意事项
- **根目录情况**：如果文件位于 Vault 的根目录，`tp.file.folder(true)` 返回空字符串，需在代码中处理这种情况（例如返回 "None" 或其他默认值）。
- **路径分隔符**：Obsidian 使用 `/` 作为路径分隔符（即使在 Windows 上），因此无需担心平台差异。
- **调试**：如果结果不符合预期，可以在模板中加入 `console.log(fullPath)`，然后在 Obsidian 的开发者工具（Ctrl+Shift+I）中查看控制台输出。
- **插件版本**：确保 Templater 插件是最新版本，以避免潜在的路径处理问题。[](https://github.com/SilentVoid13/Templater/issues/1332)
- **性能**：对于大量文件操作，建议测试模板性能，避免过于复杂的路径处理逻辑。

### 5. 进阶：获取多级上级目录
如果需要获取更高级的目录（例如上两级目录），可以调整 `slice` 或索引：
```javascript
<%* 
  const fullPath = tp.file.folder(true);
  const pathArray = fullPath.split("/");
  const grandParentFolder = pathArray.slice(0, -2).join("/") || "None"; // 上两级目录
  tR += grandParentFolder;
%>
```

**输出**：
- 如果 `fullPath` 是 `Projects/ClientA/ProjectX`，则 `grandParentFolder` 输出为 `Projects`。



### 用更深的目录填充categories （更新）

我们可以根据目录管理categories

要为 Obsidian 中位于 `content/posts/A/B/C/exam.md` 的文件使用 Templater 插件生成 YAML frontmatter 中的 `categories` 字段，只包含 `A` 和 `B`（忽略 `content`, `posts`, 和 `C`），需要从文件路径中提取特定层级的目录。以下是实现方法和代码。

### 实现步骤
1. **获取文件路径**：使用 `tp.file.folder(true)` 获取文件的相对路径（`content/posts/A/B/C`）。
2. **提取所需目录**：将路径按 `/` 分割，移除 `content` 和 `posts`，然后从剩余路径中提取 `A` 和 `B`，忽略 `C`。
3. **生成 YAML 格式**：将提取的目录（`A` 和 `B`）格式化为 YAML 列表 `categories: [A, B]`。

### Templater 代码
以下是生成所需 `categories` 字段的 Templater 模板代码：

```javascript
---
<%*
  const fullPath = tp.file.folder(true); // 获取路径，例如 "content/posts/A/B/C"
  const pathArray = fullPath.split("/"); // 分割路径为 ["content", "posts", "A", "B", "C"]
  const categories = pathArray.slice(2, 4); // 提取 ["A", "B"]，跳过 "content" 和 "posts"，忽略 "C"
  tR += "categories:\n";
  if (categories.length === 0) {
    tR += "  - None\n"; // 如果没有符合条件的目录
  } else {
    for (const category of categories) {
      tR += `  - ${category}\n`; // 格式化为 YAML 列表
    }
  }
%>
---
```

### 输出结果
对于文件 `content/posts/A/B/C/exam.md`，上述代码将生成以下 YAML frontmatter：

```yaml
---
categories:
  - A
  - B
---
```

### 代码说明
- **`tp.file.folder(true)`**：返回文件的完整相对路径（`content/posts/A/B/C`）。
- **`pathArray.slice(2, 4)`**：
  - `slice(2, 4)` 从索引 2 开始（跳过 `content` 和 `posts`），到索引 4 结束（包含 `A` 和 `B`，排除 `C`）。
  - 结果是 `["A", "B"]`。
- **空路径处理**：如果 `categories` 为空（例如路径不包含 `A` 和 `B`），输出 `None` 作为默认值。
- **YAML 格式**：通过 `for` 循环生成 `- A` 和 `- B`，确保正确的 YAML 缩进（两个空格）。

### 处理不同路径深度的通用版本
如果文件路径可能变化（例如 `content/posts/A/B/exam.md` 或 `content/posts/A/B/C/D/exam.md`），可以用以下代码确保只提取 `A` 和 `B`（路径中第 3 和第 4 级目录）：

```javascript
---
<%*
  const fullPath = tp.file.folder(true); // 获取路径，例如 "content/posts/A/B/C"
  const pathArray = fullPath.split("/"); // 分割路径
  const categories = pathArray.slice(2, 4); // 提取第3和第4级目录（"A", "B"）
  tR += "categories:\n";
  if (categories.length === 0) {
    tR += "  - None\n"; // 如果没有足够的目录
  } else {
    for (const category of categories) {
      if (category) { // 确保目录非空
        tR += `  - ${category}\n`;
      }
    }
  }
%>
---
```

**输出示例**：
- 文件 `content/posts/A/B/C/exam.md`：
  ```yaml
  ---
  categories:
    - A
    - B
  ---
  ```
- 文件 `content/posts/A/B/exam.md`：
  ```yaml
  ---
  categories:
    - A
    - B
  ---
  ```
- 文件 `content/posts/A/exam.md`：
  ```yaml
  ---
  categories:
    - A
  ---
  ```
- 文件 `content/posts/exam.md`：
  ```yaml
  ---
  categories:
    - None
  ---
  ```

### 注意事项
1. **路径层级**：
   - 代码假定 `content` 和 `posts` 是固定的前两级目录，`A` 和 `B` 是第 3 和第 4 级。
   - 如果路径结构不同（例如 `content/A/B/exam.md`），需要调整 `slice(2, 4)` 为 `slice(1, 3)`。
1. **不足的目录**：官方自带的模板还是太弱了
   - 如果路径少于 4 级（例如 `content/posts/A/exam.md`），`slice(2, 4)` 只返回存在的目录（`["A"]`）。
   - 如果路径少于 3 级（例如 `content/posts/exam.md`），`categories` 为空，输出 `None`。
3. **空目录处理**：代码检查 `category` 是否为空，以避免输出空字符串。
4. **调试**：如果结果不符合预期，可以添加 `<% console.log(fullPath, pathArray, categories) %>`，在 Obsidian 开发者工具（Ctrl+Shift+I）中查看路径和数组内容。
5. **YAML 格式**：确保缩进为两个空格，符合 YAML 规范。
6. **Templater 配置**：确保 Templater 插件启用，且模板设置为自动触发（例如通过文件夹模板或手动插入）。

### 扩展：添加其他元数据
如果需要包含其他 frontmatter 字段（例如 `title` 或 `date`），可以扩展模板：

```javascript
---
title: <% tp.file.title %>
date: <% tp.date.now("YYYY-MM-DD") %>
<%*
  const fullPath = tp.file.folder(true);
  const pathArray = fullPath.split("/");
  const categories = pathArray.slice(2, 4);
  tR += "categories:\n";
  if (categories.length === 0) {
    tR += "  - None\n";
  } else {
    for (const category of categories) {
      if (category) {
        tR += `  - ${category}\n`;
      }
    }
  }
%>
---
```

**输出**（文件 `content/posts/A/B/C/exam.md`）：
```yaml
---
title: exam
date: 2025-04-18
categories:
  - A
  - B
---
```

### 总结
通过 `tp.file.folder(true)` 获取路径，使用 `slice(2, 4)` 提取 `A` 和 `B`，即可生成所需的 `categories` 字段。代码已考虑路径深度变化和边缘情况，适用于 `content/posts/A/B/C/exam.md` 及其变体。如果路径结构有其他特殊要求（例如动态跳过层级或包含其他目录），请提供更多细节，我可以进一步优化！

###  参考资源
- Templater 官方文档：`tp.file` 模块相关内容提供了路径操作的详细说明。[](https://silentvoid13.github.io/Templater/internal-functions/internal-modules/file-module.html)
- Obsidian 论坛讨论：关于获取上级目录的常见问题和解决方案。[](https://forum.obsidian.md/t/templater-get-the-parent-folder-name/47241)[](https://forum.obsidian.md/t/templater-how-to-reference-parent-folder/58232)
