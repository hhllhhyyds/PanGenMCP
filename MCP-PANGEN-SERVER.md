# MCP Pangen Server 设计文档

## 1. 概述

MCP Pangen Server 是一个 MCP（Model Context Protocol）服务器，用于将 Pangen（光刻 OPC 软件）与 AI Agent（如 Claude Code、Lattice）连接起来，支持**手动调参**和**自动化迭代优化**两种模式。

### 1.1 整体架构

```
┌─────────────────────────────────────────────────────────────────────┐
│                        Claude Code (AI Agent)                       │
│                                                                     │
│  · 调用 MCP Server 工具（job_submit, analyze_result 等）              │
│  · 调用 RAG 知识库（search_pangen_docs）                              │
│  · 结合两者结果做分析和决策                                             │
└─────────────────────────────────────────────────────────────────────┘
                │                                    │
                │  MCP 工具调用                       │ RAG 检索
                ↓                                    ↓
┌─────────────────────────────┐          ┌─────────────────────────────┐
│     MCP Pangen Server       │          │      RAG Server             │
│                             │          │    (向量知识库)               │
│  ┌───────────────────────┐  │          └─────────────────────────────┘
│  │    迭代优化层           │  │
│  │  Task Mgr             │  │
│  │Parameter Tuner(轻量规则)│  │
│  │  Result Analyzer      │  │
│  └───────────────────────┘  │
│              ↓              │
│  ┌───────────────────────┐  │
│  │    基础工具层           │  │
│  │  Job Mgr / Config Mgr │  │
│  └───────────────────────┘  │
│              ↓              │
│  ┌───────────────────────┐  │
│  │  Pangen Interface     │  │
│  │  (SSH 执行器)          │  │
│  └───────────────────────┘  │
└─────────────────────────────┘
                │
                │ SSH
                ↓
┌─────────────────────────────────────────────────────────────────────┐
│                          GPU Cluster                                │
│  · Pangen 可执行文件 · Log 文件 · 结果文件 (GB级)                       │
└─────────────────────────────────────────────────────────────────────┘
```

### 1.2 组件职责

| 组件                      | 职责                                                           |
| ------------------------- | -------------------------------------------------------------- |
| **Claude Code / Lattice** | AI Agent，通过 MCP 协议调用工具，结合 RAG 知识做决策           |
| **RAG Server**            | 向量知识库，Claude Code 直接连接，MCP Server 不感知            |
| **GPU Cluster**           | 运行 Pangen 的服务器集群，MCP Server 通过 SSH 访问             |
| **MCP Pangen Server**     | MCP 协议处理、Job 管理、配置管理、结果分析、调参优化、确认网关 |

MCP Pangen Server 内部各模块职责如下：

| #    | 模块                   | 职责                                                                   |
| ---- | ---------------------- | ---------------------------------------------------------------------- |
| 2.1  | **MCP Protocol Layer** | JSON-RPC 通信，处理 `tools/list`、`tools/call` 请求，暴露 Resources    |
| 2.2  | **Job Manager**        | Job 生命周期管理：submit/status/list/cancel/result/logs                |
| 2.3  | **Config Manager**     | 配置文件读写、验证、管理多配置模板                                     |
| 2.4  | **Confirmation Gate**  | 高风险操作前暂停，等待用户确认（脚本执行/敏感操作/长时 Job）           |
| 2.5  | **Pangen Interface**   | SSH 命令执行、配置推送、流式 log 读取、结果文件元信息读取              |
| 2.6  | **Code Execution**     | 接收并执行 Python 脚本，封装工具无法覆盖的复杂/一次性操作              |
| 2.7  | **Result Analyzer**    | 解析二进制结果，提取数值指标（CD 误差、良率等），生成结构化报告        |
| 2.8  | **Parameter Tuner**    | 轻量规则引擎，提供预定义调参策略，记录调参历史；复杂优化由外部服务完成 |
| 2.9  | **Task Manager**       | 迭代优化任务管理：创建/追踪/暂停/恢复/终止任务，维护 job 链和调参历史  |
| 2.10 | **Local Storage**      | 状态文件（Job/Task）、配置模板、调参策略、MCP 服务器自身日志的持久化   |

---

## 2. 模块详解

### 2.1 MCP Protocol Layer

MCP 协议层，处理 JSON-RPC 通信：

- 接收来自 Claude Code 的 `tools/list`、`tools/call` 请求
- 返回工具定义和执行结果
- 暴露 Resources 供 Claude Code 查询

### 2.2 Job Manager（Job 管理模块）

核心模块，负责 Pangen 任务的完整生命周期管理。

**功能**：
- 生成 job_id（UUID v4）
- 通过 SSH 在 GPU 服务器上启动/取消 Pangen 进程
- 轮询 job 状态（从状态文件读取）
- 管理 job 工作目录

**核心接口**：

