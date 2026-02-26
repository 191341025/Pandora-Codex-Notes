# 模块四：代码提示工程

> 对应 PDF 第 42-53 页

---

## 概念地图

- **核心概念** (必须内化): 四种代码 prompting 场景（写/解释/翻译/调试）的 prompt 设计策略
- **实操要点** (动手时需要): 代码生成 prompt 的结构模式、调试 prompt 的信息要素
- **背景知识** (扩展理解): LLM 代码生成的局限性（不能推理、只是预测训练数据中的模式）

---

## 概念讲解

### 前言：LLM 写代码 ≠ LLM 理解代码

在开始之前需要校准一个期望：**LLM 生成代码的方式和它生成自然语言一样——预测最可能的下一个 token**。它不是真的"理解"了你的需求然后"设计"了一个方案，而是基于训练数据中见过的大量代码模式来预测。

这意味着：
- 生成的代码**很可能是对的**（因为训练数据中大量代码是正确的）
- 但**不保证是对的**——必须阅读和测试
- 对于常见任务（文件操作、数据处理等），效果很好
- 对于罕见/复杂的业务逻辑，可能需要多轮迭代

本模块按四个实际场景组织，每个场景都是一个完整的 prompt → 验证 → 使用 的 walk-through。

### 1. 写代码（Prompts for Writing Code）

**场景**：你有一个文件夹，里面几百个文件需要批量重命名——在文件名前加 `draft_` 前缀。你知道可以用 Bash 脚本，但不想花时间自己写。

**Prompt 设计**：

```
Goal:    Write code in Bash to rename files in a folder
Model:   gemini-pro
Temp:    0.1    Token Limit: 1024
Top-K:   N/A    Top-P: 1

Prompt:
Write a code snippet in Bash, which asks for a folder name.
Then it takes the contents of the folder and renames all the
files inside by prepending the name draft to the file name.
```

**模型输出**：

```bash
#!/bin/bash
# Ask for the folder name
echo "Enter the folder name: "
read folder_name

# Check if the folder exists
if [ ! -d "$folder_name" ]; then
  echo "Folder does not exist."
  exit 1
fi

# Get the list of files in the folder
files=( "$folder_name"/* )

# Rename each file by prepending "draft" to the file name
for file in "${files[@]}"; do
  new_file_name="draft_$(basename "$file")"
  mv "$file" "$new_file_name"
done

echo "Files renamed successfully."
```

**关键观察**：
- 模型不仅写了核心逻辑，还加了**输入验证**（检查文件夹是否存在）和**注释**
- Temperature 设为 0.1——代码生成需要确定性，不需要"创意"
- Token Limit 设为 1024——代码可能比自然语言长，需要足够的输出空间

**验证步骤（必须做）**：
1. **阅读代码**——理解逻辑是否符合预期
2. **先用测试数据跑**——建一个只有几个文件的测试文件夹
3. 确认无误后再在真实数据上运行

### 2. 解释代码（Prompts for Explaining Code）

**场景**：你接手了同事的代码，没有文档，需要快速理解逻辑。

**Prompt 设计要点**：把代码贴进去，要求模型解释。

```
Prompt:
Explain to me the below Bash code:
```
#!/bin/bash
echo "Enter the folder name: "
read folder_name
if [ ! -d "$folder_name" ]; then
  echo "Folder does not exist."
  exit 1
