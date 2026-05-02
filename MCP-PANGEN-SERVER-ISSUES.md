# MCP Pangen Server 设计文档 — 待改进问题清单

> 分析日期：2026-05-02
> 源文档：MCP-PANGEN-SERVER.md

---

## 分类索引

| 分类                                                | 优先级 | 问题数量 |
| --------------------------------------------------- | ------ | -------- |
| [一、模块边界与职责](#一模块边界与职责)             | P1~P2  | 3        |
| [二、数据流一致性](#二数据流一致性)                 | P0~P1  | 3        |
| [三、安全机制](#三安全机制)                         | P0~P1  | 4        |
| [四、SSH 连接管理](#四ssh-连接管理)                 | P0~P1  | 3        |
| [五、状态一致性与故障恢复](#五状态一致性与故障恢复) | P1     | 3        |
| [六、Config Manager](#六config-manager)             | P1~P2  | 3        |
| [七、Code Executor](#七code-executor)               | P1~P2  | 3        |
| [八、Parameter Tuner](#八parameter-tuner)           | P2     | 3        |
| [九、Result Analyzer](#九result-analyzer)           | P2     | 2        |
| [十、工具定义完整性](#十工具定义完整性)             | P0~P2  | 4        |
| [十一、迭代驱动逻辑](#十一迭代驱动逻辑)             | P2     | 3        |
| [十二、文档一致性问题](#十二文档一致性问题)         | P2     | 3        |
| [十三、Phase 计划问题](#十三phase-计划问题)         | P3     | 2        |
| [十四、边界情况与错误处理](#十四边界情况与错误处理) | P1~P2  | 3        |

---

### [P2] 1.2 Task Manager 不做流程编排，但首个 Job 提交无保障机制

**问题描述**：`task_create` 返回 `task_id` 后，如果 Claude Code 忘记或延迟调用 `job_submit`，Task 会永久停留在 `iteration=0` 的 `running` 状态，无超时、无告警。

**改进建议**：
- 增加"空 Task 检测"机制：Task 创建超过 N 分钟（可配置，默认 10 分钟）仍未提交第一个 Job，自动将状态更新为 `stale`，并在 `task_progress` 中提示
- 或在文档工作流中明确要求 `task_create` 和首个 `job_submit` 必须在同一操作序列中执行

**所在位置**：2.9 节、12.2 节

---

### [P2] 1.3 Result Analyzer 与 Parameter Tuner 调用协议缺失

**问题描述**：`tune_params(job_id)` 可在任何时候调用，即使上一次 `analyze_result` 已失败或未执行。Parameter Tuner 内部如何判定该 job 是否有可用结果？使用陈旧数据还是返回错误？

**改进建议**：
- `tune_params` 调用时，内部读取 Task 状态文件，验证 `job_id` 对应的 Job 已完成且 `analyze_result` 已执行
- 验证失败时返回明确错误：`{"error": "ANALYSIS_NOT_AVAILABLE", "job_id": "xxx"}`
- 或增加 `analysis_id` 参数，`tune_params` 要求传入上一步 `analyze_result` 的结果，确保数据链路可追溯

**所在位置**：2.7 节、2.8 节

---

## 二、数据流一致性

### [P0] 2.1 progress 字段类型定义冲突

**问题描述**：状态文件中 progress 字段的类型定义存在两处不一致。
- 2.2 节（完整 schema 示例）：`"progress": "45%"` — 明确标注为字符串，带百分号
- 5.2 节（精简 schema）：`"progress": "45%"` — 同样标注为字符串

描述中却说"通过 SSH 读取 GPU 上进程/Pangen 的实时状态（如 progress、exit_code）"——没有说明读取后如何格式化转换。

如果存字符串，数值比较（如 `progress > "50%"`）在代码中需要 parse 后再比较，容易出错。

**改进建议**：
- 统一使用浮点数 0.0~1.0 存储，避免字符串比较问题
- 在 `job_status` 返回给 Claude Code 前做格式化（转为百分比字符串）
- 明确 Pangen 侧 progress 是百分比(0-100)还是小数(0-1)，两端的转换在 Pangen Interface 中处理

**所在位置**：2.2 节（状态文件 schema）、5.2 节（状态文件）

---

### [P1] 2.2 GPU 进度→本地状态文件的同步机制缺失

**问题描述**：
- **Pangen 如何暴露 progress？** 是写入特定文件（如 `$work_dir/progress`）还是通过 log 解析？未说明
- **轮询频率**描述为"~30s"，hardcode 还是可配置？
- **网络异常处理**：SSH 读取失败（如 GPU 网络抖动），是跳过本次更新还是重试？状态文件是否会留下 stale 数据？

**改进建议**：
- 在 Pangen Interface 中明确定义 progress 的读取方式（推荐：解析 `$work_dir/progress` 文件，内容为 0.0~1.0 浮点数）
- 将轮询间隔配置化：`~/.pangen-mcp/config.toml` 中 `polling_interval_seconds = 30`
- SSH 读取失败时最多重试 2 次，超过则记录 warning 并保持上次已知状态，下次轮询时重试

**所在位置**：2.2 节

---

### [P1] 2.3 job_logs 的 offset 语义未定义

**问题描述**：`job_logs(offset?: number)` 的 offset 语义未定义：
- 是行号（offset=100 表示从第 100 行开始）？
- 还是字节偏移（offset=4096 表示从文件第 4096 字节开始）？
- 还是其他？

如果使用行号 offset，GPU 侧的 log 文件被追加写入后，行号动态增长，"续读"语义清晰。但如果使用字节偏移，追加写入后字节偏移不变，新内容就读不到了。

**改进建议**：
- 推荐使用**字节偏移**（更通用，支持随机访问）
- 返回时包含 `next_offset` 字段：`{"content": "...", "offset": 4096, "next_offset": 8192, "eof": false}`
- 明确 Pangen log 文件格式规范（如：追加写入不截断，每行不超过 8KB）

**所在位置**：2.5 节、4.2 节

---

## 三、安全机制

### [P0] 3.1 Confirmation Gate 无法拦截 Claude Code 生成的有害脚本

**问题描述**：Confirmation Gate 依赖"用户审阅代码"，但存在根本性漏洞。

**攻击场景**：Claude Code 生成了包含 SSH 危险命令的脚本：
```python
import subprocess
# 看起来是正常的 Pangen 操作
subprocess.run("ssh gpu-01 'rm -rf /data/job-xxx/* && nohup pangen ...'", shell=True)
```
Confirmation Gate 仅展示"此脚本将执行 Python 代码"，prompt 没有对脚本内容做危险操作检测。用户（尤其是非技术人员）难以识别 SSH 命令的危险性，base64 编码的 payload 更是无法审阅。

**改进建议**：
- MCP Server 对脚本内容做**危险命令预检测**：提取所有 shell 命令、SSH 调用、系统级调用（`rm -rf`、`ssh.*rm`、`kill -9`、`shutdown`）
- Confirmation prompt 中明确说明操作摘要：如"此脚本包含 SSH 命令，将删除 GPU 上的作业数据"，高危命令加红色警告
- 对于包含远程执行的脚本，默认要求更高级别的用户确认（如需要输入"CONFIRM"字样）

**所在位置**：2.4 节、7 节

---

### [P1] 3.2 长时间 Job 确认阈值不一致

**问题描述**：
- 2.4 节：超过 30 分钟的 Job 需确认
- 7 节：超过 1 小时的 Job 需确认

阈值相差一倍，Claude Code 实现时以哪个为准？用户在不同章节看到不同的阈值会产生困惑。

**改进建议**：
- 统一为可配置参数：`~/.pangen-mcp/config.toml` 中 `confirmation.long_job_threshold_minutes = 60`
- 明确这个阈值基于"预估时间"还是"历史平均时间"，如果是预估，需要说明偏差处理方式

**所在位置**：2.4 节、7 节

---

### [P1] 3.3 pending 状态存储与超时机制完全缺失

**问题描述**：

**问题 1：存储位置**。MCP Server 重启后，pending 操作是否丢失？内存 Map 存储 → 服务器重启后丢失 → 用户批准了但操作已不存在。

**问题 2：超时机制**。用户发起操作后如果永远不批准，pending 操作是否会过期？过期时间是 hardcoded 10 分钟还是可配置？过期后是自动 reject 还是保持等待？

**改进建议**：
- pending 状态必须持久化（文件存储：`~/.pangen-mcp/pending/`），而非仅存内存
- 增加 `confirmation_timeout_minutes` 配置项，默认 15 分钟
- pending 过期后自动 reject：`confirmation_reject` 返回值中注明"超时自动拒绝"

**所在位置**：2.4 节、7 节

---

## 四、SSH 连接管理

### [P0] 4.1 SSH 认证方式未明确（核心基础设施）

**问题描述**：SSH 认证方式列在"待确认事项"中未决定。这是整个系统的基础设施问题——没有认证方式，SSH 连接无法建立。

**改进建议**：
- 至少明确优先级：推荐密钥认证（passwordless），密码认证仅作为 fallback
- 配置项：`~/.pangen-mcp/config.toml` 中 `ssh.private_key_path = "~/.ssh/id_rsa"`
- 如果支持多 GPU 节点，明确如何管理多个 SSH 密钥（每个节点不同密钥？还是统一密钥？）
- 明确 SSH agent forwarding 是否支持（如需要访问远程 Git 仓库）

**所在位置**：11 节（待确认事项）

---

### [P1] 4.2 SSH 连接超时与重试策略未定义

**问题描述**：
- SSH 连接建立是否有 timeout？（如：30 秒无响应则报错）
- 连接被拒绝时是否重试？重试几次？间隔多久？
- GPU 节点网络暂时不可达（如交换机抖动），是否自动重试？

**改进建议**：
- 定义 SSH 连接 timeout 建议值：`ssh.connect_timeout_seconds = 10`
- 定义重试策略：`ssh.retry_count = 3`，`ssh.retry_interval_seconds = 5`
- 区分"连接失败"（立即报错）和"连接超时"（重试后报错）的返回错误码

**所在位置**：2.5 节

---

### [P1] 4.3 nohup 执行后立即关闭 SSH 的潜在竞态

**问题描述**：SSH 执行流程为：
```
1. 建立 SSH 连接
2. nohup 启动 Pangen
3. 关闭 SSH 连接
```
nohup 本身确保进程忽略 SIGHUP，但以下情况可能导致问题：
- Pangen 启动脚本中包含 `source ~/.bashrc` 等依赖 shell 环境的命令
- Pangen 有 daemonize 逻辑（写 PID 文件），SSH 关闭和 PID 文件写入之间可能有竞态
- 如果 Pangen 启动失败，SSH 已关闭，诊断信息无法回传

**改进建议**：
- 在 SSH 命令中加入启动确认：`nohup pangen ... > /dev/null 2>&1 & sleep 2 && cat $work_dir/startup_status.json`
- 轮询 `$work_dir/startup_status.json` 确认 Pangen 真正启动后再关闭 SSH
- 或使用更可靠的方式：让 Pangen 启动脚本将状态写入文件后退出，轮询该文件确认后再关闭 SSH

**所在位置**：2.5 节（SSH 执行流程）

---

## 五、状态一致性与故障恢复

### [P1] 5.1 MCP Server 重启后后台线程状态丢失

**问题描述**：内层轮询由"MCP Server 后台线程"执行。MCP Server 重启后：
- 后台线程是否自动重建？
- 如何知道还有哪些 Job 正在运行（需要继续轮询）？
- 如果状态文件记录 `status="running"`，但重启后没有恢复轮询线程，该 Job 会永远失去状态更新

**改进建议**：
- MCP Server 启动时，扫描 `~/.pangen-mcp/jobs/` 中所有 `status="running"` 的 Job，自动恢复后台轮询线程
- MCP Server 启动日志输出：`Resuming polling for N running jobs`
- 轮询线程设计为幂等可恢复，每次轮询从状态文件的 `last_polled_at` 继续

**所在位置**：2.2 节（状态文件更新机制）

---

### [P1] 5.2 状态文件读写并发处理不够明确

**问题描述**：原子写入（写临时文件 → rename）解决了写入并发问题，但没有讨论读取并发：
- 后台轮询线程正在 rename 状态文件时，Claude Code 同时调用 `job_status` 读取同一个文件
- 如果读到了正在写入的临时文件内容（未 rename 完成），会读到损坏数据

**改进建议**：
- 在文档中明确写入策略为"rename atomic"
- 读取时建议使用"先 rename 到临时文件再读取"策略
- 或增加状态文件版本号字段（`version: int`），读取时检查版本，连续两次读取版本不变则数据有效

**所在位置**：2.2 节（状态文件更新机制）

---

### [P1] 5.3 状态漂移的兜底验证逻辑缺失

**问题描述**：文档已标注状态漂移（最多 30s），并建议"结合 `job_logs` 做辅助判断"，但没有给出具体实现逻辑。

**实际场景**：状态文件显示 `status="running"`，但 GPU 上的 Pangen 已异常退出（OOM）。在这 30s 内，Claude Code 可能继续提交新 Job 或等待。

**改进建议**：
- 定义状态验证逻辑：如果 `status="running"` 但 `job_logs` 中出现 `error`、`killed`、`segfault`、`OOM`、`GPU error` 等关键词，应将状态文件更新为 `failed`
- 在状态文件 schema 中增加 `log_has_warnings` 布尔字段，轮询时同步检查
- 这实际上是给 Pangen Interface 的"状态检查"增加了 log 辅助验证步骤

**所在位置**：6.3 节（状态漂移 Limitations）

---

## 六、Config Manager

### [P1] 6.1 GPU 配置文件同步时机不明确

**问题描述**：文档说"下次 job_submit 时随配置一起推送"，但存在以下歧义：
- 用户连续调用 `config_set` 3 次，然后才调用 `job_submit`，配置被推送 3 次还是仅推送最新版本？（后者更优，避免重复）
- 如果用户修改配置后**不想**立即推送（想等其他参数一起改再一起推送），有什么机制？

**改进建议**：
- 增加"草稿模式"：`config_set` 可选参数 `draft: boolean = false`，草稿配置不立即推送，`job_submit` 时推送所有非草稿配置
- 或明确"实时推送"策略：每次 `config_set` 后立即同步到 GPU 本地（适合开发调试模式）
- 增加 `config_sync_status(config_name)` 查询接口：用户可查看本地配置和 GPU 侧配置是否一致

**所在位置**：2.3 节、6.1 节

---

### [P2] 6.2 运行中 Job 的配置无法修改

**问题描述**：如果一个 Job 正在运行（Pangen 进程已在 GPU 上），用户想临时增加迭代次数——文档没有定义这个场景。

**改进建议**：
- 明确策略：**推荐不支持**（运行中 Job 配置修改复杂度高，容易导致状态不一致）
- 在 `config_set` 接口的返回中提示："该配置变更将在下次 Job 提交时生效，当前运行的 Job 不受影响"
- 如果后续需要支持，需说明 Pangen 是否支持 SIGHUP/SIGUSR1 重新加载配置

**所在位置**：2.3 节

---

### [P1] 6.3 config_validate 验证规则未定义

**问题描述**：
- `config_validate` 验证哪些字段？类型检查？范围检查？互斥字段检查？
- `job_submit` 调用时是否会**自动调用** `config_validate`？验证失败是否拒绝提交？
- 验证失败返回的 `ValidationResult` 结构是什么？

**改进建议**：
- 定义验证规则表（如：`simulation.iterations` 必须是正整数，范围 1-10000；`gpu.nodes` 必须 >= 1）
- **强烈建议** `job_submit` 内部自动调用 `config_validate`，验证失败则拒绝提交，返回详细错误信息
- `ValidationResult` 结构：
```json
{
  "valid": false,
  "errors": [
    {"field": "simulation.iterations", "message": "值 'abc' 不是有效整数"},
    {"field": "gpu.nodes", "message": "值 0 小于最小值 1"}
  ]
}
```

**所在位置**：2.3 节、4.3 节

---

## 七、Code Executor

### [P1] 7.1 最大执行时间 / CPU 时间阈值未定义

**问题描述**：
- 最大执行时间是 5 分钟？30 分钟？永不超时？
- CPU 时间限制是什么（防止无限计算但允许 I/O 等待）？
- 超时后是 kill 进程还是返回错误？

**改进建议**：
- 配置化：`~/.pangen-mcp/config.toml` 中 `code_executor.max_execution_time_seconds = 600`
- 超时行为：进程收到 SIGKILL，返回 `{"success": false, "error": "TIMEOUT", "elapsed_seconds": 600, "stdout": "...(截断)"}`
- 建议为 SSH 命令设置远程 timeout（如：`ssh gpu-01 "timeout 600 python script.py"`）

**所在位置**：2.6 节

---

### [P2] 7.2 GB 级输出截断机制缺失

**问题描述**：`execute_pangen_script` 返回 stdout/stderr，但：
- 如果脚本输出了 10GB 调试信息，传输到 MCP Server 会占用多少内存？
- 截断策略是什么？保留前 N 行？最后 N 行？还是报错？
- stderr 和 stdout 是合并返回还是分开返回？

**改进建议**：
- 明确截断策略：推荐保留**最后 1000 行**（大多数错误信息在结尾）
- 返回值增加字段：`{"truncated": true, "total_output_lines": 100000, "returned_lines": 1000, "stdout_tail": "...", "stderr_tail": "..."}`
- 大文件输出不通过 MCP 返回，改用 `job_logs` 接口分段读取
- stdout 和 stderr 分别返回（两个独立字段）

**所在位置**：2.6 节

---

### [P2] 7.3 脚本执行失败的错误码和错误信息格式未定义

**问题描述**：以下场景的返回格式都未定义：
- Python 语法错误（SyntaxError）
- Python 运行时错误（Exception）
- SSH 执行失败（连接超时、认证失败）
- Pangen 程序本身的错误返回码（exit_code != 0）
- 脚本执行成功但 Pangen 报告非零 exit_code

**改进建议**：定义统一的 `ScriptResult` 结构：
```json
{
  "success": false,
  "exit_code": 1,
  "error_type": "SyntaxError|RuntimeError|SSHError|PangenError|Timeout",
  "error_message": "Python syntax error: invalid syntax",
  "stdout": "...",
  "stderr": "...",
  "elapsed_seconds": 12.5,
  "gpu_host": "gpu-01"
}
```
不同 error_type 区分处理：SSH 错误重试一次，语法错误立即返回。

**所在位置**：2.6 节

---

## 八、Parameter Tuner

### [P2] 8.1 策略定义格式未确认

**问题描述**：调参策略的具体定义格式（JSON/YAML/TOML）尚未决定，这是 Tuner 引擎实现的基础。

**改进建议**：
- 推荐使用 **YAML**（人类可读，支持注释，便于维护调参策略）
- 定义策略文件的完整 schema（字段名、类型、是否必填）
- 给出一个完整的策略示例文件（如 `~/.pangen-mcp/strategies/refine_grid.yaml`）

**所在位置**：11 节（待确认事项）

---

### [P2] 8.2 "收敛慢"触发条件的量化标准缺失

**问题描述**：
- "收敛慢"完全模糊——由谁计算？基于什么指标？阈值是什么？
- 触发条件由 Result Analyzer 计算还是 Parameter Tuner 内部计算？
- "收敛慢"是看 CD_error 的绝对值还是变化率？

**改进建议**：
- 定义量化规则表：
  | 条件       | 计算方式                               | 阈值配置                       |
  | ---------- | -------------------------------------- | ------------------------------ |
  | 收敛慢     | 连续 3 次迭代的 `cd_error` 下降率 < 5% | `convergence_threshold = 0.05` |
  | 迭代未收敛 | 最后 2 次迭代的 `cd_error` 差异 < 0.01 | `residual_threshold = 0.01`    |
  | 局部误差大 | 任意区域的 CD_error > 1.5              | `local_error_threshold = 1.5`  |
- 量化计算在 Result Analyzer 完成分析后进行，metrics 作为 `tune_params` 的前置数据

**所在位置**：2.8 节

---

### [P2] 8.3 调参历史存储位置和查询接口缺失

**问题描述**：
- 调参历史记录在哪？`~/.pangen-mcp/tasks/{task_id}.json` 的 `history` 字段？还是独立存储？
- 如何跨 Task 查询历史（如：查询历史上所有 `refine_grid` 策略的实验结果）？
- 是否有独立的查询接口？

**改进建议**：
- 明确存储位置：推荐 `~/.pangen-mcp/tuning_history/` 目录下，每个调参事件一个 JSON 文件
- 增加 `get_tuning_history(filter: TuningHistoryFilter)` 接口，支持按 `task_id`、`strategy`、`time_range`、`outcome` 过滤
- 或复用 `task_result` 返回完整的调参历史

**所在位置**：2.8 节、4.5 节

---

## 九、Result Analyzer

### [P2] 9.1 分析脚本的管理与部署机制缺失

**问题描述**：
- 分析脚本在 GPU 上什么路径？（如：`/opt/pangen-mcp/analyzer/`）
- 谁负责部署脚本？MCP Server 启动时自动部署？手动运维？
- 分析脚本的版本如何管理？更新脚本后是否需要重启 MCP Server？

**改进建议**：
- 增加 `analyzer_deploy` 工具：MCP Server 启动时或手动调用时，将分析脚本同步到所有 GPU 节点
- 分析脚本路径配置化：`analyzer.script_path = "/opt/pangen-mcp/analyze.py"`
- 脚本版本号写在文件头（如注释中的 `version: 1.2.0`），MCP Server 启动时校验版本，不匹配则自动更新
- 增加 `analyzer_version` 接口查询当前 GPU 上的脚本版本

**所在位置**：2.7 节

---

### [P2] 9.2 compare_results 在 GB 级文件不传回本地的情况下如何实现？

**问题描述**：`compare_results(job_id_1, job_id_2)` 工具需要对比两个 Job 的结果文件。但文档明确说结果文件（GB 级二进制）"不传回本地"。如果两个文件都在 GPU 上，工具如何实现？

**改进建议**：
- `compare_results` 在 GPU 侧执行：先对两个 Job 分别执行分析脚本（如果尚未分析），提取 metrics 后对比 metrics（而非原始二进制文件）
- 明确 `compare_results` 的输入是两个 job_id，输出是对比 metrics（如：CD_error 均值差值、良率对比、收敛速度对比）
- 如果确实需要原始二进制对比（如热力图叠加），需要额外设计"按需临时小批量传输"机制

**所在位置**：2.7 节、4.4 节

---

## 十、工具定义完整性

### [P0] 10.1 config_patch 原子性实现方式未说明

**问题描述**：`config_patch` 声称"批量修改多个字段（原子性）"，但：
- 如果第 3 个字段的修改失败了，前 2 个字段的修改是否回滚？
- 是"全部成功或全部失败"还是"部分成功"？
- `config_patch` 的原子性是文件级别的还是字段级别的？

**改进建议**：
- 明确原子性范围：**推荐文件级原子性**（全部成功或全部回滚）
- 实现方式：读取当前配置 → 应用 patch → 写入临时文件 → rename（原子替换）
- 如果部分字段验证失败，应在写入前完成所有验证，然后一次性原子写入
- 返回值包含 `patched_fields: string[]` 和 `failed_fields: FieldError[]`

**所在位置**：4.3 节

---

### [P0] 10.2 job_submit 的 timeout 参数单位未定义

**问题描述**：`job_submit` 的参数 `timeout?: number` 没有说明单位：
- 是秒？分钟？还是毫秒？
- `timeout=60` 是 60 秒还是 60 分钟？两者相差 60 倍
- `timeout=0` 表示不设上限，需要明确说明

**改进建议**：
- 明确单位：**推荐秒**（与 Unix 惯例一致）
- 参数命名应包含单位暗示：`timeout_seconds: number`
- `timeout=0` 或 `timeout=null` 表示不设上限

**所在位置**：4.2 节

---

### [P2] 10.3 health_check 的具体检查项和返回格式未定义

**问题描述**：
- 检查项包括哪些？（SSH 连通性？GPU 节点状态？磁盘空间？License 服务器？）
- "磁盘空间"检查哪块磁盘？本地 MCP Server 的还是 GPU 的？
- 返回格式是什么？`HealthStatus` 的字段列表在哪里？
- 检查顺序是什么？（如：SSH 不通则其他检查无意义）

**改进建议**：定义完整 `HealthStatus` 结构：
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
检查顺序：SSH 连通性优先（如果 SSH 不通，其他检查标记为 skipped）。

**所在位置**：4.8 节

---

## 十一、迭代驱动逻辑

### [P2] 11.1 轮询间隔和退避策略缺失

**问题描述**：工作流中"Claude 轮询 job_status 直到 completed"没有说明：
- 轮询间隔是多少？1 秒？5 秒？30 秒？
- 是否有退避策略？（如：前 10 次每 5 秒，之后每 30 秒）
- 2 小时 Job 如果每 5 秒轮询一次，浪费多少 API 调用？

**改进建议**：
- Claude Code 端实现退避策略：
  - `status=pending`: 每 5 秒
  - `status=running`（前 2 分钟）: 每 10 秒
  - `status=running`（2 分钟后）: 每 30 秒
  - `status=completed/failed`: 停止轮询
- 或在 `job_status` 返回中包含 `next_poll_after_seconds` 字段，MCP Server 建议下次轮询时间
- 工作流示例（12.2 节）中明确轮询策略

**所在位置**：12.2 节

---

### [P2] 11.2 迭代上限内失败场景无学习机制

**问题描述**：如果前 9 次迭代都失败（参数调整策略无效），第 10 次成功：
- Parameter Tuner 是否有机会识别"前 9 次的策略都无效"？
- 调参历史中是否记录了"哪些策略已被证伪"？
- 下次类似 Task 是否能从这个失败历史中受益？

**改进建议**：
- Task 状态文件中的 history 字段增加 outcome 标记：
  ```json
  {"iteration": 1, "strategy": "refine_grid", "outcome": "failed", "reason": "cd_error未改善"}
  ```
- `tune_params` 增加可选参数 `failed_strategies[]`，避免重复尝试无效策略
- 通过 RAG 经验固化机制（10.1 节）实现跨 Task 学习

**所在位置**：2.9 节、12.2 节

---

### [P2] 11.3 task_result 最终报告格式未定义

**问题描述**：`task_result` 返回"最终配置、调参报告、最佳结果"，但：
- 报告的格式是什么？JSON？Markdown？HTML？
- 包含哪些字段？（配置 diff？每次迭代的 metrics 变化曲线？最终结果摘要？）
- 报告是否可导出为文件？

**改进建议**：定义 `TaskResult` 结构：
```json
{
  "task_id": "...",
  "goal": "优化参数达到 CD_error < 0.5",
  "outcome": "achieved|failed|partial",
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
      "outcome": "failed"
    },
    ...
  ],
  "total_iterations": 10,
  "elapsed_time_seconds": 7200,
  "recommendation": "建议将 refine_grid 策略与 increase_iterations 结合使用"
}
```
增加 `report_format` 参数：`"json"`（默认）或 `"markdown"`。

**所在位置**：4.1 节、12.2 节

---

## 十二、文档一致性问题

### [P2] 12.1 Section 6 编号重复（6.3 出现两次）

**问题描述**：
- 6.3 状态漂移 Limitations
- 6.4 Task 恢复机制
- **6.3 典型工作流**（编号重复）

第三个"典型工作流"的编号应为 6.5。

**改进建议**：
- 将"典型工作流"编号改为 **6.5**
- 重新整理 Section 6 结构：6.1 两层轮询架构 → 6.2 Job 状态查询 → 6.3 状态漂移 Limitations → 6.4 Task 恢复机制 → 6.5 典型工作流

**所在位置**：第 6 节

---

### [P2] 12.2 "通知机制"和"确认机制"的关系未明确

**问题描述**：
- 第 6 节标题是"通知机制"，但实际内容是"轮询模式"（无通知推送）
- 第 7 节是"确认机制"，两者是否存在交叉？
- "长时 Job 确认"属于通知机制还是确认机制？（文档在 6.3/7 节分别出现）

**改进建议**：
- 将第 6 节标题改为"状态查询与轮询机制"，明确说明"本服务不支持服务端主动推送通知，采用两层轮询模式"
- 在第 7 节确认机制中明确引用第 6 节，说明两者独立："长时 Job 确认"属于确认机制（主动用户交互），与轮询状态无关
- "30 分钟 vs 1 小时"的阈值不一致问题（见 3.2）应同步修正

**所在位置**：第 6 节、第 7 节

---

### [P2] 12.3 目录结构与正文模块编号无映射

**问题描述**：第 8 节目录结构中的模块（如 `task/mod.rs`、`job/manager.rs`）与正文中的 2.1-2.10 模块编号没有直接映射关系，读者难以快速定位。

**改进建议**：在第 8 节"目录结构"小节增加映射表：

| 正文模块编号 | 正文模块名         | 代码路径              |
| ------------ | ------------------ | --------------------- |
| 2.1          | MCP Protocol Layer | `src/mcp/mod.rs`      |
| 2.2          | Job Manager        | `src/job/manager.rs`  |
| 2.9          | Task Manager       | `src/task/manager.rs` |
| ...          | ...                | ...                   |

**所在位置**：第 8 节

---

## 十三、Phase 计划问题

### [P3] 13.1 Phase 1 缺少确认工具的明确归属

**问题描述**：
- Phase 1 标题包含"确认机制"，但 Phase 1 的工具列表（13 节）中缺少 `confirmation_approve` 和 `confirmation_reject`
- 如果确认机制在 Phase 1 实现，这两个工具应该在 Phase 1 工具列表中出现

**改进建议**：
- Phase 1 工具列表明确包含 `confirmation_approve` 和 `confirmation_reject`
- 或将确认机制拆为 Phase 1A（基础 pending 存储）和 Phase 1B（approve/reject 执行），在工具列表中分别标注

**所在位置**：13 节（Phase 计划）

---

### [P3] 13.2 Phase 6 代码执行的前置依赖链不完整

**问题描述**：Phase 6"代码执行"依赖"确认机制"（Phase 1），但 Phase 计划中没有标注依赖关系。

**改进建议**：在 Phase 计划中增加依赖图：
```
Phase 1 ──┬──> Phase 2 ──> Phase 3 ──> Phase 4 ──> Phase 5 ──> Phase 7
          └──> Phase 6（依赖 Phase 1 的 confirmation gate）
```
Phase 6 描述应为："代码执行（依赖 Phase 1 的 confirmation gate）"

**所在位置**：13 节（Phase 计划）

---

## 十四、边界情况与错误处理

### [P1] 14.1 config_validate 失败时 job_submit 的行为未定义

**问题描述**：用户可能绕过 `config_validate`，直接调用 `job_submit`。文档没有说明 `job_submit` 是否会在内部自动调用 `config_validate`。

**实际场景**：
1. 用户调用 `config_set(key="simulation.iterations", value=-100)`
2. 用户调用 `job_submit`
3. Pangen 启动后因参数无效而失败，浪费了 GPU 资源

**改进建议**：
- `job_submit` 内部自动调用 `config_validate`（即使用户没有手动调用）
- 验证失败时返回明确错误：`{"error": "CONFIG_VALIDATION_FAILED", "details": [...]}`
- 用户必须先修复配置才能提交 Job

**所在位置**：4.2 节、4.3 节、14.2 节

---

### [P1] 14.2 job_cancel 时 GPU 进程已退出的处理未定义

**问题描述**：Job 可能因 Pangen 内部错误已经退出，但状态文件仍显示 `running`。此时调用 `job_cancel`：
- 应该返回什么？（cancel 成功？因为进程已不存在所以 cancel 没有意义？）
- 是否应该同时更正状态文件？

**改进建议**：
- `job_cancel` 应检查 GPU 侧进程是否仍然存在：
  - 进程存在 → 发送 SIGTERM → 返回 `{"cancelled": true, "signal_sent": true, "gpu_pid": 12345}`
  - 进程不存在 → 更新状态文件为 `cancelled`（或 `failed`）→ 返回 `{"cancelled": false, "reason": "process_already_exited", "status_updated_to": "cancelled"}`
- 使用 `gpu_host + pid` 组合来验证进程（SSH 执行 `ps -p {pid}`）

**所在位置**：4.2 节、5.4 节

---

### [P2] 14.3 同一 Task 下多个 Job 能否并发未定义

**问题描述**：文档没有说明 Task 内的 Job 是否可以并发提交：
- 如果允许并发，多个 Job 同时读写同一 Task 状态文件如何处理？
- 如果不允许，Claude Code 如何知道"必须等上一个 Job 完成后才能提交下一个"？

**改进建议**：
- 明确策略：**推荐不允许并发**（简化状态管理，降低复杂度）
- Task 状态文件中增加 `current_job_id` 字段和 `job_submission_locked` 布尔字段
- `job_submit` 应验证 `task_id` 对应的 Task 状态，如果 `current_job_id` 对应的 Job 尚未完成则拒绝提交：`{"error": "TASK_JOB_IN_PROGRESS", "current_job_id": "xxx"}`
- 或者实现乐观锁：Job 提交时检查 `history` 长度，提交后立即 +1，如果两次提交检测到不一致则后者失败

**所在位置**：2.9 节、4.1 节、4.2 节

---

## 附录：P0 问题汇总（上线前必须决策）

| 编号 | 问题                                                  | 影响                             |
| ---- | ----------------------------------------------------- | -------------------------------- |
| 2.1  | progress 字段类型定义冲突（字符串 vs 浮点数）         | 状态查询结果不一致               |
| 3.1  | Confirmation Gate 无法拦截 Claude Code 生成的有害脚本 | 安全隐患                         |
| 4.1  | SSH 认证方式未明确                                    | 核心基础设施缺失，无法连接 GPU   |
| 10.1 | config_patch 原子性实现未说明                         | 配置修改可能部分失败，数据不一致 |
| 10.2 | job_submit timeout 参数单位未定义                     | 超时行为不可预测                 |
| 14.1 | config_validate 失败时 job_submit 的行为未定义        | 可能提交无效配置到 GPU           |

---

*本文件由设计文档自动分析生成，建议配合源文档 MCP-PANGEN-SERVER.md 一起阅读。*