| 接口                  | 说明                                   |
| --------------------- | -------------------------------------- |
| `submit(config_name)` | 创建 job，SSH 启动 Pangen，返回 job_id |
| `status(job_id)`      | 查询 job 当前状态                      |
| `list()`              | 列出所有 job                           |
| `cancel(job_id)`      | 发送 SIGTERM 取消 job                  |
| `result(job_id)`      | 获取结果文件元信息（路径/大小/格式）   |
| `logs(job_id)`        | 读取 job 的 log 文件内容               |

**状态文件**（保存在本地 `~/.pangen-mcp/jobs/`）：
```json
{
  "id": "uuid-v4",
  "gpu_host": "gpu-01",
  "pid": 12345,
  "work_dir": "/data/job-xxx",
  "config_file": "/path/to/config.json",
  "status": "pending|running|completed|failed|cancelled",
  "progress": "45%",
  "start_time": "...",
  "end_time": null,
  "exit_code": null
}
```

**状态文件更新机制**（两层轮询的内层）：

Job 运行期间，MCP Server 后台线程定期（~30s）通过 SSH 检查 GPU 服务器上的进程状态，并同步更新本地状态文件。更新流程如下：

1. 通过 SSH 读取 GPU 上进程/Pangen 的实时状态（如 `progress`、`exit_code`）
2. 将最新状态写入本地 `~/.pangen-mcp/jobs/{job_id}.json`
3. 更新采用原子写入（写临时文件 → 替换原文件），避免并发写入导致文件损坏

Job Manager 同时是状态文件的**写入方**，Claude Code 通过 `job_status` 接口读取文件内容作为返回结果。

### 2.3 Config Manager（配置管理模块）

管理 Pangen 的配置文件。配置文件存放在 MCP 服务器本地，包含 JSON 格式的仿真参数。

**功能**：
- 读取/写入配置文件的指定字段
- 配置验证（参数类型、范围检查）
- 管理多个配置模板

**使用场景**：
```
用户: "把迭代次数改成 100"
Claude 调用 config_set(key="simulation.iterations", value="100")

用户: "当前配置里的迭代次数是多少"
Claude 调用 config_get(key="simulation.iterations")
```

**注意**：这里管理的"配置"是 Pangen 运行参数的配置文件，不是 MCP 服务器本身的配置。

### 2.4 Confirmation Gate（确认网关）

在执行高风险操作前，要求用户手动确认。

**需要确认的场景**：

| 场景     | 说明                                               |
| -------- | -------------------------------------------------- |
| 执行脚本 | `execute_pangen_script` 执行前，用户需审阅脚本内容 |
| 敏感操作 | 删除数据、修改集群配置等高风险操作                 |
| 长时 Job | 启动预计运行超过 30 分钟的 job                     |

**流程**：
```
1. Claude 调用需要确认的工具
2. MCP 服务器返回确认请求，暂停执行
3. Claude Code 向用户展示确认对话框
4. 用户批准 → MCP 服务器继续执行
5. 用户拒绝 → 返回错误
```

### 2.5 Pangen Interface（Pangen 接口层）

封装与 Pangen 程序交互的底层细节。

**功能**：
- SSH 命令执行器（每次命令新开一个 SSH 连接）
- 配置文件推送器（将本地配置模板同步到 GPU 工作目录）
- Log 文件读取器（流式读取，支持 offset）
- 结果文件元信息读取器（路径/大小/格式）

**配置与执行的关系**：

Job Manager 通过 `job_submit` 触发 Pangen Interface，启动流程如下：

```
1. Config Manager 提供配置（本地路径 ~/.pangen-mcp/configs/{name}.json）
         ↓
2. Pangen Interface 通过 SSH 将配置推送到 GPU 工作目录（如 /data/job-xxx/config.json）
         ↓
3. SSH 执行 Pangen 命令，指定配置文件路径（如 pangen --config /data/job-xxx/config.json）
         ↓
4. nohup 启动 Pangen，记录 PID
5. 关闭 SSH 连接，后台轮询状态文件
```

Job 状态文件中的 `config_file` 字段记录的是 GPU 集群上该配置文件的位置（而非本地路径），供后续 log 分析和结果关联使用。

**SSH 执行流程**：
```
1. 建立 SSH 连接
2. 推送配置文件到 GPU 集群服务器
3. 创建工作目录
4. nohup 启动 Pangen（携带配置文件路径），记录 PID
5. 关闭 SSH 连接
6. 后台轮询状态文件
```

### 2.6 Code Executor（代码执行模块）

`execute_pangen_script` 是 MCP 服务器中最灵活的工具，用于处理**封装工具无法覆盖的复杂或一次性操作**。

**设计理念：菜单 + 开发者模式的混合架构**

| 层次                                       | 工具类型               | 适用场景                         |
| ------------------------------------------ | ---------------------- | -------------------------------- |
| **封装工具层**（Job/Config/Task 系列接口） | 预定义工具，白名单模式 | 常规调参、状态查询等高频明确任务 |
| **代码执行层**（`execute_pangen_script`）  | 动态脚本，开发者模式   | 封装工具未覆盖的复杂/一次性操作  |

