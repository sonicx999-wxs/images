---
name: feishu-mtc-ops
version: 2.3.0
description: "飞书 MTC 模式操作技能：在 Trae MTC 模式下进行飞书文档操作（创建、获取、更新、搜索、媒体管理）、聊天操作（发送消息、获取聊天记录、创建群聊）、云空间操作（上传、下载、导出）、电子表格操作、多维表格操作、知识库操作等。默认以用户身份操作，支持应用身份创建文档后自动添加权限。包含首选方案和备用方案的完整执行流程。支持文件夹管理和默认目录配置。支持自然语言操作引导和智能错误处理。"
metadata:
  requires:
    bins: ["lark-cli"]
    mcps: ["mcp_lark-mcp"]
---

# feishu-mtc-ops (v2.3)

**飞书 MTC 模式操作技能** — 在 Trae MTC 模式下执行飞书全场景操作，默认使用用户身份，支持自动权限管理和备用方案切换。

---

## 零、快速开始（首次使用）

### 0.1 前置要求

本技能依赖 **飞书 MCP** 已配置。SOLO 会自动检测连接状态。

**检测方式**：尝试执行测试命令
```bash
lark-cli docs +search --query "测试"
```

- ✅ 成功返回 → MCP 已配置，直接使用
- ❌ 连接失败 → 执行 0.2 配置步骤

---

### 0.2 如未配置，引导用户

**检测到 MCP 未配置时**，在对话中发送以下引导（不要让用户手动操作）：

