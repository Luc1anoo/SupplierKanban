# 远端用户 MCP 数据库查询服务 — 完整使用手册

---

## 一、概述

本服务提供了通过 opencode 客户端远程查询 `antenna_ms` MySQL 数据库的能力。远端用户只需修改本地配置文件，即可在 opencode 对话中使用自然语言进行数据库查询，无需部署任何服务端代码。

### 服务信息

| 项目 | 说明 |
|------|------|
| 服务地址 | `http://10.33.128.132:8000` |
| 传输协议 | SSE (Server-Sent Events) |
| 数据库名 | `antenna_ms` |
| 可用数据表 | `t_detection_r1`, `t_detection_r2`, `t_detection_r3`, `t_department` |
| 查询权限 | 仅 `SELECT` 查询 |

---

## 二、核心概念说明

### 什么是 "opencode 工作目录"（即手册中的"项目文件夹"）

这是远端用户**本地计算机**上的一个目录。每当您在命令行中运行 `opencode` 命令时，当前所在的目录就是您的 opencode 工作目录。

例如：
```powershell
# 您在本地电脑上进入自己的项目目录
D:\my_projects\data_analysis> opencode
```
此时，`D:\my_projects\data_analysis` 就是您的 opencode 工作目录（即"项目文件夹"）。

**重要理解：**
- **不是** Django 服务端项目目录（服务端在 `10.33.128.132` 上）
- **不是** opencode 软件的安装目录
- **就是** 您当前正在用 opencode 干活的这个本地文件夹
- 每个用户每个工作目录都可以独立配置

### 什么是 `opencode.json` 配置文件

`opencode.json` 是 opencode 客户端的配置文件，放在您的工作目录根目录下。opencode 启动时会自动读取该文件，加载其中配置的 MCP 工具。

**配置生效范围：** 仅对当前所在的 opencode 工作目录生效。如果您在多个项目中使用此服务，需要在每个项目的目录下分别配置，或者将配置文件放在您的用户主目录中（全局生效，具体取决于 opencode 版本是否支持）。

---

## 三、配置步骤（详细版）

### 步骤 1：打开（或创建）`opencode.json` 文件

**操作位置：** 您本地 opencode 工作目录的根目录下

**操作步骤：**
1. 用文件资源管理器或终端进入您的 opencode 工作目录
2. 在该目录下查找是否已存在 `opencode.json` 文件
3. 如果已存在：用文本编辑器（如 VS Code、Notepad++、记事本等）打开该文件
4. 如果不存在：新建一个名为 `opencode.json` 的文件

### 步骤 2：编写配置文件内容

如果文件**不存在**，请创建并粘贴以下内容：
```json
{
  "$schema": "https://opencode.ai/config.json",
  "mcp": {
    "db-query": {
      "type": "sse",
      "url": "http://10.33.128.132:8000/mcp/sse/",
      "enabled": true
    }
  }
}
```

如果文件**已存在**且有其他 MCP 配置（例如本地 `type: "local"` 的 `db-query`），请将对应的 `db-query` 配置块修改为上述 SSE 格式。

**字段逐项详解：**

| 字段 | 值 | 说明 |
|------|-----|------|
| `$schema` | `https://opencode.ai/config.json` | 配置文件格式校验，建议保留 |
| `mcp` | 对象 | MCP 工具配置的顶层节点 |
| `db-query` | 字符串 | 服务名称，可自定义（如 `antenna-db`），不影响功能 |
| `type` | `"sse"` | **关键配置**：表示使用 SSE 协议连接远端 MCP 服务。如果是 `"local"`，opencode 会在本地启动进程，远端用户无法使用 |
| `url` | `http://10.33.128.132:8000/mcp/sse/` | 服务端 SSE 端点地址，**必须包含末尾的 `/`** |
| `enabled` | `true` | 启用该 MCP 服务 |

### 步骤 3：保存文件并检查格式

1. 保存修改后的 `opencode.json` 文件
2. 确保文件格式正确——JSON 对格式要求严格，常见的错误包括：
   - 末尾多了一个逗号（如 `"enabled": true,` 后面没有内容）
   - 引号使用了中文引号（`""`）而非英文引号（`""`）
   - 多了一对括号或花括号

### 步骤 4：重启 opencode 客户端使配置生效