**Claude Code 的决策逻辑**：
1. 分析用户意图
2. 扫描工具列表 → 封装工具能覆盖则直接调用
3. 封装工具无法满足 → 调用 RAG 知识库查询 Pangen API 和历史脚本案例
4. 编写脚本 → 调用 `execute_pangen_script` → 获取结果 → 验证

**安全机制**：

| 层级       | 机制              | 说明                                                         |
| ---------- | ----------------- | ------------------------------------------------------------ |
| **第一层** | Confirmation Gate | 脚本执行前暂停，等待用户审阅并批准代码内容（**主要安全线**） |
| **第二层** | 资源限制          | 最大执行时间 / CPU 时间，防止死循环                          |
| **第三层** | 结果校验          | 执行完成后读取输出和 log，验证是否成功                       |

> **注意**：本服务为内部使用，不设进程级沙箱。Confirmation Gate（人工审阅代码）是主要安全屏障。

**使用示例**：
```
用户: "把配置里所有 resolution > 8 的网格，其 weight 乘以 0.9"

Claude 先查询 RAG 中类似脚本案例 → 编写 Python 脚本 →
调用 execute_pangen_script(script="...") → 脚本在 GPU 服务器上执行 →
返回 stdout/stderr 和执行结果
```

**与 Job Manager 的区别**：Job Manager 管理的是完整的 Pangen 仿真任务（有生命周期、状态文件），Code Executor 处理的是任意的单次 Python 脚本调用，两者互补而非包含关系。

### 2.7 Result Analyzer（结果分析模块）

Job 完成后，分析 Pangen 的输出结果，提取数值指标。

**执行位置：GPU 服务器端**。结果文件为 GB 级二进制，不传输到本地。MCP Server 通过 SSH 在 GPU 服务器上执行预部署的分析脚本，仅将提取出的数值指标（KB 级 JSON）传回本地。

**功能**：
- 通过 SSH 在 GPU 侧执行分析脚本，解析二进制结果文件
- 提取数值指标（如 CD 误差、良率等）
- 读取 log 文件，提取关键事件和警告
- 将提取的指标结构化后返回给 Claude Code

**输出格式**：
```json
{
  "job_id": "xxx",
  "metrics": {
    "cd_error_mean": 0.23,
    "cd_error_max": 1.87,
    "yield_estimate": 94.5
  },
  "warnings": ["部分区域收敛较慢", "建议增加迭代次数"]
}
```

**注意**：模糊判断（如"布局复杂度较高"）由 Claude Code 结合 RAG 知识库完成，Result Analyzer 只负责精确的数值指标提取。

### 2.8 Parameter Tuner（参数调整模块）

轻量级规则引擎，提供预定义的调参策略。**不包含贝叶斯优化等复杂算法**——如需高级优化，由外部独立 MCP Server 或 Skill 提供。

**定位**：
- 本模块仅负责：加载预定义策略、根据规则生成配置建议、记录调参历史
- 最终决策权在 Claude Code（它结合 Tuner 建议 + RAG 经验 + 外部优化器综合判断）

**功能**：
- 加载预定义的调参策略（如"网格细化策略"、"权重调整策略"）
- 根据历史 job 结果和预设规则，生成配置调整建议
- 生成新的配置参数（不直接写入，返回给 Claude Code 决策）
- 记录调参历史

**经验固化**：每次 Task 完成后，Claude Code 将成功/失败的调参路径（如 CD_error 1.2→0.7 的参数调整过程）整理成案例，写入 RAG 或封装为 Skill，固化到知识体系中。下次遇到相似场景时，RAG 可直接检索历史经验，缩短调参周期。

**调参策略示例**：

| 策略名                | 触发条件   | 调整方式       |
| --------------------- | ---------- | -------------- |
| `refine_grid`         | 收敛慢     | 网格间距 × 0.8 |
| `increase_iterations` | 迭代未收敛 | 迭代次数 × 1.5 |
| `adjust_weight`       | 局部误差大 | 调整权重参数   |
| `restart`             | 多次失败   | 重置到初始配置 |

**工作流程**（由 Claude Code 驱动调用）：
```
1. Claude Code 调用 tune_params(job_id, strategy?) 
2. Parameter Tuner 加载策略 + 历史结果
3. 基于规则生成配置建议（不直接写入，返回给 Claude Code）
4. Claude Code 综合 Tuner 建议 + RAG + 外部优化器结果，决定是否采纳
5. Claude Code 调用 config_patch() 应用最终配置
6. 调参记录写入 Task 历史
```

### 2.9 Task Manager（任务管理模块）

管理迭代优化任务的**状态追踪**，包含多个相关的 Job。Task Manager 不驱动迭代循环，循环由 Claude Code 外部驱动。

