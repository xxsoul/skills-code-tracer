---
name: code-impact-tracker
description: Track code changes' impact on business scope through LSP call chain tracing. Use this skill when users ask about code impact analysis, business scope affected by code changes, call chain tracing, "what does this function affect", "which APIs call this code", or need impact reports with git commit references. Supports any language with LSP available.
---

# Code Impact Tracker

追踪代码变更对业务的影响范围，通过 LSP 向上回溯调用链，最终生成业务影响报告。

## 核心流程

```
输入起点 → LSP回溯调用链 → 识别业务入口 → 汇总影响范围 → 生成报告
```

---

## Step 1: 解析输入起点

用户可通过三种方式指定分析起点：

### 1.1 位置格式 (文件:行号)

```
文件路径:行号[:列号]
例如: websrv/public_websrv/userinterface/lands.go:52
```

解析步骤：
1. 确认文件存在
2. 确认行号有效（不超过文件行数）
3. 定位到该位置的符号（函数/方法/变量）

### 1.2 Git 提交编号

```
提交哈希 (完整或简短)
例如: 09ac4b9f8 或 09ac4b9
```

解析步骤：
1. 使用 `git show --stat <commit>` 查看变更文件列表
2. 使用 `git diff <commit>^..<commit>` 查看具体变更内容
3. 提取所有变更的函数/方法作为分析起点
4. 若变更较多，询问用户是否全部分析或选择特定文件

### 1.3 函数名搜索

```
函数名或方法名
例如: ProjectGroupAdd 或 project_group.GroupAdd
```

解析步骤：
1. 使用 Grep 搜索函数定义位置
2. 若找到多个定义，列出位置供用户选择
3. 确认唯一定义位置后作为起点

---

## Step 2: LSP 环境检测

在开始回溯前，**必须**检测 LSP 是否可用：

### 2.1 检测方法

根据文件类型判断需要的 LSP：

| 语言 | LSP 服务器 | 检测方式 |
|------|-----------|---------|
| Go | gopls | 检查 `gopls version` |
| Python | pylsp/pyright | 检查 `pylsp --version` 或配置 |
| Java | jdtls | 检查 jdtls 进程 |
| TypeScript/JS | typescript-language-server | 检查 `tsserver` |
| Rust | rust-analyzer | 检查进程或配置 |
| C/C++ | clangd | 检查 `clangd --version` |

### 2.2 LSP 不可用时的处理

如果 LSP 不可用：
1. **明确告知用户**：当前环境没有配置该语言的 LSP
2. **提供安装指引**：列出常见 LSP 安装方法
3. **询问是否继续**：是否使用备选方案（grep 搜索调用）

备选方案（无 LSP 时）：
- 使用 Grep 搜索函数名调用点
- 精度较低，但可提供基本分析

---

## Step 3: LSP 调用链回溯

### 3.1 回溯算法

使用 LSP 的 `prepareCallHierarchy` → `incomingCalls` 功能：

```
当前层 (起点)
    ↓ prepareCallHierarchy 获取调用层级项
    ↓ incomingCalls 获取所有调用方
第1层调用方
    ↓ 对每个调用方重复 prepareCallHierarchy + incomingCalls
第2层调用方
    ↓ ...
第N层调用方
```

### 3.2 深度控制

- **默认深度**：6 层
- **超出处理**：每满 6 层询问用户：
  ```
  已回溯 6 层，找到 X 个调用方。
  是否继续？
  - 继续回溯 6 层
  - 无限回溯直到终点
  - 停止，当前结果已足够
  ```

### 3.3 回溯数据收集

每层收集以下信息：

```json
{
  "layer": 1,
  "callers": [
    {
      "file": "path/to/file.go",
      "line": 52,
      "character": 10,
      "symbol": "FunctionName",
      "symbol_type": "function|method|variable",
      "container": "ClassName or Package",
      "call_expression": "actualCall()"
    }
  ]
}
```

### 3.4 终止条件

回溯终止于：
- **业务入口点**：被识别为 API handler 或配置驱动的入口
- **无调用方**：LSP 返回空结果
- **外部依赖**：调用来自第三方包（标注为外部依赖）
- **用户停止**：用户选择停止回溯

