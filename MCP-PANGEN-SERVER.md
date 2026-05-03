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
  "analysis_status": null,
  "progress": 0.45,  // 0.0~1.0 浮点数，从 GPU $work_dir/progress 文件读取
  "start_time": "...",
  "end_time": null,
  "exit_code": null
}
```

**状态文件更新机制**（两层轮询的内层）：

Job 运行期间，MCP Server 后台线程定期（间隔由 `~/.pangen-mcp/config.toml` 中 `polling_interval_seconds` 配置，默认 30s）通过 SSH 检查 GPU 服务器上的进程状态，并同步更新本地状态文件。更新流程如下：

1. **读取 progress**：通过 SSH 读取 `$work_dir/progress` 文件（内容为 0.0~1.0 浮点数，如 `0.45` 表示 45%），作为 `progress` 写入本地状态文件
2. **检查进程状态**：通过 `ps -p {pid}` 确认 Pangen 进程仍在运行；若已退出，读取 `exit_code`
3. **原子写入**：将最新状态写入本地 `~/.pangen-mcp/jobs/{job_id}.json`（写临时文件 → 替换原文件），避免并发写入导致文件损坏

**SSH 读取失败处理**：
- 单次读取失败后最多重试 2 次（间隔 5s），重试均失败则记录 warning 并保持上次已知状态，不更新状态文件
- 下次轮询周期继续尝试，不会因此跳过后续更新
- 若连续 3 次轮询均失败，在 MCP Server 日志中记录 `GPU_CONNECTION_DEGRADED` 警告，提示管理员检查 GPU 节点网络

> **progress 文件约定**：Pangen 启动时应写入 `$work_dir/progress` 文件，内容为 0.0~1.0 浮点数，由 Pangen 自身维护，MCP Server 只读不写。

**MCP Server 重启后恢复**：MCP Server 启动时，自动扫描 `~/.pangen-mcp/jobs/` 中所有 `status="running"` 的 Job，恢复后台轮询线程。启动日志输出 `Resuming polling for N running jobs`。轮询线程设计为幂等可恢复，每次从状态文件的 `last_polled_at` 继续，不重复也不遗漏。

Job Manager 同时是状态文件的**写入方**，Claude Code 通过 `job_status` 接口读取文件内容作为返回结果。

**状态文件读写并发处理**：写入采用"写临时文件 → rename"（原子替换），保证并发安全。读取时 MCP Server 内部使用相同原子策略：先 rename 到临时文件再读取，确保不会读到正在写入的中间数据。

### 2.3 Config Manager（配置管理模块）

管理 Pangen 的配置文件。配置文件存放在 MCP 服务器本地，包含 JSON 格式的仿真参数。

**功能**：
- 读取/写入配置文件的指定字段
- 配置验证（参数类型、范围检查，规则表见下）
- 管理多个配置模板

**配置验证规则**：每项参数有类型、范围约束，`job_submit` 内部**自动调用** `config_validate`，验证失败则拒绝提交。

| 字段路径 | 类型 | 范围/约束 |
| -------- | ---- | --------- |
| `simulation.iterations` | 正整数 | 1~10000 |
| `gpu.nodes` | 正整数 | ≥ 1 |
| `gpu.timeout_seconds` | 非负整数 | ≥ 0，0 表示不设上限 |
| `output.format` | 字符串 | 枚举：`binary`、`text` |

**ValidationResult 结构**：
```json
{
  "valid": true,
  "errors": []
}
```
验证失败时：
```json
{
  "valid": false,
  "errors": [
    {"field": "simulation.iterations", "message": "值 -100 超出范围 [1, 10000]"}
  ]
}
```

**配置同步时机**：每次 `config_set` / `config_patch` 修改后，配置**不立即推送**，下次 `job_submit` 时随配置一起推送。若用户连续多次修改配置，仅推送最新版本（避免重复传输）。

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
| 长时 Job | 启动预计运行超过 `confirmation.long_job_threshold_minutes`（默认 60 分钟）的 job；阈值由 `~/.pangen-mcp/config.toml` 配置 |

**流程**：
```
1. Claude 调用需要确认的工具
2. MCP 服务器返回确认请求，暂停执行
3. Claude Code 向用户展示确认对话框
4. 用户批准 → MCP 服务器继续执行
5. 用户拒绝 → 返回错误
```

**pending 状态持久化**：Pending 确认请求存储在 `~/.pangen-mcp/pending/{confirmation_id}.json`，MCP Server 重启后不丢失。过期时间由 `config.toml` → `confirmation.timeout_minutes` 配置（默认 15 分钟），超时后自动 reject，返回值中注明"超时自动拒绝"。

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
1. 建立 SSH 连接（认证方式见 config.toml）
2. 推送配置文件到 GPU 集群服务器
3. 创建工作目录
4. 启动 Pangen：
   - 执行 nohup 启动命令，写入 $work_dir/startup_status.json
   - 轮询 startup_status.json 确认启动成功（最多等待 10s）后再关闭 SSH
   - 若启动超时，读取启动日志诊断失败原因，返回错误
5. 关闭 SSH 连接
6. 后台轮询状态文件
```