**功能**：
- 创建调参任务（Task），记录目标和迭代上限
- 追踪任务进度（已完成轮次、当前状态、最佳结果）
- 管理任务内的 job 链和调参历史
- 支持暂停/恢复/终止任务标记

**Task 状态文件**（保存在本地 `~/.pangen-mcp/tasks/`）：
```json
{
  "id": "task-uuid",
  "goal": "优化参数达到 CD_error < 0.5",
  "max_iterations": 10,
  "current_iteration": 3,
  "status": "running|completed|failed|paused",
  "jobs": ["job-1", "job-2", "job-3"],
  "best_result": {
    "job_id": "job-2",
    "cd_error": 0.67
  },
  "start_time": "...",
  "history": [
    {"iteration": 1, "config": {...}, "result": {...}},
    {"iteration": 2, "config": {...}, "result": {...}}
  ]
}
```

### 2.10 Local Storage（本地存储）

MCP 服务器本地的文件存储。

| 路径                                 | 内容               |
| ------------------------------------ | ------------------ |
| `~/.pangen-mcp/jobs/{job_id}.json`   | Job 状态文件       |
| `~/.pangen-mcp/tasks/{task_id}.json` | Task 状态文件      |
| `~/.pangen-mcp/configs/`             | Pangen 配置模板    |
| `~/.pangen-mcp/strategies/`          | 调参策略定义       |
| `~/.pangen-mcp/logs/`                | MCP 服务器自身日志 |

---

## 3. MCP 服务器内部架构图

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                            MCP Pangen Server                                │
├─────────────────────────────────────────────────────────────────────────────┤
│  MCP Protocol Layer (JSON-RPC, tools/list, tools/call)                      │
├─────────────────────────────────────────────────────────────────────────────┤
│                              迭代优化层                                       │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │                    Task Manager（状态追踪）                          │    │
│  │   · 管理迭代任务生命周期（create/progress/pause/resume/stop）           │    │
│  │   · 维护 job 链和调参历史                                              │    │
│  │   · 注意：不驱动迭代循环，循环由 Claude Code 外部驱动                    ｜   │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                    ↓                                        │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │                 Parameter Tuner（轻量规则引擎）                      │    │
│  │   · 加载预定义调参策略，基于规则生成配置建议                            │    │
│  │   · 不含复杂优化算法（如需贝叶斯优化，由外部服务提供）                    │    │
│  │   · 返回建议给 Claude Code，由其综合决策                               │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                    ↓                                        │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │                      Result Analyzer                                │    │
│  │   · 通过 SSH 在 GPU 侧执行分析脚本，提取数值指标                        │   │
│  │   · 生成结构化分析报告（仅传回 KB 级 JSON）                             │    │
│  │   · 注意：不访问 RAG，模糊判断由 Claude Code 结合 RAG 完成                │   │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                    ↑                                        │
├─────────────────────────────────────────────────────────────────────────────┤
│                              基础工具层                                       │
│  ┌─────────────┬─────────────┬─────────────┬──────────────────────────┐     │
│  │  Job Manager│ Config Mgr  │Confirmation │      Pangen Interface    │     │
│  │             │             │   Gate      │                          │     │
│  │ · submit()  │ · get()     │             │ · ssh_exec()             │     │
│  │ · status()  │ · set()     │ · pending() │ · read_log()             │     │
│  │ · cancel()  │ · validate()│ · approve() │                          │     │
│  │ · result()  │             │ · reject()  │                          │     │
│  │ · logs()    │             │             │                          │     │
│  └─────────────┴─────────────┴─────────────┴──────────────────────────┘     │
├─────────────────────────────────────────────────────────────────────────────┤
│                           Local Storage                                     │
│   jobs/{job_id}.json  │  tasks/{task_id}.json  │  configs/  │ strategies/   │
└─────────────────────────────────────────────────────────────────────────────┘
                                        │
                                        │ SSH
                                        ↓
                               ┌──────────────────────┐
                               │    GPU Cluster       │
                               │  · Pangen 可执行文件   │
                               │  · Log 文件 (文本)    │
                               │  · 结果文件 (GB级)     │
                               └──────────────────────┘
```

### 3.1 迭代优化数据流（Claude Code 驱动）

迭代循环由 Claude Code 驱动。MCP Server 的 Task Manager 仅负责状态追踪和历史记录，不做流程编排。

```
用户: "优化参数，目标 CD_error < 0.5，最多 10 次"

Claude Code:
  1. 调用 task_create() 创建任务
  2. 调用 job_submit() 提交 job
  3. 轮询 job_status() 直到完成
  4. 调用 analyze_result() 获取分析报告（数值指标，GPU 侧执行）
  5. 调用 RAG 知识库查询类似案例（可利用历史固化经验快速定位方向）
  6. 结合两者决定下一步：
     - 如果未达标且未达上限：调用 tune_params() 获取调参建议
     - 如果达标或已达上限：停止迭代
  7. 调用 config_patch() 应用新配置
  8. 重复 2-7 直到结束
  9. 调用 task_result() 获取最终报告
 10. 将本次调参路径（成功或失败）写入 RAG / 封装为 Skill，固化经验
