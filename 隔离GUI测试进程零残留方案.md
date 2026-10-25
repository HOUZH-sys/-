# macOS 隔离 GUI 测试进程零残留方案

> 适用场景：Godot、Electron、Qt 或其他通过 macOS LaunchServices 启动的桌面 GUI，
> 需要在独立测试账户中反复运行自动化 GUI 测试，并保证任务结束后不遗留应用进程。

## 1. 方案摘要

本方案解决的核心问题不是“测试结束后偶尔执行一次 kill”，而是建立一条
**可证明、可审计、不会越界终止无关进程**的进程生命周期闭环：

```text
提交任务
  -> Bridge 启动前清理
  -> 测试 Runner 启动前清理
  -> 启动 GUI
  -> 正常完成 / 异常退出 / 外层超时
  -> Runner EXIT trap 清理
  -> Bridge 任务后再次清理
  -> 重新扫描并验证匹配 PID 数 = 0
  -> 写入清理证据
```

最终验收必须同时满足：

1. 真实 GUI 测试通过。
2. 控制台用户未被切换，`console_unchanged=true`。
3. 启动前后和退出后均有清理记录。
4. 每条记录的剩余 PID 集合为空。
5. 独立 `ps` 检查也看不到目标应用进程。

## 2. 问题定义

### 2.1 表面症状

- 每次隔离 GUI 测试后，测试账户中会多留下一个游戏窗口。
- 长时间运行后累计出现多个 Godot 进程。
- CPU 持续占用，测试账户越来越卡。
- 进程可能由 LaunchServices 拉起，和测试脚本并非简单的父子关系。
- 只在成功路径清理，无法覆盖失败、异常或超时。

### 2.2 真正的风险

这不是单纯的资源浪费，而是测试系统完整性问题：

- 上一个任务可能污染下一个任务的进程环境。
- 新截图可能来自旧窗口，而不是当前候选版本。
- 并行任务可能同时操作多个应用实例。
- 宽泛的 `pkill` 修复可能误杀开发机上的其他 Godot 项目。
- 如果测试桥以错误账户运行，跨账户清理可能影响正常用户桌面。
- 清理失败却仍返回 PASS，会让错误进入发布链路。

### 2.3 故障模型

GUI 测试可能从以下任意路径退出：

1. 正常完成。
2. 测试断言失败。
3. 脚本参数或输入错误。
4. App 启动后未写入状态文件。
5. App 崩溃或窗口消失。
6. CuaDriver 权限或交互失败。
7. 外层任务超时。
8. Bridge 清理证据文件不可写。
9. 清理命令发送成功，但进程没有退出。
10. 进程扫描本身失败。

方案必须覆盖所有路径，不能把清理挂在“成功出口”上。

## 3. 目标与非目标

### 3.1 目标

- 只清理当前测试账户中的目标 App。
- 只清理位于允许构建根目录内、由目标 App 包启动的精确可执行文件。
- 正常、失败、异常和超时路径统一进入清理闭环。
- 先温和终止，超时后有限升级为强制终止。
- 每次清理都重新扫描并证明剩余 PID 为零。
- 清理失败必须让测试失败。
- 记录机器可读证据，支持复盘和自动验收。
- 两层调用可以重复执行，但行为保持幂等。

### 3.2 非目标

- 不清理整个测试账户的所有进程。
- 不按窗口标题或模糊名称匹配。
- 不终止其他 Godot 项目或正常用户会话。
- 不依赖测试脚本和 GUI 进程的父子关系。
- 不通过重启整台机器掩盖清理缺陷。
- 不把前台桌面自动化当作隔离后台测试的替代方案。

## 4. 六条设计原则

### 4.1 精确身份，而不是名称匹配

一个可安全终止的目标至少要同时满足：

- App 路径位于 `--allowed-root` 内。
- `CFBundleIdentifier` 等于预期值。
- `CFBundleExecutable` 来自 App 自己的 `Info.plist`。
- 可执行文件真实路径仍位于该 App 的 `Contents/MacOS/` 内。
- 候选进程 UID 等于当前测试账户 UID。
- 候选进程的 `comm` 等于目标可执行文件真实路径。