**SSH 连接参数**（均通过 `~/.pangen-mcp/config.toml` 配置）：

| 参数 | 默认值 | 说明 |
| ---- | ------ | ---- |
| `ssh.connect_timeout_seconds` | 10 | 连接建立超时 |
| `ssh.retry_count` | 3 | 连接失败重试次数 |
| `ssh.retry_interval_seconds` | 5 | 重试间隔 |
| `ssh.private_key_path` | `~/.ssh/id_rsa` | 密钥认证路径；多 GPU 节点可共用同一密钥或各节点配置独立密钥 |

**SSH 认证方式**：推荐密钥认证（passwordless），密码认证仅作为 fallback。若多 GPU 节点使用不同密钥，在 `config.toml` 的 `gpu_nodes` 列表中为每个节点指定其对应的私钥路径。

**nohup 启动确认机制**：为防止 SSH 关闭时 Pangen 尚未完全启动导致诊断信息丢失，Job Manager 在 nohup 执行后轮询 `$work_dir/startup_status.json`（内容如 `{"status": "running", "pid": 12345}`），确认 Pangen 真正进入运行状态后再关闭 SSH。若启动脚本中包含依赖 shell 环境的命令（如 `source ~/.bashrc`），启动确认可有效捕获这类错误。

### 2.6 Code Executor（代码执行模块）

`execute_pangen_script` 用于处理**封装工具无法覆盖的 GPU 侧复杂一次性操作**。

**执行位置：GPU 服务器端**。通过调用 Pangen Interface 的 SSH 执行器将脚本发送到 GPU 服务器执行，**Code Executor 自身不管理 SSH 连接**。

**适用场景**（封装工具无法覆盖的 GPU 侧操作）：
- 结果文件的自定义解析（如非标准二进制格式批量提取）
- GPU 端自定义数值计算和指标合并
- 跨多个结果文件的联合分析

**不适用场景（由封装工具完成）**：

| 操作 | 应使用工具 |
|------|-----------|
| 修改本地配置文件 | `config_patch` |
| 查询 job 状态 | `job_status` |
| 读取 log 文件 | `job_logs` |

**安全机制**：

| 层级       | 机制              | 说明                                                         |
| ---------- | ----------------- | ------------------------------------------------------------ |
| **第一层** | Confirmation Gate | 脚本执行前暂停，等待用户审阅并批准代码内容（**主要安全线**） |
| **第一层补充** | 危险命令预检测 | 对脚本内容提取 shell 命令，检测危险操作（`rm -rf`、`ssh.*rm`、`kill -9`、`shutdown`、`reboot`），检测到时在确认 prompt 中用警告高亮说明 |
| **第二层** | 资源限制          | 最大执行时间（`config.toml` → `code_executor.max_execution_time_seconds`，默认 600s）；超时后进程收到 SIGKILL |
| **第三层** | 结果校验          | 执行完成后读取输出和 log，验证是否成功                       |