1. 如果 opencode 正在运行，先退出当前会话
2. 重新进入您的工作目录，再次运行 `opencode` 命令启动客户端

### 步骤 5：验证配置是否成功

在 opencode 对话框中，您可以：
1. 查看可用工具列表，确认能看到 `get_system_prompt` 和 `execute_sql` 两个工具
2. 或直接尝试发送一条查询指令：`帮我查询 antenna_ms 数据库的最新一条测试记录`
3. 如果 opencode 正常调用并返回了结果，说明配置成功

---

## 四、服务提供的工具详解

### 工具 1：`get_system_prompt` — 获取数据库查询提示词

**作用：** 从服务端获取 `antenna_ms` 数据库的完整表结构信息，包含字段名、类型、约束条件、表关系等。远端的 AI 模型根据这个提示词来生成正确的 SQL 查询语句。

**参数：**
- `prompt_name`（可选）：指定提示词名称，默认为 `"generate_system_promt"`

**返回内容包含：**
- 数据库角色定义及时间格式规则
- `t_detection_r1` 表的 33 个字段及约束条件
- `t_detection_r2` 表的 6 个字段及约束条件
- `t_detection_r3` 表的 13 个字段及约束条件
- `t_department` 表的 6 个字段及约束条件
- 表之间的关联关系（外键映射）
- 默认查询条件：必须添加 `data_status = 0`

### 工具 2：`execute_sql` — 执行 SQL 查询

**作用：** 在服务端执行 SQL 查询语句，将结果返回给远端用户。

**参数：**
- `sql`（必填）：标准 MySQL SELECT 查询语句

**返回数据格式：**
```json
{
  "sql": "实际执行的 SQL 语句",
  "rowcount": 12,
  "columns": ["id", "product_name", "test_result", ...],
  "data": [
    [1, "APMI反射镜", "合格", ...],
    [2, "其他产品", "不合格", ...]
  ]
}
```

**安全限制：**
- 仅允许 `SELECT` 查询
- `DROP`、`DELETE`、`TRUNCATE`、`UPDATE`、`INSERT` 等操作会被服务端拒绝并返回错误信息

---

## 五、使用流程详解

### 第一次使用（完整流程）

1. **您向 opencode 提出需求**，例如：`查询 2026 年 6 月 APMI反射镜有多少条测试记录`
2. **opencode 自动调用 `get_system_prompt`** — 首次查询时自动从服务端获取完整的数据库表结构信息
3. **远端 AI 生成 SQL** — opencode 的 AI 模型根据表结构信息和您的需求，在本地生成标准 SQL 语句，例如：
   ```sql
   SELECT COUNT(*) FROM t_detection_r1
   WHERE product_name='APMI反射镜'
   AND end_time BETWEEN '2026/06/01 00:00:00' AND '2026/06/30 23:59:59'
   AND data_status=0
   ```
4. **opencode 调用 `execute_sql`** — 将生成的 SQL 发送到服务端执行
5. **服务端返回查询结果** — 包含行数、列名、数据行
6. **结果展示** — opencode 格式化输出，您可以直接查看统计结果

### 查询结果为空时

如果查询返回 0 条数据或 SQL 执行出错：
1. opencode 会自动提示：`查询未返回数据，是否需要调整 SQL？`
2. 您回答"是"
3. opencode 的 AI 模型会根据错误信息重新修正 SQL 语句
4. 修正后的 SQL 会再次发送给服务端执行
5. 这个过程可以重复多次，直到查询到有效数据

### 日常使用（简化流程）

首次获取系统提示词后，后续查询直接使用，无需重复调用 `get_system_prompt`。您只需在对话中用自然语言描述查询需求即可：

```
查询最近一周不合格产品的数量
查看 APMI反射镜的所有测试记录
统计各产品的合格率
```

---

## 六、查询能力与数据表说明

### 可用的数据表

| 表名 | 字段数 | 主要用途 |
|------|--------|----------|
| `t_detection_r1` | 33 | 主检测记录表，包含产品名、时间、结果等核心字段 |
| `t_detection_r2` | 6 | R1 的扩展数据（通过 `r1_id` 关联） |
| `t_detection_r3` | 13 | R2 的扩展数据（通过 `r2_id` 关联） |
| `t_department` | 6 | 部门信息表（通过 `product_name` 关联 R1） |