---

## Step 4: 识别业务入口点

业务入口点识别采用**混合模式**：自动识别 + 用户确认

### 4.1 API 入口识别

自动检测以下特征：

| 特征类型 | 检测规则 | 示例 |
|---------|---------|------|
| HTTP Handler | 函数名包含 Handler、Handle | `ProjectGroupHandler` |
| 路由注册 | 文件名含 router/lands/route | `lands.go` |
| 参数特征 | 参数含 context、Request、Response | `func Handler(c *fiber.Ctx)` |
| 注解特征 | 注释含 @api、@endpoint | `// @api POST /path` |
| 框架特征 | 注册到 router 对象 | `router.Post("/path", handler)` |

识别后**询问用户确认**：
```
检测到以下潜在 API 入口：
1. router.Post("/project_group/add", GroupAdd) - 项目组添加接口
2. router.Get("/resource/view", Overview) - 资源总览接口

请确认这些是否为业务 API 入口？如有遗漏请补充。
```

### 4.2 配置文件业务识别

搜索配置文件中的业务定义：

1. **扫描配置文件**：项目根目录下的 `config*.yaml`、`config*.json`
2. **识别业务定义**：
   - 配置项的 key/value 结构
   - 任务名称、服务名称
   - 定时任务配置
3. **追溯配置使用**：
   - 搜索代码中读取配置的位置
   - 将配置读取代码纳入调用链

### 4.3 业务标签整理

最终为每个入口点标注业务标签：

```json
{
  "entry_type": "api|config|schedule|unknown",
  "business_name": "项目组管理",
  "api_path": "/vesta/v2/public/websrv/project_group/add",
  "method": "POST",
  "config_key": "project_group.enable",
  "description": "添加项目组的 API 接口"
}
```

---

## Step 5: 汇总影响范围

### 5.1 影响树构建

将回溯数据构建为影响树：

```
起点函数
├── 影响 API-1: 项目组添加
│   ├── 调用链: Handler → Service → 起点
│   └── 业务影响: 用户无法添加项目组
├── 影响 API-2: 项目组列表
│   └── ...
├── 影响配置: project_group.yaml
│   └── ...
```

### 5.2 影响分类

将影响分为：

| 影响级别 | 描述 | 示例 |
|---------|------|------|
| 直接影响 | API/入口直接调用该函数 | 接口返回错误 |
| 间接影响 | 通过中间层调用 | 下游功能受限 |
| 配置影响 | 配置驱动相关逻辑 | 功能开关失效 |
| 外部依赖 | 第三方包调用 | 需关注上游变更 |

---

## Step 6: 生成报告

同时生成 Markdown 和 JSON 格式报告。

### 6.1 报告文件名

```
impact_report_{identifier}_{date}.md
impact_report_{identifier}_{date}.json

identifier: 函数名、提交哈希简写、或自定义标识
date: YYYYMMDD 格式
```

### 6.2 Markdown 报告结构

```markdown
# 代码影响分析报告

## 基本信息
- **分析日期**: YYYY-MM-DD HH:MM:SS
- **分析起点**: 文件路径:行号
- **起点符号**: FunctionName
- **Git 提交**: commit_hash (如适用)
- **回溯深度**: N 层

## 影响摘要
- **影响 API 数量**: X 个
- **影响配置项**: X 个
- **影响业务模块**: X 个
- **外部依赖**: X 个

## 业务影响详情

### API 影响列表

| 序号 | API 路径 | 方法 | 业务名称 | 影响描述 |
|-----|---------|------|---------|---------|
| 1 | /path/to/api | POST | 业务名 | 描述 |

### 配置影响列表

| 序号 | 配置文件 | 配置项 | 业务影响 |
|-----|---------|-------|---------|
| 1 | config.yaml | key.subkey | 描述 |

### 业务模块影响

| 序号 | 模块名 | 影响程度 | 影响范围 |
|-----|-------|---------|---------|
| 1 | 模块A | 直接 | 功能描述 |

## 调用链详情

### 调用链 1: API → 起点
```
[Layer 0] FunctionName (起点)
    ↑ called by