**ScriptResult 结构**（返回给 Claude Code 的完整结果）：
```json
{
  "success": true,
  "exit_code": 0,
  "stdout": "...",
  "stderr": "...",
  "elapsed_seconds": 12.5,
  "gpu_host": "gpu-01",
  "truncated": false,
  "total_output_lines": 500,
  "returned_lines": 500
}
```
执行失败时：
```json
{
  "success": false,
  "exit_code": 1,
  "error_type": "SyntaxError|RuntimeError|SSHError|PangenError|Timeout",
  "error_message": "Python syntax error: invalid syntax",
  "stdout": "...",
  "stderr": "...",
  "elapsed_seconds": 1.2,
  "gpu_host": "gpu-01",
  "truncated": false,
  "total_output_lines": 50,
  "returned_lines": 50
}
```

**输出截断策略**：stdout/stderr 分别返回，保留**最后 1000 行**（大多数错误信息在结尾）。`truncated: true` 表示输出被截断，`total_output_lines` 告知实际总行数。

> **注意**：本服务为内部使用，不设进程级沙箱。Confirmation Gate（人工审阅代码 + 危险命令预检测）是主要安全屏障。

**与 Job Manager 的区别**：Job Manager 管理的是完整的 Pangen 仿真任务（有生命周期、状态文件），Code Executor 处理的是任意的单次 Python 脚本调用，两者互补而非包含关系。

### 2.7 Result Analyzer（结果分析模块）

Job 完成后，分析 Pangen 的输出结果，提取数值指标。

**执行位置：GPU 服务器端**。结果文件为 GB 级二进制，不传输到本地。MCP Server 通过 SSH 在 GPU 服务器上执行预部署的分析脚本，仅将提取出的数值指标（KB 级 JSON）传回本地。

**分析脚本部署**：
- 脚本存放在 `~/.pangen-mcp/analyzer/` 目录
- MCP Server 启动时自动同步到所有 GPU 节点，路径由 `config.toml` → `analyzer.script_path` 配置（默认 `/opt/pangen-mcp/analyze.py`）
- 脚本版本号写在文件头（如 `# version: 1.2.0`），MCP Server 启动时校验版本，不匹配则自动更新
- 增加 `analyzer_version` 接口查询当前 GPU 上的脚本版本

**compare_results 实现**：在 GPU 侧执行，先对两个 Job 分别执行分析脚本（如尚未分析），提取 metrics 后对比 metrics（而非原始二进制文件）。输入为两个 `job_id`，输出为 CD_error 差值、良率对比等结构化对比结果。

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

**调参策略定义格式**：使用 YAML（人类可读，支持注释，便于维护）。策略文件存放在 `~/.pangen-mcp/strategies/` 下，每个策略一个 `.yaml` 文件。

**策略文件 schema**：
```yaml
name: refine_grid
description: "网格细化策略，适用于收敛慢的场景"
trigger_condition:
  type: convergence_rate   # convergence_rate | residual | local_error
  threshold: 0.05
  window: 3                # 连续 N 次迭代
adjustment:
  grid_spacing: "* 0.8"
  iterations: "* 1.0"
```

**调参触发条件的量化标准**：由 Result Analyzer 完成分析后，metrics 作为 `tune_params` 的输入数据，触发条件的量化计算在 Parameter Tuner 内部进行：

| 条件       | 计算方式                               | 阈值配置字段                |
| ---------- | -------------------------------------- | -------------------------- |
| 收敛慢     | 连续 3 次迭代的 `cd_error` 下降率 < 5% | `convergence_threshold=0.05` |
| 迭代未收敛 | 最后 2 次迭代的 `cd_error` 差异 < 0.01 | `residual_threshold=0.01`  |
| 局部误差大 | 任意区域的 CD_error > 1.5              | `local_error_threshold=1.5` |

**调参历史存储**：每次调用 `tune_params` 和 `apply_tuning` 时，记录调参事件到 `~/.pangen-mcp/tuning_history/` 目录下，每个事件一个 JSON 文件。可通过 `task_result` 获取当前 Task 的完整调参历史（`iterations_summary`），无需独立查询接口。

| 策略名                | 触发条件   | 调整方式       |
| --------------------- | ---------- | -------------- |
| `refine_grid`         | 收敛慢     | 网格间距 × 0.8 |
| `increase_iterations` | 迭代未收敛 | 迭代次数 × 1.5 |
| `adjust_weight`       | 局部误差大 | 调整权重参数   |
| `restart`             | 多次失败   | 重置到初始配置 |