```

```
用户: "优化参数，目标 CD_error < 0.5，最多 10 次"

    ┌─────────────────────────────────────────────────────────────┐
    │              Claude Code 驱动循环: iteration = 1             │
    │                                                             │
    │  1. Claude → job_submit() → Job Manager 启动 Pangen         │
    │                           ↓                                 │
    │  2. Pangen 运行 → Job 结束                                   │
    │                           ↓                                 │
    │  3. Claude → analyze_result() → GPU 侧执行分析脚本           │
    │                           ↓                                 │
    │  4. Claude 判断: CD_error = 0.8 > 0.5, 未达标                │
    │  5. Claude → tune_params() → 获取调参建议                    │
    │  6. Claude → config_patch() → 应用新配置                     │
    │                           ↓                                 │
    │  7. iteration = 2，回到步骤 1                                │
    │  ... (循环直到 iteration = 10 或达标)                         │
    │                                                             │
    └─────────────────────────────────────────────────────────────┘
```

---

## 4. 工具列表

### 4.1 Task 工具（Task Management）

| 工具名          | 参数                                                           | 返回                | 说明                   |
| --------------- | -------------------------------------------------------------- | ------------------- | ---------------------- |
| `task_create`   | `goal: string, max_iterations: number, initial_config: string` | `{task_id: string}` | 创建调参任务           |
| `task_progress` | `task_id: string`                                              | `TaskProgress`      | 获取任务进度           |
| `task_pause`    | `task_id: string`                                              | `boolean`           | 暂停任务               |
| `task_resume`   | `task_id: string`                                              | `boolean`           | 恢复任务               |
| `task_stop`     | `task_id: string`                                              | `boolean`           | 停止任务               |
| `task_list`     | 无                                                             | `TaskInfo[]`        | 列出所有任务           |
| `task_result`   | `task_id: string`                                              | `TaskResult`        | 获取任务最终结果和报告 |

### 4.2 Job 工具（Job Management）

| 工具名       | 参数                                    | 返回               | 说明           |
| ------------ | --------------------------------------- | ------------------ | -------------- |
| `job_submit` | `config_name: string, timeout?: number` | `{job_id: string}` | 提交 job       |
| `job_status` | `job_id: string`                        | `JobStatus`        | 查询状态       |
| `job_list`   | 无                                      | `JobInfo[]`        | 列出所有 job   |
| `job_cancel` | `job_id: string`                        | `boolean`          | 取消 job       |
| `job_result` | `job_id: string`                        | `ResultMetadata`   | 获取结果元信息 |
| `job_logs`   | `job_id: string, offset?: number`       | `string`           | 读取 log       |

### 4.3 Config 工具（Configuration Management）

| 工具名            | 参数                                 | 返回               | 说明                       |
| ----------------- | ------------------------------------ | ------------------ | -------------------------- |
| `config_get`      | `key: string`                        | `string`           | 读取配置字段               |
| `config_set`      | `key: string, value: string`         | `boolean`          | 写配置字段                 |
| `config_patch`    | `config_name: string, patch: object` | `boolean`          | 批量修改多个字段（原子性） |
| `config_diff`     | `config_a: string, config_b: string` | `DiffResult`       | 对比两个配置的差异         |
| `config_list`     | 无                                   | `ConfigInfo[]`     | 列出所有配置模板           |
| `config_validate` | `config_name: string`                | `ValidationResult` | 验证配置合法性             |

**说明**：每次 `config_set` / `config_patch` 修改配置时，自动备份上一版本到 `~/.pangen-mcp/configs/{name}.{timestamp}.bak`，支持回溯。

### 4.4 Result Analyzer 工具（Result Analysis）

| 工具名            | 参数                                 | 返回               | 说明                    |
| ----------------- | ------------------------------------ | ------------------ | ----------------------- |
| `analyze_result`  | `job_id: string`                     | `AnalysisReport`   | 分析 job 结果，生成报告 |
| `get_metrics`     | `job_id: string`                     | `Metrics`          | 获取数值指标            |
| `compare_results` | `job_id_1: string, job_id_2: string` | `ComparisonResult` | 对比两个 job 的结果     |

### 4.5 Parameter Tuner 工具（Parameter Tuning — 轻量规则引擎）

| 工具名            | 参数                                            | 返回               | 说明                                                  |
| ----------------- | ----------------------------------------------- | ------------------ | ----------------------------------------------------- |
| `tune_params`     | `job_id: string, strategy?: string`             | `TuningSuggestion` | 基于预定义规则生成调参建议（供 Claude Code 决策参考） |
| `list_strategies` | 无                                              | `StrategyInfo[]`   | 列出可用的预定义调参策略                              |
| `apply_tuning`    | `task_id: string, suggestion: TuningSuggestion` | `boolean`          | 应用调参建议到配置                                    |

### 4.6 代码执行工具（Code Execution）

| 工具名                  | 参数             | 返回           | 说明                    |
| ----------------------- | ---------------- | -------------- | ----------------------- |
| `execute_pangen_script` | `script: string` | `ScriptResult` | 执行 Pangen Python 脚本 |

### 4.7 确认工具（Confirmation）

| 工具名                 | 参数                      | 返回      | 说明                    |
| ---------------------- | ------------------------- | --------- | ----------------------- |
| `confirmation_approve` | `confirmation_id: string` | `any`     | 批准 pending 操作并执行 |
| `confirmation_reject`  | `confirmation_id: string` | `boolean` | 拒绝 pending 操作       |

### 4.8 辅助工具（Utilities）

| 工具名         | 参数 | 返回           | 说明                                      |
| -------------- | ---- | -------------- | ----------------------------------------- |
| `health_check` | 无   | `HealthStatus` | 检查 SSH 连通性、GPU 节点状态、磁盘空间等 |

### 4.9 资源（Resources）

| URI                              | 说明             |
| -------------------------------- | ---------------- |
| `pangen://jobs/{job_id}`         | Job 状态资源     |
| `pangen://jobs`                  | 所有 Job 列表    |
| `pangen://tasks/{task_id}`       | Task 状态资源    |
| `pangen://tasks`                 | 所有 Task 列表   |
| `pangen://configs/{config_name}` | 配置模板内容     |
| `pangen://configs`               | 所有配置模板列表 |
| `pangen://strategies`            | 所有调参策略列表 |