[Layer 1] CallerFunc (文件:行号)
    ↑ called by
[Layer 2] HandlerFunc (文件:行号)
    ↑ [API 入口] POST /path/to/api
```

## 建议
1. 变更此函数需要关注以下 API 的测试覆盖
2. 建议通知相关业务负责人
3. 配置项变更需同步更新文档

## 附录
- LSP 版本信息
- 分析耗时
- 数据完整性说明
```

### 6.3 JSON 报告结构

```json
{
  "report_meta": {
    "generated_at": "2026-04-07T12:00:00Z",
    "analyzer_version": "1.0",
    "lsp_server": "gopls 0.12.0",
    "language": "go"
  },
  "analysis_source": {
    "type": "location|commit|function",
    "identifier": "commit_hash or function_name",
    "file": "path/to/file",
    "line": 52,
    "symbol": "FunctionName"
  },
  "trace_config": {
    "max_depth": 6,
    "actual_depth": 12,
    "user_interactions": 1
  },
  "impact_summary": {
    "api_count": 3,
    "config_count": 2,
    "module_count": 5,
    "external_count": 1
  },
  "call_chains": [
    {
      "chain_id": 1,
      "layers": [
        {
          "layer": 0,
          "file": "path",
          "line": 52,
          "symbol": "FunctionName"
        },
        {
          "layer": 1,
          "file": "path",
          "line": 100,
          "symbol": "CallerFunc"
        }
      ],
      "entry_point": {
        "type": "api",
        "path": "/api/path",
        "method": "POST",
        "business_name": "业务名"
      }
    }
  ],
  "business_impacts": [
    {
      "entry_type": "api",
      "business_name": "项目组管理",
      "api_path": "/vesta/v2/public/websrv/project_group/add",
      "method": "POST",
      "impact_level": "direct",
      "impact_description": "添加项目组功能将受影响"
    }
  ],
  "recommendations": [
    "变更此函数需要关注项目组管理模块的测试",
    "建议通知项目负责人"
  ]
}
```

---

## 工作目录结构

分析过程中创建以下目录结构：

```
code-impact-tracker-workspace/
├── analysis_input.json      # 用户输入记录
├── trace_data.json          # 回溯中间数据
├── business_entries.json    # 业务入口识别结果
├── impact_report_XXX_YYYYMMDD.md
└── impact_report_XXX_YYYYMMDD.json
```

---

## 使用示例

### 示例 1: 从函数名开始

```
用户: 分析 GroupAdd 函数的业务影响

助手:
1. 搜索 GroupAdd 定义位置
2. 确认起点: project_group.go:25
3. 检测 LSP (gopls) 可用
4. 开始 6 层回溯...
   Layer 1: 找到 2 个调用方
   Layer 2: 找到 3 个调用方
   ...
5. 识别业务入口: POST /project_group/add
6. 生成报告
```

### 示例 2: 从 Git 提交开始

```
用户: 分析提交 09ac4b9f8 的业务影响

助手:
1. git show 09ac4b9f8 --stat
2. 变更涉及 3 个文件:
   - config.yaml
   - handler.go
   - service.go
3. 提取变更函数: ConfigLoader, HandleRequest, ProcessData
4. 对每个函数进行回溯分析
5. 汇总所有影响
6. 生成报告
```

### 示例 3: 从位置开始

```
用户: 分析 websrv/public_websrv/userinterface/lands.go:52 的业务影响

助手:
1. 定位到 router.Post("/project_group/add", ...)
2. 检测 LSP 可用
3. 回溯调用链
4. 识别这是 API 路由注册点
5. 确认业务: 项目组添加接口
6. 生成报告
```

---

## 注意事项

1. **LSP 是核心依赖**：尽量确保 LSP 可用，否则分析精度下降
2. **交互确认重要**：业务入口识别需要用户确认，避免误判
3. **深度控制灵活**：让用户决定是否深入回溯
4. **报告即时保存**：每完成一部分就保存中间数据，防止中断丢失
5. **跨语言通用**：设计适配多语言，但具体 LSP 调用可能有差异