**工作流程**（由 Claude Code 驱动调用）：
```
1. Claude Code 调用 tune_params(job_id, strategy?)
2. Parameter Tuner 验证 job_id 对应的 Job 分析状态（见下方"调用前验证"）
3. 验证通过后，加载策略 + 历史结果
4. 基于规则生成配置建议（不直接写入，返回给 Claude Code）
5. Claude Code 综合 Tuner 建议 + RAG + 外部优化器结果，决定是否采纳
6. Claude Code 调用 config_patch() 应用最终配置
7. 调参记录写入 Task 历史
```

**调用前验证（Pre-call Validation）**

`tune_params(job_id)` 被调用时，**必须验证**对应 Job 的结果分析已完成：

- 读取 `~/.pangen-mcp/jobs/{job_id}.json` 中的 `analysis_status` 字段
- 验证逻辑：
  - `analysis_status == "completed"` → 继续执行调参逻辑
  - `analysis_status != "completed"`（包括 `"pending"`、`"failed"`、字段不存在）→ 返回明确错误
- 错误响应格式：
  ```json
  {
    "error": "ANALYSIS_NOT_AVAILABLE",
    "code": "TUNER_001",
    "message": "Job xxx 的分析结果不可用，无法生成调参建议",
    "details": {
      "job_id": "xxx",
      "analysis_status": "pending" | "failed" | null
    }
  }
  ```
- Claude Code 收到 `ANALYSIS_NOT_AVAILABLE` 后，应先调用 `analyze_result(job_id)` 补上分析步骤，再重新调用 `tune_params`

> 此验证机制确保调参建议始终基于真实数据，避免 Parameter Tuner 使用陈旧或空数据产生误导性输出。

### 2.9 Task Manager（任务管理模块）

管理迭代优化任务的**状态追踪**，包含多个相关的 Job。Task Manager 不驱动迭代循环，循环由 Claude Code 外部驱动。

**功能**：
- 创建调参任务（Task），记录目标和迭代上限
- 追踪任务进度（已完成轮次、当前状态、最佳结果）
- 管理任务内的 job 链和调参历史
- 支持暂停/恢复/终止任务标记
- **检测空 Task（stale）状态**，防止 Task 创建后长期无 job 提交

**空 Task 检测机制（Stale Detection）**

为防止 `task_create` 返回后 Claude Code 忘记或延迟调用 `job_submit` 导致 Task 永久停留在 `iteration=0` 的 `running` 状态，Task Manager 通过后台线程检测并标记空 Task：

- **触发条件**：Task `status == "running"` 且 `jobs == []` 且自创建起超过 `stale_threshold`（默认 10 分钟，可配置）
- **处理结果**：将 Task 状态更新为 `stale`，并在 `task_progress` 中返回提示
- **执行位置**：与 Job 状态轮询共用同一后台线程，无额外资源开销
- **后续处理**：Claude Code 可通过 `task_progress` 感知 `stale` 状态，自行决定是继续调用 `job_submit` 还是调用 `task_stop` 终止任务

> 状态漂移说明：检测线程与外层查询之间最大存在 30s 延迟，stale 状态最晚会在创建后 `stale_threshold + 30s` 左右被检测到。

**Task 状态文件**（保存在本地 `~/.pangen-mcp/tasks/`）：
```json
{
  "id": "task-uuid",
  "goal": "优化参数达到 CD_error < 0.5",
  "max_iterations": 10,
  "current_iteration": 3,
  "status": "running|completed|failed|paused|stale",
  "current_job_id": null,
  "jobs": ["job-1", "job-2", "job-3"],
  "best_result": {
    "job_id": "job-2",
    "cd_error": 0.67
  },
  "start_time": "...",
  "history": [
    {
      "iteration": 1,
      "config": {...},
      "metrics": {"cd_error": 1.2},
      "strategy": "refine_grid",
      "outcome": "failed",
      "reason": "cd_error未改善"
    },
    {
      "iteration": 2,
      "config": {...},
      "metrics": {"cd_error": 0.67},
      "strategy": "increase_iterations",
      "outcome": "success"
    }
  ]
}
```