---

## 5. Job 管理机制

### 5.1 job_id 生成策略

MCP 服务器本地生成 UUID v4 作为 job_id，通过 SSH 执行时记录映射信息。

### 5.2 Job 状态文件

路径：`~/.pangen-mcp/jobs/{job_id}.json`

```json
{
  "id": "uuid-v4",
  "gpu_host": "gpu-01",
  "pid": 12345,
  "work_dir": "/data/job-xxx",
  "config_file": "/path/to/config.json",
  "status": "running|completed|failed|cancelled",
  "start_time": "2026-05-01T10:00:00Z",
  "end_time": null,
  "exit_code": null,
  "progress": "45%"
}
```

### 5.3 SSH 执行流程

1. 用户调用 `job_submit`
2. MCP server 生成 `uuid-v4` 作为 job_id
3. 创建 `jobs/{job_id}.json` 状态文件
4. SSH 到 GPU 服务器，创建工作目录
5. nohup 执行 Pangen，记录 PID
6. 写入 PID 到状态文件
7. 返回 `job_id` 给 Claude Code

### 5.4 取消机制

通过 `ssh_session` 或 `gpu_host + pid` 发送 SIGTERM。

---

## 6. 通知机制

Claude Code **不支持** MCP 服务器主动推送通知。采用**两层轮询**模式：

### 6.1 两层轮询架构

| 层级     | 执行者              | 机制                                         | 频率     |
| -------- | ------------------- | -------------------------------------------- | -------- |
| **内层** | MCP Server 后台线程 | 定期 SSH 检查 GPU 进程状态，更新本地状态文件 | 每 30s   |
| **外层** | Claude Code         | 按需调用 `job_status` 读取本地状态文件       | 秒级响应 |

MCP Server 启动 Job 后，内部后台线程负责：
1. 定期（~30s）SSH 到 GPU 检查进程状态（`ps -p {pid}`）
2. 读取 GPU 上 Pangen 的进度信息
3. 原子写入更新本地 `~/.pangen-mcp/jobs/{job_id}.json`

Claude Code 调用 `job_status` 时，MCP Server 仅读取本地状态文件返回，无需实时 SSH。

### 6.2 Job 状态查询

Claude Code 通过 `job_status` 或 `resources/read pangen://jobs/{id}` 获取状态。

### 6.3 典型工作流

```
用户: "启动 pangen job，用配置 A"

Claude 调用 job_submit → 返回 job_id="xxx"
Claude 调用 job_status(job_id="xxx") → status="running"
Claude 调用 job_logs(job_id="xxx") → 读取最新 log
Claude 继续轮询 job_status 直到 status="completed"
Claude 调用 job_result(job_id="xxx") → 返回结果路径/大小/格式
```

---

## 7. 确认机制

三种场景需要用户确认：

| 场景       | 说明                                                 |
| ---------- | ---------------------------------------------------- |
| 代码执行前 | Claude 执行 `execute_pangen_script` 前需用户批准代码 |
| 敏感操作   | 删除数据、修改集群配置等高风险操作                   |
| 长时间 Job | 启动 1 小时以上的 Job 前确认                         |

### 7.1 实现方式（两步调用）

确认机制采用**两步调用**模式，不在单次请求中阻塞等待：