### 4.2 发送信号前再次核验

PID 会复用，扫描结果也可能过时。发送 `SIGTERM` 或 `SIGKILL` 前必须：

1. 按 PID 重新读取 UID 和 `comm`。
2. 再次确认 UID 与可执行路径。
3. 只有全部匹配才发送信号。

### 4.3 所有退出路径统一清理

清理不能只放在测试函数末尾。至少需要：

- Bridge 层：任务开始前和任务结束后。
- Runner 层：启动前和 `EXIT` trap。
- 超时包装层：超时后终止测试进程组。

多层冗余不是重复劳动，而是防止某一层未执行。

### 4.4 有界升级

永不无限等待：

```text
扫描目标 PID
  -> 发送 SIGTERM
  -> 最多等待 3 秒
  -> 仍有目标 PID
  -> 重新核验后发送 SIGKILL
  -> 最多等待 3 秒
  -> 重新扫描
```

等待时间应有上限，且应写入证据。

### 4.5 失败关闭

以下任一情况都不能返回 PASS：

- 无法读取 App 元数据。
- 可执行文件真实路径越出 App 包。
- 进程扫描失败。
- 证据文件无法写入。
- `SIGKILL` 后仍有匹配 PID。
- 测试结果存在，但清理验证未完成。

### 4.6 可审计

每个关键节点写入结构化记录。记录应能回答：

- 何时清理？
- 清理哪个 App？
- 目标可执行文件的真实路径是什么？
- 使用哪个 UID？
- 清理前有哪些 PID？
- 哪些 PID 收到 TERM？
- 哪些 PID 收到 KILL？
- 最终还剩哪些 PID？
- 最终是否证明为零？

## 5. 推荐架构

### 5.1 三层职责

| 层 | 责任 | 不能承担的职责 |
|---|---|---|
| 登录 LaunchAgent / Bridge | 领取任务、启动前清理、任务后清理、生成最终结果 | 不按模糊名称扫描，不跨账户操作 |
| 单次 GUI Runner | 校验构建、启动 App、执行交互、`EXIT` trap 清理 | 不修改源码，不跳过真实截图和输入 |
| Exact Reaper | 精确枚举、双重身份校验、TERM/KILL 升级、证据写入 | 不决定测试业务是否通过 |

### 5.2 数据流

```text
job.json
  |
  v
Bridge
  |-- reap(label=bridge-before-launch)
  |-- cua-playtest-cards.sh
  |     |-- reap(label=cards-before-launch)
  |     |-- open App
  |     |-- real screenshots + mouse/keyboard
  |     `-- EXIT trap -> reap(label=cards-exit)
  `-- reap(label=bridge-post-job)
        |
        v
process-cleanup.jsonl  +  result.json
```

### 5.3 为什么需要两层清理

只依赖 Runner 的问题：

- Runner 可能在加载脚本前失败。
- 外层超时可能直接终止 Runner。
- Runner 进程可能崩溃，`EXIT` trap 来不及执行。

只依赖 Bridge 的问题：

- 开发者在隔离环境中单独运行 Runner 时仍可能泄漏。
- Bridge 任务执行期间可能堆积多个实例。

因此两层都需要调用同一个 Exact Reaper，保证规则一致。

## 6. Exact Reaper 的匹配模型

### 6.1 输入

```bash
reap-looped-vows-processes.sh reap \
  --app /allowed/builds/<id>/LoopedVows.app \
  --allowed-root /allowed/builds \
  --evidence <result-dir>/process-cleanup.jsonl \
  --label cards-exit
```

### 6.2 安全校验顺序