**TaskResult 结构**（`task_result` 接口返回值）：
```json
{
  "task_id": "...",
  "goal": "优化参数达到 CD_error < 0.5",
  "outcome": "achieved|partial|failed",
  "final_config": {...},
  "best_result": {
    "job_id": "...",
    "metrics": {"cd_error_mean": 0.45, "cd_error_max": 1.2, "yield_estimate": 91.5}
  },
  "iterations_summary": [
    {
      "iteration": 1,
      "job_id": "job-001",
      "config": {...},
      "metrics": {...},
      "tuning_applied": "refine_grid",
      "outcome": "failed",
      "reason": "cd_error未改善"
    }
  ],
  "total_iterations": 10,
  "elapsed_time_seconds": 7200,
  "recommendation": "建议将 refine_grid 策略与 increase_iterations 结合使用"
}
```
`report_format` 参数支持 `"json"`（默认）或 `"markdown"`。

**Task 内 Job 串行约束**：Task 下的 Job **不允许并发提交**。`job_submit` 时验证当前 Task 的 `current_job_id` 是否已完成；若前一个 Job 尚未完成，返回 `{"error": "TASK_JOB_IN_PROGRESS", "current_job_id": "xxx"}`。第一个 Job 提交时 `current_job_id` 为 null，不受此限制。

### 2.10 Local Storage（本地存储）

MCP 服务器本地的文件存储。

| 路径                                 | 内容               |
| ------------------------------------ | ------------------ |
| `~/.pangen-mcp/jobs/{job_id}.json`   | Job 状态文件       |
| `~/.pangen-mcp/tasks/{task_id}.json` | Task 状态文件      |
| `~/.pangen-mcp/configs/`             | Pangen 配置模板    |
| `~/.pangen-mcp/strategies/`          | 调参策略定义       |
| `~/.pangen-mcp/config.toml`          | MCP Server 自身配置（轮询间隔等） |
| `~/.pangen-mcp/logs/`                | MCP 服务器自身日志 |
| `~/.pangen-mcp/pending/`             | Pending 确认状态文件（MCP Server 重启不丢失） |

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
| `task_resume`   | `task_id: string`                                              | `boolean`           | 恢复 stale 或 paused 的任务，继续迭代 |
| `task_stop`     | `task_id: string`                                              | `boolean`           | 停止任务               |
| `task_list`     | 无                                                             | `TaskInfo[]`        | 列出所有任务           |
| `task_result`   | `task_id: string`                                              | `TaskResult`        | 获取任务最终结果和报告 |

### 4.2 Job 工具（Job Management）

| 工具名       | 参数                                    | 返回               | 说明           |
| ------------ | --------------------------------------- | ------------------ | -------------- |
| `job_submit` | `config_name: string, timeout_seconds?: number` | `{job_id: string}` | 提交 job；`timeout_seconds=0` 表示不设上限 |
| `job_status` | `job_id: string`                        | `JobStatus`        | 查询状态                               |
| `job_list`   | 无                                      | `JobInfo[]`        | 列出所有 job                           |
| `job_cancel` | `job_id: string`                        | `JobCancelResult`  | 取消 job；若进程已不存在则更新状态文件  |
| `job_result` | `job_id: string`                        | `ResultMetadata`   | 获取结果元信息 |
| `job_logs`   | `job_id: string, offset?: number`       | `string`           | 读取 log       |

### 4.3 Config 工具（Configuration Management）

| 工具名            | 参数                                 | 返回               | 说明                       |
| ----------------- | ------------------------------------ | ------------------ | -------------------------- |
| `config_get`      | `key: string`                        | `string`           | 读取配置字段               |
| `config_set`      | `key: string, value: string`         | `boolean`          | 写配置字段；注意：该变更在**下次 job_submit 时生效**，当前运行的 Job 不受影响 |
| `config_patch`    | `config_name: string, patch: object` | `PatchResult`     | 批量修改多个字段（**文件级原子性**：全部成功或全部回滚） |
| `config_diff`     | `config_a: string, config_b: string` | `DiffResult`       | 对比两个配置的差异         |
| `config_list`     | 无                                   | `ConfigInfo[]`     | 列出所有配置模板           |
| `config_validate` | `config_name: string`                | `ValidationResult` | 验证配置合法性             |