**步骤 1：发起操作，返回 pending**

Claude Code 调用需要确认的工具时，MCP Server 创建一个 pending 确认请求，立即返回确认 ID 和描述信息：

```json
// MCP 返回（第一步）
{
  "content": [{
    "type": "text",
    "text": "需要确认。confirmation_id: cf-xxx\n\n此操作将启动一个 2 小时的 job。\n操作: job_submit\n配置: config_A\nGPU: gpu-01"
  }]
}
```

**步骤 2：用户决策后，调用 approve/reject**

Claude Code 将确认信息展示给用户，用户批准后调用：
- `confirmation_approve(confirmation_id)` → MCP Server 执行操作，返回结果
- `confirmation_reject(confirmation_id)` → MCP Server 取消操作，返回已拒绝

---

## 8. 目录结构

```
pangen-mcp-server/
├── src/
│   ├── main.rs              # 入口
│   ├── mcp/
│   │   ├── mod.rs           # MCP 协议处理
│   │   ├── tools.rs         # 工具定义
│   │   └── resources.rs     # 资源定义
│   ├── task/
│   │   ├── mod.rs           # Task 管理
│   │   ├── manager.rs       # Task 生命周期
│   │   └── state.rs         # Task 状态持久化
│   ├── job/
│   │   ├── mod.rs           # Job 管理
│   │   ├── manager.rs       # Job 生命周期
│   │   └── state.rs         # Job 状态持久化
│   ├── config/
│   │   ├── mod.rs           # 配置管理
│   │   └── validator.rs     # 配置验证
│   ├── confirmation/
│   │   ├── mod.rs           # 确认网关
│   │   └── gate.rs          # 确认逻辑（pending/approve/reject）
│   ├── analyzer/
│   │   ├── mod.rs           # 结果分析
│   │   └── metrics.rs       # 数值指标提取
│   ├── tuner/
│   │   ├── mod.rs           # 参数调整
│   │   ├── strategy.rs      # 调参策略
│   │   └── engine.rs        # 调参引擎
│   ├── code/
│   │   ├── mod.rs           # 代码执行
│   │   └── executor.rs      # 脚本执行器（资源限制）
│   ├── ssh/
│   │   ├── mod.rs           # SSH 接口
│   │   └── executor.rs      # 命令执行
│   ├── pangen/
│   │   ├── mod.rs           # Pangen 接口
│   │   └── parser.rs        # Log/结果解析
├── jobs/                    # Job 状态文件（对应 ~/.pangen-mcp/jobs/）
├── tasks/                   # Task 状态文件（对应 ~/.pangen-mcp/tasks/）
├── configs/                 # 配置模板（对应 ~/.pangen-mcp/configs/）
├── strategies/              # 调参策略定义（对应 ~/.pangen-mcp/strategies/）
├── logs/                    # MCP 服务器自身日志
└── Cargo.toml
```

---

## 9. 技术选型

| 组件     | 选择                   | 理由                            |
| -------- | ---------------------- | ------------------------------- |
| 语言     | Rust                   | 高性能、易维护、与 Lattice 一致 |
| MCP SDK  | `mcp-sdk` / `rust-mcp` | 官方或社区实现                  |
| SSH      | `ssh2` crate           | 纯 Rust 实现                    |
| 状态存储 | JSON 文件              | 简单够用、便于调试              |
| 序列化   | `serde` + `serde_json` | Rust 标准                       |

---

## 10. RAG 知识库

RAG 作为独立服务，Claude Code 直接访问，MCP Server 不感知。

| 组件       | 说明                                    |
| ---------- | --------------------------------------- |
| RAG Server | 独立部署，向量数据库（ChromaDB/Milvus） |
| RAG Client | Claude Code 内置，MCP Server 不涉及     |
| 知识库内容 | Pangen 官方文档、API 手册、历史调参经验 |

**Claude Code 使用 RAG 的场景**：
- 根据 `analyze_result()` 返回的数值指标，结合 RAG 进行模糊判断
- 查询类似问题的历史调参经验
- 获取调参策略建议

### 10.1 经验固化

Claude Code 在每次调参任务中积累的经验通过以下两种方式固化：

| 方式             | 说明                                                                 | 适用场景                      |
| ---------------- | -------------------------------------------------------------------- | ----------------------------- |
| **RAG 案例写入** | 将成功/失败的调参路径（配置变更 → 结果指标）整理为检索文档，写入 RAG | 跨 session 复用、历史经验检索 |
| **封装为 Skill** | 将重复有效的调参模式封装为 Skill（如"收敛慢 → 网格细化"策略）        | 快速复用、同类问题批量处理    |

固化后的经验可被后续 Task 直接复用，减少重复试错，缩短调参周期。

---

## 11. 待确认事项