```text
1. 解析并 realpath App
2. 验证 App 位于 allowed-root 内
3. 从 Info.plist 读取 Bundle ID 与 CFBundleExecutable
4. 验证可执行文件名只包含安全字符
5. realpath 可执行文件
6. 验证可执行文件没有通过符号链接越出 App
7. 记录当前账户 UID
8. 扫描 ps，仅保留 UID 和精确 comm 都匹配的 PID
9. 对每个 PID 再次核验
10. 发送 TERM
11. 有界等待并升级 KILL
12. 最终重新扫描并证明 PID 集合为空
```

### 6.3 匹配伪代码

```text
function is_target(pid):
    current_uid, current_command = read(pid)
    return current_uid == current_test_uid
       and realpath(current_command) == realpath(expected_executable)
```

不是下面这种写法：

```text
kill every process whose name contains "Godot"
```

也不是：

```text
kill every process whose path contains the build directory name
```

后两种写法无法证明不会误杀。

## 7. 清理状态机

### 7.1 正常路径

```text
candidates = scan_exact_pids()
send(SIGTERM, revalidate(candidates))
wait_until_zero(3s)
remaining = scan_exact_pids()
if remaining is empty:
    write_evidence(verified_zero=true)
    exit 0
```

### 7.2 需要升级

```text
send(SIGTERM, revalidate(candidates))
if wait_until_zero(3s) fails:
    send(SIGKILL, revalidate(remaining))
    wait_until_zero(3s)
```

### 7.3 失败路径

```text
if scan fails:
    write_evidence(verified_zero=false)
    exit 1

if remaining_pids is not empty after SIGKILL:
    write_evidence(verified_zero=false)
    exit 1

if evidence cannot be written:
    exit 1
```

调用方不得忽略非零退出码。

## 8. 证据格式

推荐使用 JSON Lines，每次清理追加一行：

```json
{
  "at": "2026-10-25T01:01:43Z",
  "label": "cards-exit",
  "app": "/allowed/builds/<id>/LoopedVows.app",
  "executable": "/allowed/builds/<id>/LoopedVows.app/Contents/MacOS/LoopedVows",
  "bundle_id": "com.codexautomation.loopedvows",
  "account_uid": 502,
  "match_scope": "exact-bundled-executable-and-current-account",
  "term_timeout_seconds": 3,
  "kill_timeout_seconds": 3,
  "before_pids": [27376],
  "sigterm_pids": [27376],
  "sigkill_pids": [],
  "remaining_pids": [],
  "inspection_error": "",
  "verified_zero": true
}
```

建议标签：

- `bridge-before-launch`
- `cards-before-launch`
- `cards-exit`
- `bridge-post-job`

一次完整测试至少应出现这四个节点。

## 9. 必须覆盖的故障矩阵

| 场景 | 预期行为 | 验收证据 |
|---|---|---|
| 测试正常完成 | TERM 退出，最终 PID=0 | `cards-exit` 与 `bridge-post-job` |
| 测试断言失败 | `EXIT` trap 仍执行清理 | `result.status=failed` 且清理证据存在 |
| 测试脚本异常 | Bridge 仍执行任务后清理 | `bridge-post-job` |
| 外层超时 | 先结束测试进程组，再精确清理 App | 超时错误与零 PID 记录 |
| App 不响应 TERM | 有界升级 KILL | `sigkill_pids` 非空，最终 PID=0 |
| 清理后仍存活 | 测试必须 FAIL | `remaining_pids` 非空 |
| 进程扫描失败 | 测试必须 FAIL | `inspection_error` 非空 |
| 证据文件不可写 | 测试必须 FAIL | 明确初始化错误 |
| App 路径越界 | 拒绝清理 | 非零退出，不发送信号 |
| Bundle ID 不匹配 | 拒绝清理 | 非零退出，不发送信号 |
| 无关 Godot 进程存在 | 保持存活 | 单测中的 unrelated PID 仍存在 |
| 同一 App 多实例 | 精确清理所有匹配 PID | `before_pids` 包含多个 PID |

## 10. 测试策略

### 10.1 单元测试

至少覆盖：