**说明**：每次 `config_set` / `config_patch` 修改配置时，自动备份上一版本到 `~/.pangen-mcp/configs/{name}.{timestamp}.bak`，支持回溯。

**PatchResult 结构**（`config_patch` 返回值）：
```json
{
  "success": true,
  "patched_fields": ["simulation.iterations", "gpu.nodes"]
}
```
原子性：全部成功或全部回滚——若任何字段验证失败，整次 patch 不写入。若部分写入后遭遇磁盘错误，则回滚到操作前状态。

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
| `health_check` | 无   | `HealthStatus` | 检查 SSH 连通性、GPU 节点状态、磁盘空间、Pangen 可执行文件版本等；检查顺序：SSH 连通性优先（不通则其他检查标记为 skipped） |

**HealthStatus 结构**：
```json
{
  "overall": "healthy|degraded|unhealthy",
  "checked_at": "2026-05-01T10:00:00Z",
  "checks": [
    {"name": "ssh_gpu01", "host": "gpu-01", "status": "ok", "latency_ms": 120},
    {"name": "ssh_gpu02", "host": "gpu-02", "status": "ok", "latency_ms": 118},
    {"name": "local_disk", "status": "ok", "free_gb": 450, "path": "/"},
    {"name": "gpu_disk", "status": "ok", "free_gb": 1200, "host": "gpu-01", "path": "/data"},
    {"name": "pangen_binary", "status": "ok", "version": "2.4.1", "host": "gpu-01"}
  ],
  "warnings": ["gpu-02 磁盘空间低于 20%"]
}
```
`overall` 判断逻辑：`degraded`（存在 warning）、`unhealthy`（任意 check 失败）、`healthy`（全部 ok）。

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
  "analysis_status": null,
  "start_time": "2026-05-01T10:00:00Z",
  "end_time": null,
  "exit_code": null,
  "progress": 0.45  // 0.0~1.0 浮点数，从 GPU $work_dir/progress 文件读取
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

`job_cancel` 时，先通过 SSH 执行 `ps -p {pid}` 检查 GPU 侧进程是否存在：
- 进程存在 → 发送 SIGTERM → 返回 `{"cancelled": true, "signal_sent": true, "gpu_pid": xxx}`
- 进程不存在 → 更新状态文件为 `cancelled` → 返回 `{"cancelled": false, "reason": "process_already_exited", "status_updated_to": "cancelled"}`

---

## 6. 状态查询与轮询机制

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

### 6.3 状态漂移 Limitations

内层轮询（~30s）和外层查询之间存在最大 30s 的状态漂移窗口。若 Pangen 在此窗口内异常退出，Claude Code 获取到的状态可能仍为 `running`。为缓解此问题，状态文件更新时同步检查 GPU 上 `$work_dir/log` 的最后几行，若出现 `error`、`killed`、`segfault`、`OOM`、`GPU error` 等关键词，**立即将状态文件更新为 `failed`**，不再等待下次轮询。

### 6.4 Task 恢复机制

Claude Code 断开后重连，可通过 `task_progress(task_id)` 获取当前状态（`current_iteration`、`jobs[]` 链、`history`），从断点继续。Task Manager 不做自动恢复，循环的 resume 由 Claude Code 手动驱动。

### 6.5 典型工作流

```
用户: "启动 pangen job，用配置 A"

Claude 调用 job_submit → 返回 job_id="xxx"
Claude 调用 job_status(job_id="xxx") → status="running"
Claude 调用 job_logs(job_id="xxx") → 读取最新 log
Claude 继续轮询 job_status 直到 status="completed"（退避策略：pending 时每 5s，running 前 2 分钟每 10s，之后每 30s）
Claude 调用 job_result(job_id="xxx") → 返回结果路径/大小/格式
```

---

## 7. 确认机制

确认机制与第 6 节轮询机制**相互独立**：轮询负责状态感知，确认负责高风险操作的人工审批。"长时 Job 确认"属于确认机制（主动用户交互），不受轮询状态影响。

三种场景需要用户确认：