fi
files=( "$folder_name"/* )
for file in "${files[@]}"; do
  new_file_name="draft_$(basename "$file")"
  mv "$file" "$new_file_name"
done
echo "Files renamed successfully."
```
```

**模型输出**（结构化解释）：

```
1. User Input:
   - Prompts user to enter folder name, stores in folder_name

2. Folder Existence Check:
   - Uses [ ! -d "$folder_name" ] to verify folder exists
   - Exits with code 1 if not found

3. File Listing:
   - Creates array of all file paths: files=( "$folder_name"/* )

4. File Renaming:
   - Iterates through files array
   - Generates new name with "draft_" prefix using basename
   - Moves/renames using mv command

5. Success Message:
   - Confirms completion
```

**Prompt 技巧**：
- 如果代码有注释，可以**先去掉注释再让模型解释**——这样测试模型是否真正理解了逻辑，而不是复述注释
- 可以指定解释深度：`Explain at a beginner level` vs `Explain the design patterns used`
- 可以要求特定格式：`Explain step by step` 或 `Explain in a numbered list`

### 3. 翻译代码（Prompts for Translating Code）

**场景**：Bash 脚本能用，但你想把它做成一个有 UI 的应用——Python 更合适。

**Prompt 设计**：

```
Prompt:
Translate the below Bash code to a Python snippet.
```bash
[Bash 代码]
```
```

**模型输出**：

```python
import os
import shutil

# Get the folder name from the user
folder_name = input("Enter the folder name: ")

# Check if the folder exists
if not os.path.isdir(folder_name):
    print("Folder does not exist.")
    exit(1)

# Get a list of all files in the folder
files = os.listdir(folder_name)

# Iterate over the files
for file in files:
    # Create a new file name with the prefix "draft_"
    new_file_name = f"draft_{file}"

    # Move the file to the new name
    shutil.move(
        os.path.join(folder_name, file),
        os.path.join(folder_name, new_file_name)
    )

# Print a success message
print("Files renamed successfully.")
```

**关键观察**：
- 模型正确地把 Bash 的 `[ ! -d ]` 转成了 Python 的 `os.path.isdir()`
- `basename` → `os.path.join` 的路径处理也转换正确
- 保留了注释结构

> **注意**：在 Vertex AI Studio 中生成 Python 代码时，需要点击 **Markdown** 按钮查看输出，否则纯文本模式下缩进会丢失——对 Python 来说缩进丢失意味着代码无法运行。

**翻译代码的 prompt 技巧**：
- 明确源语言和目标语言：`Translate from Bash to Python`
- 如果有特定框架要求：`Translate to Python using Flask framework`
- 如果需要保持接口兼容：`Keep the same input/output behavior`

### 4. 调试和审查代码（Prompts for Debugging & Reviewing）

**场景**：你手动改了 Python 代码，引入了 bug，报了 `NameError`。

**有 bug 的代码**：

```python
import os
import shutil

folder_name = input("Enter the folder name: ")
prefix = input("Enter the string to prepend to the filename: ")
text = toUpperCase(prefix)  # ← Bug! Python 没有 toUpperCase

if not os.path.isdir(folder_name):
    print("Folder does not exist.")
    exit(1)

files = os.listdir(folder_name)
for file in files:
    new_filename = f"{text}_{file}"
    shutil.move(
        os.path.join(folder_name, file),
        os.path.joi(folder_name, new_file_name)  # ← 还有 typo!
    )
print("Files renamed successfully.")
```

**错误信息**：

```
NameError: name 'toUpperCase' is not defined
```

**Prompt 设计**——把错误信息 + 代码一起给模型：

```
Prompt:
The below Python code gives an error:
[错误信息]

Debug what's wrong and explain how I can improve the code.
[代码]
```

**模型输出的三个层次**：

1. **直接 bug 修复**：`toUpperCase(prefix)` → `prefix.upper()`（Python 用 `.upper()` 方法）

2. **发现更多 bug**（模型主动发现了你没注意到的问题）：
   - `os.path.joi` → `os.path.join`（拼写错误）
   - `new_file_name` → `new_filename`（变量名不一致）

3. **代码改进建议**：
   - 保留文件扩展名
   - 处理文件名中的空格
   - 加 try-except 错误处理
   - 输出改进后的完整代码

> **注意**：上面的原始示例中，模型的改进建议因 token limit 到达而被截断（输出末尾提示 "The response was truncated because it has reached the token limit"）。这再次说明：代码调试任务要给足够的 token limit，否则最有价值的改进建议可能被截断。

**调试 prompt 的关键要素**：

| 必须包含 | 说明 |
|----------|------|
| **错误信息** | 完整的 traceback，不要只贴最后一行 |
| **完整代码** | 不要只贴出错的那一行，模型需要看上下文 |
| **你的意图** | "Debug what's wrong" vs "Debug and suggest improvements" 得到的回答深度不同 |

| 可选但有用 | 说明 |
|------------|------|
| 运行环境 | Python 版本、操作系统 |
| 你已经尝试过什么 | 避免模型给出你已经试过的方案 |

> **常见误用**：只给模型看错误代码，不给错误信息。模型可能找到 bug，也可能找不到——给了错误信息，模型能直接定位问题。

### 四种场景的参数推荐

| 场景 | Temperature | Token Limit | 理由 |
|------|-------------|-------------|------|
| 写代码 | 0.1 | 1024+ | 代码需要确定性，留够输出空间 |
| 解释代码 | 0.1 | 1024 | 解释也需要准确 |
| 翻译代码 | 0.1 | 1024 | 翻译是精确映射，不需要创意 |
| 调试代码 | 0.1 | 1024+ | 调试建议可能很长，需要更多 token |

共同点：**所有代码任务 temperature 都应该设低**。代码不需要"创意"，需要"正确"。

---

## 重点标记

1. **LLM 生成的代码必须阅读和测试**：它是基于模式预测的，不是经过推理验证的。先用测试数据验证，再在真实环境运行。
2. **调试 prompt 要给完整上下文**：错误信息 + 完整代码 + 你的意图，三件套缺一不可。
3. **模型可以发现你没注意到的 bug**：调试 prompt 不只是修你报告的错误，模型会扫描整段代码并发现额外问题。
4. **代码任务 temperature 一律设低**：0.1 是安全起点，代码不需要创意，需要确定性。
5. **Python 代码注意输出格式**：在某些工具中纯文本输出会丢失缩进，务必用 Markdown 模式查看。

---

## 自测：你真的理解了吗？

**Q1**：你要让 LLM 写一个 Python 脚本来调用 REST API 并解析 JSON 响应。你的 prompt 应该包含哪些关键信息？Temperature 设多少？

**Q2**：你让模型调试代码，只给了代码但没给错误信息。模型给出了一个修改建议，但不是你遇到的那个 bug。问题出在哪？

**Q3**：你让模型把 JavaScript 代码翻译成 Python。输出看起来对，但运行时报了一个 Python 特有的缩进错误。可能的原因是什么？怎么避免？

**Q4**：模型生成了一段"看起来对"的代码，你直接部署到生产环境。这样做有什么风险？正确的流程应该是什么？