- 普通进程收到 TERM 后退出。
- 忽略 TERM 的进程升级为 KILL。
- 相同名称但不在目标 App 中的进程不受影响。
- 目标 App 位于允许根目录外时拒绝执行。
- Bundle ID 错误时拒绝执行。
- 证据路径不可写时返回失败。
- 最终仍有目标 PID 时返回失败。

### 10.2 集成测试

使用真实隔离 GUI：

```bash
/Users/Shared/godot-playtest/runtime/submit-card-playtest.sh --demo=menu
```

验收步骤：

1. 读取命令输出的结果目录。
2. 检查 `result.json` 的 `status` 与 `console_unchanged`。
3. 检查 `process-cleanup.jsonl` 的四个标签。
4. 对每条记录检查：
   - `match_scope=exact-bundled-executable-and-current-account`
   - `remaining_pids=[]`
   - `verified_zero=true`
   - `inspection_error=""`
5. 独立执行只读 `ps` 扫描。
6. 确认匹配进程数为零。

### 10.3 发布前命令

```bash
bash -n \
  automation/godot/bridge.sh \
  automation/godot/cua-playtest-cards.sh \
  automation/godot/prepare-shared.sh \
  automation/godot/reap-looped-vows-processes.sh \
  automation/godot/reap-looped-vows-processes.test.sh

git diff --check

./automation/godot/reap-looped-vows-processes.test.sh

./godot-court/check.sh
```

## 11. 运行与排障手册

### 11.1 出现问题时的检查顺序

1. 确认 Bridge 使用哪个账户和哪个共享运行时。
2. 比较仓库脚本与共享运行时哈希。
3. 检查最近一次 `process-cleanup.jsonl`。
4. 检查是否有标签缺失。
5. 检查 `inspection_error`、`remaining_pids`、`verified_zero`。
6. 用只读 `ps` 独立确认当前进程数。
7. 若共享运行时与仓库不一致，重新发布并精确重启 Bridge。
8. 只运行一个真实 GUI 测试，避免掩盖并发问题。

### 11.2 只读诊断命令

```bash
ps -axo pid=,uid=,command= |
  rg '/Users/Shared/godot-playtest/builds/[^ ]*/LoopedVows\.app/Contents/MacOS/LoopedVows'
```

该命令只诊断，不会终止任何进程。

### 11.3 重启 Bridge

必须使用明确的 LaunchAgent 标签，不使用模糊进程匹配：

```bash
/bin/launchctl kickstart -k gui/502/com.codex.godot-playtest
```

执行前确认 UID 与标签，不要把 `502` 当作所有机器的固定值。

## 12. 常见错误

### 错误 1：只在成功路径清理

结果：失败测试持续泄漏。

正确做法：使用 `EXIT` trap，并在外层任务结束后再清理一次。

### 错误 2：按进程名清理

```bash
pkill -f Godot
```

风险：误杀其他项目、用户正在编辑的编辑器或无关 GUI。

正确做法：UID + App 根目录 + Bundle ID + 精确可执行文件 realpath。

### 错误 3：扫到 PID 就直接 kill

风险：PID 在扫描和终止之间被复用。

正确做法：发送信号前重新读取 UID 与 `comm`。

### 错误 4：只检查 kill 命令返回码

`kill` 成功只代表信号已发送，不代表进程已退出。

正确做法：有界等待后重新扫描，并证明 PID 集合为空。

### 错误 5：清理失败仍保留 PASS

风险：进程泄漏被测试成功结果掩盖。

正确做法：清理失败强制测试失败。

### 错误 6：依赖父进程关系

LaunchServices 可能让 App 被系统服务拉起，测试脚本不是其直接父进程。

正确做法：按应用身份扫描，不依赖进程树。

## 13. 验收清单

发布前逐项勾选：