| 场景       | 说明                                                 |
| ---------- | ---------------------------------------------------- |
| 代码执行前 | Claude 执行 `execute_pangen_script` 前需用户批准代码 |
| 敏感操作   | 删除数据、修改集群配置等高风险操作                   |
| 长时间 Job | 启动预计运行超过 `confirmation.long_job_threshold_minutes`（默认 60 分钟）的 Job；阈值由 `~/.pangen-mcp/config.toml` 配置 |

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

**正文模块编号与代码路径映射**：

| 正文模块编号 | 正文模块名           | 代码路径                    |
| ------------ | -------------------- | -------------------------- |
| 2.1          | MCP Protocol Layer   | `src/mcp/mod.rs`           |
| 2.2          | Job Manager          | `src/job/manager.rs`       |
| 2.3          | Config Manager       | `src/config/mod.rs`        |
| 2.4          | Confirmation Gate    | `src/confirmation/gate.rs` |
| 2.5          | Pangen Interface     | `src/pangen/mod.rs`        |
| 2.6          | Code Executor        | `src/code/executor.rs`     |
| 2.7          | Result Analyzer      | `src/analyzer/mod.rs`      |
| 2.8          | Parameter Tuner      | `src/tuner/engine.rs`      |
| 2.9          | Task Manager         | `src/task/manager.rs`      |
| 2.10         | Local Storage        | 各模块共用 `~/.pangen-mcp/` |

**目录结构**：
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
2. 配置文件的字段定义和验证规则（当前已定义基本规则表，见 2.3 节；完整字段集待与 Pangen 团队确认）
3. Log 文件的格式和解析方式
4. 结果文件的具体格式（OPC 相关二进制格式）
5. 结果分析中数值指标的具体定义（如 CD 误差的计算方式）
6. GPU 侧结果分析脚本的依赖环境

### 调参相关
7. 初始调参策略由谁提供（用户提供 / MCP 内置 / 从 RAG 学习）
8. 策略不奏效时的回退机制

### 已决定的事项
- ✅ SSH 认证方式：推荐密钥认证（passwordless），`ssh.private_key_path` 默认 `~/.ssh/id_rsa`；多 GPU 节点可共用或各配独立密钥；密码认证仅作 fallback
- ✅ SSH 连接参数配置化：`ssh.connect_timeout_seconds`（默认 10）、`ssh.retry_count`（默认 3）、`ssh.retry_interval_seconds`（默认 5）
- ✅ nohup 启动后轮询 `$work_dir/startup_status.json` 确认启动成功后再关闭 SSH，防止启动失败诊断信息丢失
- ✅ Pending 确认状态持久化到 `~/.pangen-mcp/pending/`，MCP Server 重启不丢失；过期时间 `confirmation.timeout_minutes`（默认 15 分钟），超时自动 reject
- ✅ 长时 Job 确认阈值配置化：`confirmation.long_job_threshold_minutes`（默认 60 分钟）
- ✅ 危险命令预检测：脚本内容检测 `rm -rf`、`ssh.*rm`、`kill -9`、`shutdown`、`reboot` 等，检测到时在确认 prompt 中高亮警告
- ✅ config_patch 为文件级原子性（全部成功或全部回滚）；`job_submit` 内部自动调用 `config_validate`，验证失败拒绝提交
- ✅ `job_submit` 的 timeout 参数单位为秒（`timeout_seconds`），`timeout_seconds=0` 表示不设上限
- ✅ execute_pangen_script 超时配置化：`code_executor.max_execution_time_seconds`（默认 600s），超时 SIGKILL；输出截断保留最后 1000 行，stdout/stderr 分别返回
- ✅ compare_results 在 GPU 侧执行（提取两个 Job 的 metrics 后对比），不传输 GB 级二进制文件
- ✅ Analysis 脚本部署：MCP Server 启动时同步到所有 GPU 节点，路径 `config.toml` → `analyzer.script_path`，版本号写在文件头，启动时校验
- ✅ 迭代循环由 Claude Code 驱动，Task Manager 仅做状态追踪
- ✅ 轮询采用两层架构（MCP Server 后台线程 + Claude Code 按需读取）
- ✅ 状态漂移 limitations 已标注（内层轮询与外层查询之间最多 30s 延迟）
- ✅ 第一版每次新开 SSH 连接，轮询频率不高；后续 benchmark 再优化连接复用
- ✅ Result Analyzer 在 GPU 服务器端执行
- ✅ Confirmation Gate 采用两步调用（pending → approve/reject）
- ✅ 多 GPU 调度由 Pangen 自身负责（启动参数文件指定）
- ✅ 不设沙箱，Confirmation Gate 作为主要安全线（内部使用）
- ✅ 贝叶斯优化器不放在 MCP Server 内，如需要则作为独立 MCP Server 或 Skill
- ✅ Task 可恢复：Claude Code 断开后重连，通过 task_progress 获取状态，从断点继续；Task Manager 不做自动恢复
- ✅ 空 Task 检测：Task 创建后超过 stale_threshold（默认 10 分钟）仍无首个 job 提交，自动标记为 stale；与 job 状态轮询共用后台线程
- ✅ tune_params 调用前验证：tune_params(job_id) 必须验证 Job 的 analysis_status == "completed"，否则返回 ANALYSIS_NOT_AVAILABLE 错误；Job 状态文件新增 analysis_status 字段
- ✅ 错误处理基线：记录错误 + 返回失败状态 + 留给 Claude Code 决策
- ✅ config_set / config_patch 修改本地配置文件，不自动推送；下次 job_submit 时随配置一起推送
- ✅ health_check 暂不覆盖 license server / NFS 挂载点，后续遇到再调整
- ✅ execute_pangen_script 动态接收脚本内容，通过 SSH 发送到 GPU 服务器执行
- ✅ progress 文件约定：Pangen 启动时写入 `$work_dir/progress`，内容为 0.0~1.0 浮点数，MCP Server 只读
- ✅ polling_interval_seconds 配置化：从 `~/.pangen-mcp/config.toml` 读取，默认 30s
- ✅ SSH 读取失败处理：单次失败重试 2 次（间隔 5s），均失败则记录 warning 并保持上次状态；连续 3 次轮询失败记录 `GPU_CONNECTION_DEGRADED` 警告