> 🔗 **检测到飞书未连接，需要简单配置**
>
> 请按以下步骤操作：
> 1. 进入 [Trae 后台 - 外部应用授权](https://console.trae.cn/marketplace)
> 2. 找到「飞书」应用，点击「授权」
> 3. 扫码绑定你的飞书账号
>
> 配置完成后，告诉我「已完成」，我会重新检测连接。

---

### 0.3 验证连接

配置完成后，执行验证命令：
```bash
lark-cli docs +search --query "验证"
```

返回包含 `ok: true` 即表示连接成功。

---

### 0.4 开始使用

连接成功后，你可以通过自然语言操作飞书：

| 你说 | AI 执行 |
|------|---------|
| "帮我创建一份周报" | 创建文档 → 返回链接 |
| "把这份文档放到工作文件夹" | 移动到文件夹 |
| "搜索关于 AI 的文档" | 搜索并返回结果 |
| "给产品组发条消息" | 发送飞书消息 |
| "在表格里追加一行数据" | 写入电子表格 |

---

## ⚠️ 重要：执行前必读

### MTC 模式特点

1. **凭证自动注入**：飞书 MCP 自动注入凭证，无需 `config init` 或 `auth login`
2. **固定 user 身份**：飞书 CLI 自动以 user 身份运行，**不要**添加 `--as user` 或 `--as bot` 参数
3. **配置命令不可用**：`config show`、`config init`、`auth login` 会返回 `external_provider` 错误，这是正常的，直接跳过即可

### 执行优先级

**文档/云空间操作**：
```
首选方案：飞书 CLI 命令（user 身份，文档归用户）
    ↓ 失败时
备用方案：run_mcp 调用飞书 MCP 工具（useUAT: false，文档归应用）
    ↓ 成功后
添加权限：为用户添加文档权限（drive_v1_permissionMember_create）
```

**聊天操作**：
```
首选方案：run_mcp 调用飞书 MCP 工具（应用身份，应用发消息给用户）
    ↓ 失败时
提示用户：检查应用权限或添加应用到群聊
```

> ⚠️ **关键区别**：文档操作首选 user 身份（文档归用户），聊天操作使用应用身份（应用发消息给用户）

---

## 一、文档操作

### 1.1 创建文档

#### 【首选方案】飞书 CLI 命令

**创建文档**：
```bash
lark-cli docs +create --title "文档标题" --markdown "# 内容

正文内容..."
```

**参数说明**：
- `--title`：文档标题
- `--markdown`：Markdown 格式内容

**成功判断**：返回 JSON 中 `ok: true` 且包含 `doc_id` 和 `doc_url`

> ⚠️ **注意**：`docs +create` 不支持直接指定文件夹。如需将文档放入指定文件夹，请先创建文档，再使用 `drive +move` 移动（见 [2.4 移动文件/文件夹](#24-移动文件文件夹)）。

**创建文档并移动到指定文件夹**：
```bash
# 步骤 1：创建文档
lark-cli docs +create --title "文档标题" --markdown "# 内容"

# 返回：{ "ok": true, "data": { "doc_id": "xxx" } }

# 步骤 2：移动到目标文件夹
lark-cli drive +move --file-token "文档ID" --folder-token "文件夹token" --type "docx"
```

---

#### 【备用方案】飞书 MCP 工具

**步骤 1**：创建文档
```json
{
  "server_name": "mcp_lark-mcp",
  "tool_name": "docx_builtin_import",
  "args": {
    "data": {
      "file_name": "文档标题",
      "markdown": "# 内容\n\n正文内容..."
    },
    "useUAT": false
  }
}
```

**步骤 2**：为用户添加权限（见 [权限管理](#三权限管理)）

---

### 1.2 获取文档内容

#### 【首选方案】飞书 CLI 命令

```bash
lark-cli docs +fetch --doc "文档ID或完整URL"
```

---

#### 【备用方案】飞书 MCP 工具

```json
{
  "server_name": "mcp_lark-mcp",
  "tool_name": "docx_v1_document_rawContent",
  "args": {
    "path": { "document_id": "文档ID" },
    "useUAT": true
  }
}
```

---

### 1.3 更新文档内容

#### 追加内容（append 模式）

```bash
lark-cli docs +update --doc "文档ID" --mode append --markdown "## 新章节

追加的内容..."
```

#### 替换章节（replace_range 模式）

```bash
lark-cli docs +update --doc "文档ID" --mode replace_range --selection-by-title "## 旧章节" --markdown "## 新章节

替换后的内容..."
```

#### 删除章节（delete_range 模式）

```bash
lark-cli docs +update --doc "文档ID" --mode delete_range --selection-by-title "## 要删除的章节"
```

#### 在指定位置插入

```bash
# 在指定内容前插入
lark-cli docs +update --doc "文档ID" --mode insert_before --selection-with-ellipsis "目标内容" --markdown "插入的内容"

# 在指定内容后插入
lark-cli docs +update --doc "文档ID" --mode insert_after --selection-with-ellipsis "目标内容" --markdown "插入的内容"
```

---

### 1.4 搜索文档

#### 【首选方案】飞书 CLI 命令

```bash
lark-cli docs +search --query "关键词" --page-size 10
```

---

#### 【备用方案】飞书 MCP 工具

```json
{
  "server_name": "mcp_lark-mcp",
  "tool_name": "docx_builtin_search",
  "args": {
    "data": { "search_key": "关键词", "count": 10 },
    "useUAT": true
  }
}
```

---

### 1.5 插入图片/文件到文档

#### 【首选方案】飞书 CLI 命令

```bash
lark-cli docs +media-insert --doc "文档ID" --file "/本地文件路径.png"
```

**参数说明**：
- `--doc`：文档 ID 或 URL
- `--file`：本地文件路径（支持图片、附件等）
- `--type`：文件类型（可选，自动检测）

**成功示例**：
```json
{
  "ok": true,
  "data": {
    "file_token": "xxx",
    "message": "媒体文件插入成功"
  }
}
```

---

### 1.6 下载文档中的媒体文件

```bash
lark-cli docs +media-download --doc "文档ID" --file-token "文件token" --output "/保存路径"
```

---

### 1.7 预览文档中的媒体文件

```bash
lark-cli docs +media-preview --doc "文档ID" --file-token "文件token"
```

---

### 1.8 自然语言意图解析

当用户使用自然语言描述需求时，自动解析为对应命令：

| 用户说 | AI 解析并执行 |
|--------|---------------|
| "帮我创建一份周报" | `lark-cli docs +create --title "周报" --markdown "..."` |
| "搜索一下关于 AI 的文档" | `lark-cli docs +search --query "AI"` |
| "看看这份文档内容" | `lark-cli docs +fetch --doc "文档ID"` |
| "在文档里加一段总结" | `lark-cli docs +update --doc "文档ID" --mode append --markdown "总结内容"` |
| "把图片插入到文档" | `lark-cli docs +media-insert --doc "文档ID" --file "图片路径"` |
| "搜索飞书里的表格" | `lark-cli docs +search --query "关键词" --type sheet` |

---

## 二、云空间操作

### 2.1 上传文件

```bash
lark-cli drive +upload --file "/本地文件路径" --folder "文件夹token"
```

**参数说明**：
- `--file`：本地文件路径
- `--folder`：目标文件夹 token（可选，默认根目录）
- `--name`：上传后的文件名（可选）
- `--type`：文件类型（可选）

---

### 2.2 下载文件

```bash
lark-cli drive +download --token "文件token" --output "/本地保存路径"
```

**参数说明**：
- `--token`：文件 token 或 URL
- `--output`：本地保存路径
- `--type`：文件类型（可选）

---

### 2.3 创建文件夹

```bash
lark-cli drive +create-folder --name "文件夹名称" --parent "父文件夹token"
```

**参数说明**：
- `--name`：文件夹名称
- `--parent`：父文件夹 token（可选，默认根目录）

---

### 2.4 移动文件/文件夹

```bash
lark-cli drive +move --file-token "文件token" --folder-token "目标文件夹token" --type "docx"
```

**参数说明**：
- `--file-token`：要移动的文件/文件夹 token
- `--folder-token`：目标文件夹 token
- `--type`：文件类型（file/docx/sheet/bitable 等）

**成功示例**：
```json
{
  "ok": true,
  "data": {
    "file_token": "xxx",
    "folder_token": "目标文件夹token",
    "type": "docx"
  }
}
```

---

### 2.5 导出文档

```bash
lark-cli drive +export --token "文档token" --type "pdf" --output "/保存路径"
```

**参数说明**：
- `--token`：文档 token 或 URL
- `--type`：导出格式（pdf/docx/xlsx/html 等）
- `--output`：本地保存路径

**成功示例**：
```json
{
  "ok": true,
  "data": {
    "ticket": "xxx",
    "message": "导出任务已创建"
  }
}
```

> **注意**：导出是异步操作，需要使用 `drive +task_result` 查询结果

---

### 2.6 导入本地文件为云文档

```bash
lark-cli drive +import --file "/本地文件.docx" --folder "文件夹token"
```

**参数说明**：
- `--file`：本地文件路径
- `--folder`：目标文件夹 token（可选）
- `--type`：目标文档类型（docx/sheet/bitable 等，可选）

---

### 2.7 添加文档评论

```bash
lark-cli drive +add-comment --token "文档token" --content "评论内容"
```

**参数说明**：
- `--token`：文档 token 或 URL
- `--content`：评论内容
- `--type`：文档类型（可选）

---

### 2.8 申请文档权限

```bash
lark-cli drive +apply-permission --token "文档token" --perm "edit" --remark "申请理由"
```

**参数说明**：
- `--token`：文档 token 或 URL
- `--perm`：申请的权限类型（view/edit）
- `--remark`：申请备注（可选）

---

### 2.9 创建快捷方式

```bash
lark-cli drive +create-shortcut --file-token "源文件token" --folder-token "目标文件夹token" --type "docx"
```

**参数说明**：
- `--file-token`：源文件 token
- `--folder-token`：目标文件夹 token
- `--type`：源文件类型（file/docx/sheet/bitable 等）

---

### 2.10 查询异步任务结果

```bash
# 查询导出任务
lark-cli drive +task_result --scenario "export" --ticket "任务ticket"

# 查询导入任务
lark-cli drive +task_result --scenario "import" --ticket "任务ticket"

# 查询移动/删除任务
lark-cli drive +task_result --scenario "task_check" --task-id "任务ID"
```

**参数说明**：
- `--scenario`：任务场景（import/export/task_check/wiki_move/wiki_delete_space）
- `--ticket`：导入/导出任务的 ticket
- `--task-id`：其他任务的 ID
- `--file-token`：用于查询导出状态的文档 token

---

### 2.11 文件夹管理

> 💡 **核心功能**：创建文件夹、搜索文件夹、设置默认目录，用于组织文档结构。

#### 2.11.1 创建文件夹

```bash
# 在根目录创建文件夹
lark-cli drive +create-folder --name "文件夹名称"

# 在指定文件夹下创建子文件夹
lark-cli drive +create-folder --name "子文件夹名称" --parent "父文件夹token"
```

**参数说明**：
- `--name`：文件夹名称
- `--parent`：父文件夹 token（可选，默认根目录）

**成功示例**：
```json
{
  "ok": true,
  "data": {
    "token": "fldcnxxxxxxxxxx",
    "name": "文件夹名称",
    "type": "folder"
  }
}
```

> 📌 **重要**：创建成功后，**记录返回的 `token`**。后续创建文档时，需先创建文档再使用 `drive +move` 移动到此文件夹。

---

#### 2.11.2 搜索文件夹

**方法一：通过文档搜索间接定位**
```bash
lark-cli docs +search --query "已知文档标题"
```

返回结果中的 `parent_token` 即为文档所在文件夹的 token。

**方法二：通过 Wiki 知识库定位**
```json
{
  "server_name": "mcp_lark-mcp",
  "tool_name": "wiki_v2_space_getNode",
  "args": {
    "params": {
      "token": "wiki节点token"
    }
  }
}
```

---

#### 2.11.3 默认目录配置

**推荐做法**：在会话中维护一个默认文件夹 token，避免每次创建文档时重复指定。

**配置存储位置**：
- **临时存储**：在当前会话的上下文中保存（会话结束后失效）
- **持久存储**：写入到本地配置文件 `/data/user/work/feishu-default-folder.json`

**配置文件格式**：
```json
{
  "default_folder_token": "fldcnxxxxxxxxxx",
  "default_folder_name": "SOLO生成文档",
  "created_at": "2024-01-01T00:00:00Z"
}
```

**使用流程**：
```
1. 首次使用时创建或指定默认文件夹
   └─ lark-cli drive +create-folder --name "SOLO生成文档"
   └─ 记录返回的 token

2. 后续创建文档时自动使用默认文件夹
   └─ lark-cli docs +create --title "标题" --markdown "内容"
   └─ lark-cli drive +move --file-token "文档ID" --folder-token "默认token" --type "docx"

3. 如需更换默认文件夹，重新执行步骤 1 并更新配置
```

---

#### 2.11.4 完整工作流示例

**场景**：创建一个"工作日志"文件夹，并在其中创建日报文档

```bash
# 步骤 1：创建文件夹
lark-cli drive +create-folder --name "工作日志"

# 返回：{ "ok": true, "data": { "token": "fldcnabc123" } }

# 步骤 2：创建文档
lark-cli docs +create --title "2024-01-15 日报" --markdown "
# 今日工作

- 完成任务 A
- 完成任务 B

# 明日计划

- 开始任务 C
"

# 返回：{ "ok": true, "data": { "doc_id": "LU8Pdxxx" } }

# 步骤 3：移动文档到文件夹
lark-cli drive +move --file-token "LU8Pdxxx" --folder-token "fldcnabc123" --type "docx"

# 返回：{ "ok": true, "data": { "file_token": "LU8Pdxxx", "folder_token": "fldcnabc123" } }
```

---

## 三、聊天操作

> ⚠️ **重要**：聊天操作应使用**应用身份**（useUAT: false），因为消息是由应用发送给用户的。

### 3.1 发送消息

#### 【首选方案】飞书 MCP 工具（应用身份）

```json
{
  "server_name": "mcp_lark-mcp",
  "tool_name": "im_v1_message_create",
  "args": {
    "data": {
      "receive_id": "聊天ID",
      "msg_type": "text",
      "content": "{\"text\":\"消息内容\"}"
    },
    "params": { "receive_id_type": "chat_id" }
  }
}
```

**注意**：
- `content` 必须是 JSON 字符串格式
- `receive_id_type` 可选值：`chat_id`、`open_id`、`user_id`、`email`

---

### 3.2 获取聊天列表

```json
{
  "server_name": "mcp_lark-mcp",
  "tool_name": "im_v1_chat_list",
  "args": {
    "data": { "count": 20 }
  }
}
```

---

### 3.3 获取聊天记录

```json
{
  "server_name": "mcp_lark-mcp",
  "tool_name": "im_v1_message_list",
  "args": {
    "params": {
      "container_id": "聊天ID",
      "container_id_type": "chat",
      "page_size": 20,
      "sort_type": "ByCreateTimeDesc"
    }
  }
}
```

**提取用户 open_id**：`items[0].sender.id` 即为用户的 open_id

---

### 3.4 创建群聊

```json
{
  "server_name": "mcp_lark-mcp",
  "tool_name": "im_v1_chat_create",
  "args": {
    "data": {
      "name": "群聊名称",
      "chat_mode": "group",
      "chat_type": "private"
    },
    "params": { "user_id_type": "open_id" }
  }
}
```

---

### 3.5 获取群成员列表

```json
{
  "server_name": "mcp_lark-mcp",
  "tool_name": "im_v1_chatMembers_get",
  "args": {
    "path": { "chat_id": "群聊ID" },
    "params": {
      "member_id_type": "open_id",
      "page_size": 50
    }
  }
}
```

**成功示例**：
```json
{
  "has_more": false,
  "items": [
    {
      "member_id": "ou_xxx",
      "member_id_type": "open_id",
      "name": "用户名"
    }
  ]
}
```

---

## 四、知识库操作

### 4.1 搜索知识库节点

#### 【首选方案】飞书 MCP 工具

```json
{
  "server_name": "mcp_lark-mcp",
  "tool_name": "wiki_v1_node_search",
  "args": {
    "data": {
      "query": "关键词",
      "space_id": "知识空间ID"
    },
    "params": { "page_size": 20 }
  }
}
```

**参数说明**：
- `query`：搜索关键词（必填）
- `space_id`：知识空间 ID（可选，不填则搜索全部）
- `node_id`：父节点 ID（可选，搜索子节点）

---

### 4.2 获取知识库节点详情（解析 Wiki URL）

#### 【首选方案】飞书 MCP 工具

```json
{
  "server_name": "mcp_lark-mcp",
  "tool_name": "wiki_v2_space_getNode",
  "args": {
    "params": {
      "token": "wiki节点token或文档token",
      "obj_type": "docx"
    }
  }
}
```

**参数说明**：
- `token`：Wiki URL 中的 token 或实际文档 token
- `obj_type`：文档类型（doc/docx/sheet/bitable/file/slides/mindnote/wiki），当传入实际文档 token 时必填

**成功示例**：
```json
{
  "node": {
    "obj_type": "docx",
    "obj_token": "实际文档token",
    "title": "文档标题",
    "node_type": "origin",
    "space_id": "知识空间ID"
  }
}
```

> **重要**：Wiki URL 中的 token 不等于文档 token，必须通过此接口获取真实的 `obj_token`

---

### 4.3 创建知识库节点

```bash
lark-cli wiki +node-create --space "知识空间ID" --parent "父节点token" --obj-type "docx" --obj-token "文档token"
```

**参数说明**：
- `--space`：知识空间 ID
- `--parent`：父节点 token（可选）
- `--obj-type`：对象类型（docx/sheet/bitable/file/wiki）
- `--obj-token`：文档 token（创建文档节点时必填）
- `--title`：节点标题（可选）

---

## 五、电子表格操作

> ⚠️ **重要**：电子表格操作需要使用 `sheet_id`（工作表ID），格式为 `<sheetId>!A1:D10`。可使用 `sheets +info` 获取 sheet_id。

### 5.1 创建电子表格

```bash
lark-cli sheets +create --title "表格名称"
```

**参数说明**：
- `--title`：表格名称

**成功示例**：
```json
{
  "ok": true,
  "data": {
    "spreadsheet_token": "xxx",
    "url": "https://www.feishu.cn/sheets/xxx"
  }
}
```

> ⚠️ **注意**：`sheets +create` 不支持直接指定文件夹。如需将表格放入指定文件夹，请使用 `drive +move` 移动。

---

### 5.2 获取表格信息（获取 sheet_id）

```bash
lark-cli sheets +info --spreadsheet-token "表格token"
```

**成功示例**：
```json
{
  "ok": true,
  "data": {
    "sheets": {
      "sheets": [
        {
          "sheet_id": "0841af",
          "title": "Sheet1",
          "grid_properties": {
            "row_count": 200,
            "column_count": 20
          }
        }
      ]
    }
  }
}
```

> 📌 **重要**：记录返回的 `sheet_id`（如 `0841af`），后续读写操作需要使用此 ID。

---

### 5.3 读取单元格值

```bash
lark-cli sheets +read --spreadsheet-token "表格token" --range "<sheetId>!A1:C10"
```

**参数说明**：
- `--spreadsheet-token`：表格 token
- `--range`：读取范围，**必须包含 sheet_id**（格式：`<sheetId>!A1:C10`）

**示例**：
```bash
lark-cli sheets +read --spreadsheet-token "EdzRsq9q3hU8pXteWwYcnk7Nngb" --range "0841af!A1:D10"
```

---

### 5.4 写入单元格值

```bash
lark-cli sheets +write --spreadsheet-token "表格token" --range "<sheetId>!A1" --values '[["姓名","年龄"],["张三",25]]'
```

**参数说明**：
- `--spreadsheet-token`：表格 token
- `--range`：写入范围，**必须包含 sheet_id**（格式：`<sheetId>!A1:B2`）
- `--values`：写入数据（JSON 数组格式）

**示例**：
```bash
lark-cli sheets +write --spreadsheet-token "EdzRsq9q3hU8pXteWwYcnk7Nngb" --range "0841af!A1:B2" --values '[["姓名","年龄"],["张三",25]]'
```

---

### 5.5 追加行数据

```bash
lark-cli sheets +append --spreadsheet-token "表格token" --range "<sheetId>!A:D" --values '[["李四",30]]'
```

**参数说明**：
- `--spreadsheet-token`：表格 token
- `--range`：追加范围，**必须包含 sheet_id**（格式：`<sheetId>!A:D`）
- `--values`：追加数据（JSON 数组格式）

**示例**：
```bash
lark-cli sheets +append --spreadsheet-token "EdzRsq9q3hU8pXteWwYcnk7Nngb" --range "0841af!A:D" --values '[["1","张三","研发部","通过"],["2","李四","产品部","通过"]]'
```

**成功示例**：
```json
{
  "ok": true,
  "data": {
    "updates": {
      "updatedCells": 8,
      "updatedRows": 2,
      "updatedRange": "0841af!A1:D2"
    }
  }
}
```

---

## 六、多维表格操作

> ⚠️ **重要**：多维表格操作使用**应用身份**（useUAT: false）

### 6.1 创建多维表格应用

```json
{
  "server_name": "mcp_lark-mcp",
  "tool_name": "bitable_v1_app_create",
  "args": {
    "data": {
      "name": "多维表格名称",
      "folder_token": "文件夹token"
    },
    "useUAT": false
  }
}
```

**参数说明**：
- `name`：多维表格名称
- `folder_token`：目标文件夹 token（可选，默认根目录）
- `time_zone`：时区（可选）

**成功示例**：
```json
{
  "app": {
    "app_token": "xxx",
    "name": "多维表格名称"
  }
}
```

---

### 6.2 创建数据表

```json
{
  "server_name": "mcp_lark-mcp",
  "tool_name": "bitable_v1_appTable_create",
  "args": {
    "path": { "app_token": "多维表格token" },
    "data": {
      "table": {
        "name": "数据表名称",
        "default_view_name": "默认视图",
        "fields": [
          { "field_name": "姓名", "type": 1 },
          { "field_name": "年龄", "type": 2 }
        ]
      }
    },
    "useUAT": false
  }
}
```

**字段类型（type）**：
| 值 | 类型 |
|----|------|
| 1 | 多行文本 |
| 2 | 数字 |
| 3 | 单选 |
| 4 | 多选 |
| 5 | 日期 |
| 7 | 复选框 |
| 11 | 人员 |
| 15 | 超链接 |
| 17 | 附件 |

---

### 6.3 列出数据表

```json
{
  "server_name": "mcp_lark-mcp",
  "tool_name": "bitable_v1_appTable_list",
  "args": {
    "path": { "app_token": "多维表格token" },
    "useUAT": false
  }
}
```

---

### 6.4 列出字段

```json
{
  "server_name": "mcp_lark-mcp",
  "tool_name": "bitable_v1_appTableField_list",
  "args": {
    "path": {
      "app_token": "多维表格token",
      "table_id": "数据表ID"
    },
    "useUAT": false
  }
}
```

---

### 6.5 创建记录

```json
{
  "server_name": "mcp_lark-mcp",
  "tool_name": "bitable_v1_appTableRecord_create",
  "args": {
    "path": {
      "app_token": "多维表格token",
      "table_id": "数据表ID"
    },
    "data": {
      "fields": {
        "姓名": "张三",
        "年龄": 25
      }
    },
    "useUAT": false
  }
}
```

---

### 6.6 搜索记录

```json
{
  "server_name": "mcp_lark-mcp",
  "tool_name": "bitable_v1_appTableRecord_search",
  "args": {
    "path": {
      "app_token": "多维表格token",
      "table_id": "数据表ID"
    },
    "data": {
      "field_names": ["姓名", "年龄"],
      "filter": {
        "conjunction": "and",
        "conditions": [
          { "field_name": "年龄", "operator": "isGreater", "value": ["20"] }
        ]
      }
    },
    "params": { "page_size": 100 },
    "useUAT": false
  }
}
```

**过滤操作符（operator）**：
| 值 | 说明 |
|----|------|
| is | 等于 |
| isNot | 不等于 |
| contains | 包含 |
| doesNotContain | 不包含 |
| isEmpty | 为空 |
| isNotEmpty | 不为空 |
| isGreater | 大于 |
| isLess | 小于 |

---

### 6.7 更新记录

```json
{
  "server_name": "mcp_lark-mcp",
  "tool_name": "bitable_v1_appTableRecord_update",
  "args": {
    "path": {
      "app_token": "多维表格token",
      "table_id": "数据表ID",
      "record_id": "记录ID"
    },
    "data": {
      "fields": {
        "年龄": 26
      }
    },
    "useUAT": false
  }
}
```

---

## 七、通讯录操作

### 7.1 获取用户信息

```bash
lark-cli contact +get-user --user-id "用户open_id"
```

**参数说明**：
- `--user-id`：用户 ID（open_id/union_id/user_id）
- `--user-id-type`：用户 ID 类型（可选，默认 open_id）

---

### 7.2 搜索用户

```bash
lark-cli contact +search-user --query "用户名或邮箱"
```

**参数说明**：
- `--query`：搜索关键词（姓名/邮箱/手机号）
- `--page-size`：返回数量（可选）

---

### 7.3 通过邮箱获取用户 ID

```json
{
  "server_name": "mcp_lark-mcp",
  "tool_name": "contact_v3_user_batchGetId",
  "args": {
    "data": { "emails": ["user@example.com"] },
    "params": { "user_id_type": "open_id" }
  }
}
```

---

## 八、权限管理

### 8.1 为用户添加文档权限

**使用场景**：
- 使用备用方案（应用身份）创建文档后
- 需要编辑他人创建的文档时

```json
{
  "server_name": "mcp_lark-mcp",
  "tool_name": "drive_v1_permissionMember_create",
  "args": {
    "data": {
      "token": "文档ID",
      "type": "file",
      "member_type": "openid",
      "member_id": "用户open_id",
      "perm": "full_access"
    }
  }
}
```

### 8.2 权限级别说明

| 权限级别 | 说明 | 使用场景 |
|---------|------|---------|
| `view` | 仅查看 | 只需要用户能查看文档 |
| `edit` | 可编辑 | 需要用户能编辑文档 |
| `full_access` | 可管理 | 需要用户能编辑、分享、删除文档 |

### 8.3 获取用户 open_id

**方法一**：从聊天记录中获取（见 [3.3 获取聊天记录](#33-获取聊天记录)）

**方法二**：通过邮箱查询（见 [7.3 通过邮箱获取用户 ID](#73-通过邮箱获取用户-id)）

**方法三**：从群成员列表获取（见 [3.5 获取群成员列表](#35-获取群成员列表)）

---

## 九、错误处理与备用方案

### 9.1 常见错误及处理

| 错误信息 | 含义 | 自动处理方案 |
|---------|------|-------------|
| `user_access_token is invalid or expired` | 用户令牌过期 | 切换到飞书 MCP（useUAT: false） |
| `forbidden` 或错误码 1770032 | 无编辑权限 | 添加权限后重试 |
| `Access denied. One of the following scopes is required` | 应用缺少权限 | 提示用户在开发者后台开通权限 |
| `external_provider` | MTC 模式不支持配置命令 | 跳过配置，直接使用飞书 CLI 命令 |
| `--as bot is not supported` | 搜索不支持 bot 身份 | 使用 user 身份 |
| `permission denied` | 无操作权限 | 申请权限或切换身份 |
| `file not found` | 文件不存在 | 搜索确认文件位置 |

### 9.2 智能错误处理流程

```
当操作失败时，AI 自动执行以下逻辑：

┌─────────────────────────────────────────────────────────────┐
│                      智能错误处理流程                          │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  1. 分析错误类型                                             │
│     │                                                        │
│     ├─ 令牌过期/无效                                        │
│     │   └─→ 自动切换到 MCP（useUAT: false）                  │
│     │      └─→ 成功后为用户添加权限                           │
│     │                                                        │
│     ├─ 权限不足                                              │
│     │   └─→ 尝试添加权限                                     │
│     │      └─→ 重试原操作                                    │
│     │                                                        │
│     ├─ 缺少 scope                                            │
│     │   └─→ 提供权限申请链接                                 │
│     │      └─→ 等待用户完成后再重试                          │
│     │                                                        │
│     └─ 其他错误                                              │
│         └─→ 记录错误，提供详细说明                           │
│            └─→ 询问用户是否尝试备用方案                       │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### 9.3 备用方案执行流程

```
┌─────────────────────────────────────────────────────────────┐
│                    操作执行流程                               │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  1. 尝试首选方案（飞书 CLI）                                  │
│     ├── 成功 → 返回结果，结束                                 │
│     └── 失败 → 分析错误，进入步骤 2                           │
│                                                             │
│  2. 分析错误类型                                             │
│     ├── 令牌过期 → 执行步骤 3A                               │
│     ├── 权限不足 → 执行步骤 3B                               │
│     └── 其他错误 → 执行步骤 3A                               │
│                                                             │
│  3A. 切换到飞书 MCP（useUAT: false）                         │
│     ├── 成功 → 为用户添加权限 → 返回结果                      │
│     └── 失败 → 提示用户检查配置                               │
│                                                             │
│  3B. 添加权限后重试首选方案                                   │
│     ├── 成功 → 返回结果                                       │
│     └── 失败 → 执行步骤 3A                                   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## 十、快速参考卡

### 自然语言操作指南

| 你说 | AI 执行 |
|------|---------|
| "帮我创建一份周报" | `docs +create` → 返回文档链接 |
| "把这份文档放到工作文件夹" | `drive +move` → 移动到文件夹 |
| "搜索关于 AI 的文档" | `docs +search` → 返回搜索结果 |
| "给产品组发条消息" | MCP `im_v1_message_create` |
| "在表格里追加一行数据" | `sheets +append` → 写入数据 |
| "看看我有哪些待办" | `contact +search-user` 查询任务 |

### 飞书 CLI 命令（文档/云空间操作首选）

| 操作 | 命令 |
|------|------|
| 创建文档 | `lark-cli docs +create --title "标题" --markdown "内容"` |
| 获取文档 | `lark-cli docs +fetch --doc "文档ID"` |
| 追加内容 | `lark-cli docs +update --doc "文档ID" --mode append --markdown "内容"` |
| 替换章节 | `lark-cli docs +update --doc "文档ID" --mode replace_range --selection-by-title "## 章节" --markdown "新内容"` |
| 删除章节 | `lark-cli docs +update --doc "文档ID" --mode delete_range --selection-by-title "## 章节"` |
| 搜索文档 | `lark-cli docs +search --query "关键词"` |
| 插入图片 | `lark-cli docs +media-insert --doc "文档ID" --file "本地路径"` |
| 下载媒体 | `lark-cli docs +media-download --doc "文档ID" --file-token "token"` |
| 上传文件 | `lark-cli drive +upload --file "本地路径" --folder-token "文件夹token"` |
| 下载文件 | `lark-cli drive +download --file-token "文件token" --output "保存路径"` |
| 创建文件夹 | `lark-cli drive +create-folder --name "名称"` |
| **移动文件** | `lark-cli drive +move --file-token "文件token" --folder-token "目标token" --type "docx"` |
| 导出文档 | `lark-cli drive +export --file-token "文档token" --type "pdf"` |
| 导入文件 | `lark-cli drive +import --file "本地路径"` |
| 添加评论 | `lark-cli drive +add-comment --file-token "文档token" --content "评论"` |
| 申请权限 | `lark-cli drive +apply-permission --file-token "文档token" --perm "edit"` |
| 创建快捷方式 | `lark-cli drive +create-shortcut --file-token "源token" --folder-token "目标token"` |
| 查询任务 | `lark-cli drive +task_result --scenario "export" --ticket "ticket"` |
| 创建知识库节点 | `lark-cli wiki +node-create --space "空间ID" --obj-token "文档token"` |
| 创建表格 | `lark-cli sheets +create --title "名称"` |
| **获取表格信息** | `lark-cli sheets +info --spreadsheet-token "表格token"` |
| **读取单元格** | `lark-cli sheets +read --spreadsheet-token "表格token" --range "<sheetId>!A1:C10"` |
| **写入单元格** | `lark-cli sheets +write --spreadsheet-token "表格token" --range "<sheetId>!A1" --values '[[...]]'` |
| **追加行** | `lark-cli sheets +append --spreadsheet-token "表格token" --range "<sheetId>!A:D" --values '[[...]]'` |
| 获取用户 | `lark-cli contact +get-user --user-id "open_id"` |
| 搜索用户 | `lark-cli contact +search-user --query "关键词"` |

### run_mcp 调用（聊天操作首选 / 文档操作备用）

| 操作 | tool_name | 关键参数 |
|------|-----------|---------|
| 创建文档（备用） | `docx_builtin_import` | `data.file_name`, `data.markdown`, `useUAT: false` |
| 获取文档（备用） | `docx_v1_document_rawContent` | `path.document_id` |
| 搜索文档（备用） | `docx_builtin_search` | `data.search_key`, `useUAT: true` |
| **发送消息（首选）** | `im_v1_message_create` | `data.receive_id`, `data.msg_type`, `data.content` |
| **获取聊天记录（首选）** | `im_v1_message_list` | `params.container_id`, `params.container_id_type` |
| **获取聊天列表（首选）** | `im_v1_chat_list` | `data.count` |
| **创建群聊（首选）** | `im_v1_chat_create` | `data.name`, `data.chat_mode` |
| **获取群成员（首选）** | `im_v1_chatMembers_get` | `path.chat_id` |
| **搜索知识库（首选）** | `wiki_v1_node_search` | `data.query`, `data.space_id` |
| **获取知识库节点（首选）** | `wiki_v2_space_getNode` | `params.token`, `params.obj_type` |
| **创建多维表格** | `bitable_v1_app_create` | `data.name`, `useUAT: false` |
| **创建数据表** | `bitable_v1_appTable_create` | `path.app_token`, `data.table` |
| **列出数据表** | `bitable_v1_appTable_list` | `path.app_token` |
| **列出字段** | `bitable_v1_appTableField_list` | `path.app_token`, `path.table_id` |
| **创建记录** | `bitable_v1_appTableRecord_create` | `path.app_token`, `path.table_id`, `data.fields` |
| **搜索记录** | `bitable_v1_appTableRecord_search` | `path.app_token`, `path.table_id` |
| **更新记录** | `bitable_v1_appTableRecord_update` | `path.app_token`, `path.table_id`, `path.record_id` |
| 添加权限 | `drive_v1_permissionMember_create` | `data.token`, `data.member_id`, `data.perm` |
| 获取用户ID | `contact_v3_user_batchGetId` | `data.emails`, `params.user_id_type` |

---

## 十二、场景化工作流

### 场景 1：每日日报自动生成与推送

**需求**：每天早上自动生成日报，保存到飞书并发送通知

```bash
# 步骤 1：创建日报文档
lark-cli docs +create --title "$(date '+%Y-%m-%d') 日报" --markdown "
# 今日工作

（自动生成）

# 明日计划

（自动生成）
"

# 步骤 2：移动到日报文件夹
lark-cli drive +move --file-token "文档ID" --folder-token "日报文件夹token" --type "docx"

# 步骤 3：发送通知
# 使用 MCP 发送消息到用户
```

**完整流程**：
```
1. WebSearch(今日科技/行业新闻)
2. AI 整理生成日报内容
3. 创建飞书文档
4. 移动到指定文件夹
5. 发送飞书消息通知用户
```

---

### 场景 2：会议纪要快速整理

**需求**：从群聊记录提取关键信息，生成纪要并存入知识库

```bash
# 步骤 1：获取群聊记录
# im_v1_message_list 获取最近消息

# 步骤 2：AI 提取关键信息并生成纪要

# 步骤 3：创建纪要文档
lark-cli docs +create --title "会议纪要-XXX" --markdown "..."

# 步骤 4：存入知识库
lark-cli wiki +node-create --space "知识空间ID" --obj-token "文档ID"
```

---

### 场景 3：数据报表自动化更新

**需求**：定期从数据库提取数据，生成报表并存入飞书表格

```bash
# 步骤 1：获取/创建电子表格
lark-cli sheets +create --title "销售报表"
# 或：lark-cli docs +search --query "销售报表"

# 步骤 2：获取 sheet_id
lark-cli sheets +info --spreadsheet-token "表格token"

# 步骤 3：写入数据
lark-cli sheets +write --spreadsheet-token "表格token" --range "0841af!A1:D10" --values '[[...]]'

# 步骤 4：追加新数据
lark-cli sheets +append --spreadsheet-token "表格token" --range "0841af!A:D" --values '[[...]]'
```

---

### 场景 4：文档批量处理

**需求**：批量将本地 Markdown 文件上传到飞书指定文件夹

```bash
# 遍历本地文件
for file in *.md; do
  # 导入为飞书文档
  lark-cli drive +import --file "./$file" --type docx
done

# 移动到目标文件夹
lark-cli drive +move --file-token "文件token" --folder-token "目标文件夹" --type "docx"
```

---

### 场景 5：定时任务自动播报

**需求**：每天定时推送资讯简报到飞书群

```bash
# 在定时任务中执行：
# 1. 搜索最新资讯
# lark-cli docs +search --query "AI"

# 2. 生成简报
# AI 整理内容

# 3. 创建文档
lark-cli docs +create --title "每日简报-$(date '+%Y-%m-%d')" --markdown "..."

# 4. 发送群消息（使用 MCP）
# im_v1_message_create 发送通知到群
```

---

## 十三、注意事项

1. **MTC 模式固定 user 身份**：飞书 CLI 自动以 user 身份运行，**不要**添加 `--as user` 或 `--as bot`
2. **文档归属**：user 身份创建的文档归用户，useUAT: false 创建的文档归应用
3. **聊天操作使用应用身份**：消息发送、聊天记录获取等操作应使用飞书 MCP 工具（应用身份）
4. **权限管理**：使用应用身份创建文档后，必须为用户添加权限
5. **错误判断**：检查返回 JSON 中的 `ok` 字段或 `error` 字段
6. **异步任务**：导出、导入、移动等操作是异步的，需要使用 `drive +task_result` 查询结果

---

> **相关技能**：
> - `feishu-doc` — 飞书云文档详细操作
> - `feishu-im` — 飞书即时通讯详细操作
> - `feishu-drive` — 飞书云空间操作
> - `feishu-sheets` — 飞书电子表格详细操作
> - `feishu-base` — 飞书多维表格详细操作
> - `feishu-shared` — 飞书公共配置和认证