- [ ] App 只允许从指定构建根目录启动。
- [ ] Bundle ID 和 `CFBundleExecutable` 来自 App 自身元数据。
- [ ] 可执行文件真实路径经过越界检查。
- [ ] 只匹配当前测试 UID。
- [ ] 发送信号前逐 PID 二次校验。
- [ ] 启动前有清理。
- [ ] Runner 有 `EXIT` trap。
- [ ] 外层有任务后清理。
- [ ] 超时路径会结束测试进程组。
- [ ] TERM 与 KILL 都有上限。
- [ ] 清理后重新扫描并验证零 PID。
- [ ] 扫描或证据失败会让测试失败。
- [ ] 单元测试覆盖无关进程不受影响。
- [ ] 真实 GUI 测试包含四个清理节点。
- [ ] 独立 `ps` 检查为零。
- [ ] 共享运行时与仓库脚本逐字节一致。
- [ ] 用户已有未提交改动未被覆盖。

## 14. 本项目的实际落地记录

### 14.1 修复提交

```text
b85540fa88ffb4f3466f09a9ca0b6ae38a23b7fa
codex: reap Godot GUI processes exactly
```

涉及文件：

- `automation/godot/reap-looped-vows-processes.sh`
- `automation/godot/reap-looped-vows-processes.test.sh`
- `automation/godot/bridge.sh`
- `automation/godot/cua-playtest-cards.sh`
- `automation/godot/prepare-shared.sh`

### 14.2 真实 GUI 证据

结果目录：

```text
/Users/Shared/godot-playtest/results/cards-20261025-100110-25110
```

关键结果：

- `status=passed`
- `console_unchanged=true`
- `bridge-before-launch`：零 PID
- `cards-before-launch`：零 PID
- `cards-exit`：PID `27376` 收到 TERM 并退出
- `bridge-post-job`：零 PID
- 独立 `ps` 检查：零匹配进程

### 14.3 共享运行时校验

```text
bridge.sh                  a10e70796f936b72ca058ed5cf13ce9b9c2d0c3e4f170f14f5297b0e81d57df1
cua-playtest-cards.sh      1845ff7b9605451dd4ef62797fe8d1edbceee5cd6d4cf2c75d11c0e5824ecb03
reap-looped-vows-processes  b1c83cd95f5995c87d37f3517db8f266a35930ee302c14783b658cbc82f448da
```

仓库与共享运行时逐字节一致。

### 14.4 最终验证

- `bash -n`：PASS
- `git diff --check`：PASS
- Exact Reaper 单元测试：PASS
- 完整 `godot-court/check.sh`：PASS
- Tester 独立 GUI：PASS
- 最终匹配 PID 数：0
- Host guard：已停止，无残留
- 原有用户未提交改动：保持不变

## 15. 可迁移到其他桌面应用

替换以下参数即可复用主体设计：

- `allowed-root`
- App 路径
- 期望 Bundle ID
- `CFBundleExecutable`
- 测试账户 UID
- 结果证据目录

若目标应用会派生独立 helper：

1. 不要直接放宽为模糊名称匹配。
2. 单独定义 helper 身份规则。
3. 同样校验 UID、签名、Bundle 归属或真实路径。
4. 为 helper 增加单独测试证据。
5. 明确 helper 是否允许在应用主进程退出后继续存在。

## 16. 教学讨论题

1. 为什么“kill 命令返回 0”不能证明进程已退出？
2. 为什么启动前也要清理，而不能只处理结束路径？
3. 如果 Bridge 超时直接杀掉 Runner，哪一层还能完成清理？
4. 为什么需要两层调用同一个 Reaper？
5. PID 复用会如何破坏只扫描一次的实现？
6. 如果清理证据写入失败，为什么必须阻止测试 PASS？
7. 如何在不使用 `pkill` 的前提下清理同一 App 的多个实例？
8. 如何验证新规则没有误杀同机上另一个 Godot 项目？

## 17. 结论

可靠的 GUI 测试清理不是一条 kill 命令，而是一个完整协议：

```text
精确身份
  + 所有退出路径
  + 有界信号升级
  + 零 PID 复验
  + 失败关闭
  + 结构化证据
  + 真实 GUI 验收
```

只有这些条件同时成立，才能把“测试完成”与“测试环境已恢复干净”视为同一件事。