---

## 12. 典型工作流示例

### 手动调参工作流（用户单次 job）

```
用户: "用配置 A 运行一次 pangen"

Claude 调用 job_submit(config_name="A")
→ 返回 job_id="xxx"

Claude 轮询 job_status 直到 completed（退避策略：pending 时每 5s，running 前 2 分钟每 10s，之后每 30s，completed/failed 时停止）

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

# Claude Code 驱动迭代循环（轮询退避策略：pending 每 5s，running 前 2 分钟每 10s，之后每 30s）：
循环开始：
  1. Claude 调用 job_submit(config_name="current") → 提交 job
  2. Claude 轮询 job_status() 直到 completed（退避策略同上）
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

1. **Phase 1**: 核心 Job 管理（submit/status/cancel/logs）+ 确认机制（confirmation gate，包含 `confirmation_approve`/`confirmation_reject`）
2. **Phase 2**: 配置管理（config_get/config_set/config_patch/config_diff/validate）
3. **Phase 3**: 结果分析（analyze_result/metrics，GPU 侧分析脚本）
4. **Phase 4**: 调参策略（tune_params/list_strategies）
5. **Phase 5**: Task 管理（task_create/progress/stop）
6. **Phase 6**: 代码执行（execute_pangen_script，依赖 Phase 1 的 Confirmation Gate）
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

### 14.2 错误处理与容错（第一版基线）

第一版采用「记录错误 + 返回失败状态 + 留给 Claude Code 决策」的处理策略。以下错误场景有基本覆盖：

| 场景 | 处理方式 |
|------|---------|
| SSH 连接失败 | 按 `ssh.retry_count`（默认 3 次）和 `ssh.retry_interval_seconds` 重试，仍失败返回 error，Claude Code 决策是否继续 |
| Job 异常退出（OOM/segfault/GPU 错误） | 状态文件更新为 failed，写入 exit_code，Claude Code 查询后决策 |
| GPU 节点不可达 | job_status 返回 error，Claude Code 查询 log 定位 |
| 状态文件损坏 | 检测到格式错误时返回 error，不尝试自动修复 |

以下场景暂不处理，留待后续版本：
- Claude Code session 断开后的 auto-recovery（见 6.4 Task 恢复机制）
- 状态文件的原子性修复
- 网络中断后的自动重连