### 表关联关系
```
t_detection_r2.r1_id → t_detection_r1.id
t_detection_r3.r2_id → t_detection_r2.id
t_department.product_name → t_detection_r1.product_name
```

### 默认查询条件
所有查询自动添加 `data_status = 0` 条件，仅返回有效数据。

### 时间格式要求
- `end_time` 等字段：`YYYY/mm/dd HH:MM:SS`（如 `'2026/06/30 23:59:59'`）
- `update_time` 字段：`YYYY-mm-dd HH:MM:SS`（如 `'2026-06-30 23:59:59'`）

---

## 七、常见问题排查

| 问题现象 | 可能原因 | 解决方法 |
|----------|----------|----------|
| 连接超时，无任何工具可用 | 本机无法访问服务端 IP | 确认能 ping 通 `10.33.128.132`，联系网络管理员 |
| 报错 "301 Moved Permanently" | URL 缺少末尾的 `/` | 将 `"url"` 改为 `"http://10.33.128.132:8000/mcp/sse/"`（带末尾斜杠） |
| 工具列表中没有 `db-query` 相关工具 | 配置未生效 | 完全退出 opencode 后重新启动 |
| 提示 `query_database` 工具找不到 | 仍使用的是旧版 `type: "local"` 配置 | 将 `"type"` 从 `"local"` 改为 `"sse"` |
| `execute_sql` 返回 "危险操作" 错误 | SQL 中包含 `DROP` / `DELETE` / `TRUNCATE` / `UPDATE` / `INSERT` | 仅允许 `SELECT` 查询，请改用查询语句 |
| `execute_sql` 返回 SQL 语法错误 | 生成的 SQL 不符合 MySQL 语法 | 让 opencode AI 自动修正，或手动提供修正指令 |
| 查询超时（等待很久无结果） | 查询涉及大量数据或复杂 JOIN | 缩小查询范围，添加时间等条件限制 |
| SSE 连接自动断开 | 连接空闲超过 30 分钟 | 重新发起查询会自动重建连接，无需手动操作 |

---

## 八、注意事项

1. **仅支持 SELECT 查询** — 服务端严格禁止 `DROP`、`DELETE`、`TRUNCATE`、`UPDATE`、`INSERT` 等写操作。这些请求会被安全校验机制拦截并返回错误。
2. **查询超时时间** — 单次查询超时限制为 120 秒。如果查询涉及大量数据，建议添加精确条件缩小范围。
3. **连接自动管理** — SSE 连接在空闲 30 分钟（1800 秒）后会自动断开。下次查询时 opencode 会自动重新建立连接，无需用户手动干预。
4. **JSON 配置格式要求** — `opencode.json` 必须是合法的 JSON 格式。常见的格式错误（如末尾逗号、中文引号）会导致配置无法读取。

---

## 九、快速上手示例

### 示例 1：统计查询

直接对 opencode 说：
```
查询 2026 年 6 月 APMI反射镜 有多少条测试记录
```

opencode 自动完成的步骤：
1. 调用 `get_system_prompt` 获取表结构
2. AI 生成 SQL：`SELECT COUNT(*) FROM t_detection_r1 WHERE product_name='APMI反射镜' AND end_time BETWEEN '2026/06/01 00:00:00' AND '2026/06/30 23:59:59' AND data_status=0`
3. 调用 `execute_sql` 执行查询
4. 返回结果：`共 XX 条记录`

### 示例 2：数据查询

```
查看最近 7 天所有不合格产品的记录
```

### 示例 3：关联查询

```
查询 APMI反射镜的检测结果，同时显示对应的部门信息
```

---

## 附录：完整配置示例

```json
{
  "$schema": "https://opencode.ai/config.json",
  "mcp": {
    "db-query": {
      "type": "sse",
      "url": "http://10.33.128.132:8000/mcp/sse/",
      "enabled": true
    }
  }
}
```

如果已有其他配置，只需在 `mcp` 对象中添加或替换 `db-query` 配置块即可：
```json
{
  "$schema": "https://opencode.ai/config.json",
  "mcp": {
    "your-other-tool": {
      "type": "local",
      "command": ["...", "..."],
      "enabled": true
    },
    "db-query": {
      "type": "sse",
      "url": "http://10.33.128.132:8000/mcp/sse/",
      "enabled": true
    }
  }
}
```