### Pangen 相关
1. Pangen CLI 命令具体格式和参数
2. 配置文件的字段定义和验证规则
3. Log 文件的格式和解析方式
4. 结果文件的具体格式（OPC 相关二进制格式）
5. 结果分析中数值指标的具体定义（如 CD 误差的计算方式）
6. GPU 侧结果分析脚本的预部署方式和依赖环境

### 调参相关
7. 调参策略的具体定义格式（JSON/YAML/TOML）
8. 初始调参策略由谁提供（用户提供 / MCP 内置 / 从 RAG 学习）
9. 策略不奏效时的回退机制

### 系统集成
10. SSH 连接的具体认证方式（密钥/密码）

### 已决定的事项
- ✅ 迭代循环由 Claude Code 驱动，Task Manager 仅做状态追踪
- ✅ 轮询采用两层架构（MCP Server 后台线程 + Claude Code 按需读取）
- ✅ 第一版每次新开 SSH 连接，轮询频率不高
- ✅ Result Analyzer 在 GPU 服务器端执行
- ✅ Confirmation Gate 采用两步调用（pending → approve/reject）
- ✅ 多 GPU 调度由 Pangen 自身负责（启动参数文件指定）
- ✅ 不设沙箱，Confirmation Gate 作为主要安全线（内部使用）
- ✅ 贝叶斯优化器不放在 MCP Server 内，如需要则作为独立 MCP Server 或 Skill

---

## 12. 典型工作流示例

### 手动调参工作流（用户单次 job）

```
用户: "用配置 A 运行一次 pangen"

Claude 调用 job_submit(config_name="A")
→ 返回 job_id="xxx"

Claude 轮询 job_status 直到 completed

Claude 调用 job_logs(job_id="xxx")
→ 读取 log，分析结果

Claude 调用 job_result(job_id="xxx")
→ 返回结果路径/大小/格式
```

### 自动化迭代优化工作流（Task）

迭代循环由 **Claude Code 驱动**，MCP Server 的 Task Manager 仅负责状态追踪，不做流程编排。

```
用户: "帮我优化参数，目标 CD 误差 < 0.5，最多跑 10 次"

Claude 调用 task_create(goal="CD_error < 0.5", max_iterations=10, initial_config="default")
→ 返回 task_id="xxx"

# Claude Code 驱动迭代循环：
循环开始：
  1. Claude 调用 job_submit(config_name="current") → 提交 job
  2. Claude 轮询 job_status() 直到 completed
  3. Claude 调用 analyze_result(job_id) → 获取数值指标
  4. Claude 查询 RAG 知识库获取类似案例
  5. Claude 结合指标 + RAG 判断：
     - 达标 → 调用 task_stop()，退出循环
     - 未达标且未达上限 → 调用 tune_params() 获取调参建议
     - 已达上限 → 调用 task_stop()，退出循环
  6. Claude 调用 config_patch() 应用新配置
  7. 回到步骤 1

Claude 调用 task_result(task_id="xxx")
→ 返回最终配置、调参报告、最佳结果
```

---

## 13. 后续计划

1. **Phase 1**: 核心 Job 管理（submit/status/cancel/logs）+ 确认机制（confirmation gate）
2. **Phase 2**: 配置管理（config_get/config_set/config_patch/config_diff/validate）
3. **Phase 3**: 结果分析（analyze_result/metrics，GPU 侧分析脚本）
4. **Phase 4**: 调参策略（tune_params/list_strategies）
5. **Phase 5**: Task 管理（task_create/progress/stop）
6. **Phase 6**: 代码执行（execute_pangen_script）
7. **Phase 7**: 辅助工具（health_check、job_logs 分级等）
8. **Phase 8**: 与 Claude Code/Lattice 集成测试

---

## 14. 开放设计问题（待后续决策）

### 14.1 贝叶斯黑盒优化器（已决策）

**决策**：贝叶斯黑盒优化器（如 Gaussian Process / TPE）**不放在 MCP Pangen Server 内部**。

如果后续需要，以独立服务形式提供：
- 作为单独的 MCP Server（如 `mcp-bayesian-optimizer`），Claude Code 同时连接两个 MCP Server
- 或封装为 Claude Code 的 Skill

**理由**：
- 保持 MCP Pangen Server 职责单一（Pangen 交互 + 状态管理）
- 优化器是通用能力，不应与特定领域工具耦合
- 独立部署可被其他项目复用

MCP Pangen Server 内的 Parameter Tuner 保持为轻量规则引擎（预定义策略），复杂优化决策由 Claude Code 结合外部优化器 + RAG 完成。

### 14.2 错误处理与容错（后续版本）

以下错误场景需要在后续版本中设计处理策略：

- SSH 连接失败（网络中断、认证过期）
- Job 异常退出（OOM、segfault、GPU 错误）
- GPU 节点不可达时的降级策略
- 状态文件损坏的恢复机制
- 长时间 Job 期间 Claude Code session 断开后的恢复

**状态**：⏸️ 第一版先实现 happy path，后续版本逐步补充。
