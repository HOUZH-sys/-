# GUI 自动化验收完整方案

> 定位：Looped Vows 当前 macOS GUI 自动化验收体系的教学、实施和运维主文档。
>
> 核心目标：在隔离账户中用真实窗口、真实截图和原生输入验证已提交的 Godot
> 构建，并证明任务结束后没有残留游戏进程。

## 目录

1. [一页看懂](#1-一页看懂)
2. [适用场景与非目标](#2-适用场景与非目标)
3. [六条设计原则](#3-六条设计原则)
4. [角色与职责](#4-角色与职责)
5. [目录与运行边界](#5-目录与运行边界)
6. [完整数据流](#6-完整数据流)
7. [阶段说明](#7-阶段说明)
8. [原生输入决策树](#8-原生输入决策树)
9. [截图与视觉证据](#9-截图与视觉证据)
10. [进程零残留协议](#10-进程零残留协议)
11. [封印与独立 Validator](#11-封印与独立-validator)
12. [从安装到验收的操作手册](#12-从安装到验收的操作手册)
13. [故障矩阵](#13-故障矩阵)
14. [当前保留与已淘汰的路线](#14-当前保留与已淘汰的路线)
15. [指标与质量目标](#15-指标与质量目标)
16. [迁移到其他 GUI 项目](#16-迁移到其他-gui-项目)
17. [教学路径](#17-教学路径)
18. [Reviewer 清单](#18-reviewer-清单)
19. [相关文档](#19-相关文档)

## 1. 一页看懂

完整链路不是“启动游戏截一张图”，而是一条可证明的验收流水线：

```text
干净的 Developer 提交
  -> 构建
  -> 发布不可混淆的 App/PCK 指纹
  -> 主账户提交带 task/SHA/scenario 的任务
  -> godot-test Bridge 领取任务
  -> 启动前清零进程
  -> 启动真实 Godot 窗口
  -> 原生 AX / 精确窗口键鼠输入
  -> 应用渲染截图 + 原生窗口截图
  -> Runner 退出时再次清零进程
  -> Bridge 任务后再清零并验证 PID=0
  -> 规范化、封印证据
  -> 独立 Validator 交叉验证
  -> 能力票和状态机接受 PASS
```

一句话原则：

> **原生语义优先，精确窗口兜底，证据必须闭环，清理失败即验收失败。**

`result.json` 中的 `status=passed` 只是 Runner 自报状态，不等于最终验收。
只有独立 Validator 与上层状态机也通过，才能把结果视为可接受。

## 2. 适用场景与非目标

### 2.1 适用场景

- Godot 原生 macOS App 的真实 GUI 冒烟测试。
- 需要点击、拖拽、键盘操作、窗口截图和状态断言的交互验收。
- 多个 Agent 或自动化任务复用同一台 Mac，但日常桌面不能被测试窗口占用。
- 需要可审计地证明“测试结束后应用进程为零”。
- 需要把某个提交、构建、测试场景和验收证据绑定在一起。

### 2.2 非目标

- 这不是内核级安全沙箱。隔离边界是独立 macOS 账户、精确进程身份和共享目录。
- 证据封印是本地防篡改检查，不是密码学签名。
- 不证明同 UID 的恶意进程无法伪造证据；它用于发现意外漂移和后置修改。
- 不替代玩法 Headless 测试。Headless 测试负责快速逻辑回归，GUI 测试负责真实呈现和输入。
- 不通过 CUA 前台控制日常桌面账户。

## 3. 六条设计原则

1. **原生优先**

   有可操作 AX 元素时使用 AXPress；有焦点控制时使用精确窗口键盘；
   只有自绘控件没有可操作 AX 元素时，才使用实时 Godot `Control` 矩形对应的窗口点击。

2. **目标必精确**

   每次截图、输入和断言都必须绑定已确认的 `pid + window_id`。禁止按应用名、
   窗口标题模糊操作，也禁止向整个桌面注入输入。

3. **账户与环境隔离**

   主账户负责构建、发布、提交和最终验收；`godot-test` 账户负责 Bridge、
   CuaDriver、真实 GUI 运行和结果生成。

4. **失败关闭**

   构建不干净、指纹不符、窗口无法确认、输入回执不完整、截图缺失、
   进程未清零、证据未封印或 completion 未完成，都不能得到 PASS。

5. **一条输入路径只保留一个责任人**

   AX、精确窗口键盘、实时 Control 矩形点击和精确窗口拖拽各有明确职责。
   同一动作不再保留 desktop、double-click、reference-layout 等重复兜底。

6. **验收证据不可只看代码**

   视觉质量必须来自真实窗口截图和真实输入结果。代码静态检查只能证明契约存在，
   不能替代 GUI 证据。

## 4. 角色与职责

| 角色 | 运行账户 | 职责 | 不应承担 |
|---|---|---|---|
| Supervisor | 主账户 | 选任务、提交候选、检查结果、调用独立验收 | 代替 Tester 操作前台游戏 |
| Developer | 独立克隆 | 实现和提交候选代码 | 自行宣布 GUI PASS |
| Build/Publish | 主账户 | 从干净提交构建并发布 App/PCK 指纹 | 发布脏工作区或错 SHA |
| Submit Client | 主账户 | 写 job、等待 completion 和结果 | 自行修改结果状态 |
| Bridge | `godot-test` | 串行领取任务、启动 Runner、清理和封装结果 | 接受前台任意桌面任务 |
| Runner | `godot-test` | 启动 App、执行场景、输入、截图和状态断言 | 修改候选源码 |
| CuaDriver | `godot-test` | AX、原生窗口截图和精确窗口输入 | 对主账户桌面注入输入 |
| Sealer | `godot-test` | 哈希、权限和只读封印 | 判断视觉质量 |
| Validator | 主账户 | 独立交叉验证证据、构建、进程和任务 | 相信 Runner 自报 PASS |
| Capability Gate | 主账户 | 绑定 Product Design/2.5D 等能力证据 | 用通用理由代替具体证据 |

## 5. 目录与运行边界

共享根目录默认是：

```text
/Users/Shared/godot-playtest/
  builds/       # 发布后的 App、manifest 和 latest 指针
  queue/        # 待领取 job
  processed/    # claim、completed marker 和已领取 job
  results/      # 每次运行的结果与证据
  runtime/      # Bridge、Runner、CuaDriver 和工具脚本
```

仓库内核心文件：

| 文件 | 作用 |
|---|---|
| `automation/godot/prepare-shared.sh` | 原子同步共享 runtime |
| `automation/godot/install-bridge-login.command` | 在 `godot-test` 安装 LaunchAgent 和权限 |
| `automation/godot/build.sh` | 从干净提交导出、签名并生成 manifest |
| `automation/godot/publish-build.sh` | 把已验证构建发布到 canonical build 目录 |
| `automation/godot/submit-card-playtest.sh` | 主账户提交 job 并等待 terminal marker |
| `automation/godot/bridge.sh` | `godot-test` 后台任务循环与最终收口 |
| `automation/godot/cua-playtest-cards.sh` | 当前卡牌场景的 GUI Runner |
| `automation/godot/reap-looped-vows-processes.sh` | 精确 UID/path 进程回收器 |
| `automation/godot/seal-card-playtest-result.sh` | 结果文件哈希和只读封印 |
| `automation/godot/validate-card-playtest-result.sh` | 独立验收 Validator |
| `automation/godot/GUI-ACCEPTANCE-CONTRACT.md` | 机器实现对应的验收契约 |
| `automation/godot/gui-native-path.test.sh` | 原生输入路径静态防回归门禁 |

## 6. 完整数据流

```text
Developer commit
  |
  v
build.sh
  |-- source_clean
  |-- source_sha
  |-- binary_sha256
  `-- pck_sha256
        |
        v
publish-build.sh
  |-- builds/<build_id>/LoopedVows.app
  |-- builds/<build_id>/manifest.json
  `-- builds/card-latest.json
        |
        v
submit-card-playtest.sh
  |-- validates candidate/build/pointer
  `-- queue/<job_id>.json
        |
        v
bridge.sh (godot-test)
  |-- exclusive bridge lock
  |-- claim job
  |-- bridge-before-launch reap
  |-- cua-playtest-cards.sh
  |     |-- cards-before-launch reap
  |     |-- launch and bind pid/window
  |     |-- native input + paired screenshots
  |     `-- cards-exit reap
  |-- bridge-post-job reap
  |-- normalize result
  |-- seal passed evidence
  `-- processed/<job_id>.completed.json
        |
        v
validate-card-playtest-result.sh
  `-- PASS / FAIL
```

## 7. 阶段说明

### 7.1 构建：先把候选变成可信 artifact

`build.sh` 只接受干净的游戏子树，并生成：

- 完整 `source_sha`。
- 7 位 source prefix。
- 唯一 build id。
- macOS App bundle。
- executable 与 PCK 的 SHA-256。
- `source_clean=true` 的 manifest。

构建阶段解决的是“测的是哪个代码”问题。没有完整 SHA 或工作区不干净，后续
截图即使漂亮也不能作为候选验收证据。

### 7.2 发布：只允许 canonical 路径

`publish-build.sh` 验证本仓 `HEAD`、manifest、App、binary 和 PCK，再发布到：

```text
/Users/Shared/godot-playtest/builds/<build_id>/
```

job 中的 App 和 manifest 必须位于对应 build id 目录，不能是指向其他位置的
符号链接。发布后更新 `latest.json` 和 `card-latest.json` 指针。

### 7.3 提交：把任务定义写进不可混淆的 job

提交客户端要求：

- 安全 `task_id`。
- 完整 40 位 `candidate_sha`。
- 明确 `demo` 场景。
- 受限超时。
- App/PCK 指纹与发布 manifest 一致。

job 的 `result_dir` 只能是由 `job_id` 推导出的共享 results 子目录。

### 7.4 Bridge：单消费者、双层清理、最终封装

Bridge 必须由 `godot-test` 运行，并使用独占锁串行处理 job。它：

1. 只领取 `mode=isolated-background`。
2. 验证 job id、路径、manifest、source SHA 和 binary/PCK 指纹。
3. 执行 `bridge-before-launch`。
4. 以独立进程组运行 Runner，并有界处理超时。
5. 执行 `bridge-post-job`。
6. 规范化 job/task/candidate/identity 字段。
7. 只对 Runner 自报 PASS 的结果执行封印。
8. 写 completion marker，主账户才可结束等待。

### 7.5 Runner：只负责真实执行和证据采集

Runner 必须：

- 确认运行用户是 `godot-test`。
- 校验 canonical manifest、App、binary 和 PCK。
- 启动前再次清理目标 App 进程。
- 找到唯一目标 PID。
- 通过窗口列表、AX 树和至少两次尺寸一致的原生截图确认 `window_id`。
- 只向该窗口发送原位输入。
- 每个视觉节点同时保存应用渲染和原生窗口截图。
- 同时用 `state.json`、AX 和动作回执验证状态变化。
- Runner 的 EXIT trap 尽力执行 `cards-exit`；即使 trap 来不及运行，
  Bridge 的 `bridge-post-job` 仍必须清理并证明零进程。

## 8. 原生输入决策树

任何交互都按以下顺序选择，不能为了“更容易写脚本”跳到桌面输入：

```text
有可操作 AX 元素？
  yes -> AXPress（background）
  no
   |
   v
是否可用精确窗口焦点键盘完成？
  yes -> exact-window key
  no
   |
   v
是否是可读取实时矩形的 Godot Control/自绘控件？
  yes -> exact-window single click
  no
   |
   v
是否需要拖拽或保持手势？
  yes -> exact-window drag
  no
   |
   v
目标窗口明确拒绝 background，但返回 exact-window foreground 建议？
  yes -> 同一 pid/window 的 foreground HID
  no  -> FAIL，不使用桌面兜底
```

### 8.1 输入路线与证据字段

| 路线 | 适用对象 | `geometry_source` | 必要回执 |
|---|---|---|---|
| AXPress | 按钮、筛选项、卡牌行、关闭、结束回合 | `accessibility_control` | background + accessibility + AXPress |
| 精确窗口键盘 | 焦点顺序、Enter、Space、Shift+Tab | `focused_control` | 绑定目标 PID/窗口 |
| 精确窗口单击 | 只有实时 Control 矩形、没有可操作 AX 的自绘控件 | `godot_control_rect` | exact-window click |
| 精确窗口拖拽 | 卡牌到槽位、棋盘拖放 | `godot_control_rect` | 目标 PID/window + HID drag 回执 |
| 精确窗口前台升级 | background 被目标窗口明确拒绝时 | 保持真实动作来源 | 同一 `pid + window_id`，回执必须命名 PID |

click 和 key 必须分别校验回执。没有 PID 绑定的 desktop key、全局 click、
`CGEventPostToPid`、Terminal fallback 和 `viewport.push_input` 都不能进入验收账本。

### 8.2 为什么像素点击仍然存在

“原生优先”不等于“完全禁用像素”。Godot 自绘卡牌、棋盘和奖励控件不一定暴露
可操作 AX 元素。此时最可靠的原生路径是：

1. 从实时 Godot `Control` 节点读取矩形。
2. 把矩形换算到已确认窗口的坐标。
3. 对该 `pid + window_id` 执行一次单击或拖拽。
4. 用状态变化、截图和输入回执共同确认动作成功。

禁止的是脱离实时控件的硬编码 `reference_layout` 坐标。

## 9. 截图与视觉证据

### 9.1 每个检查点必须成对

```text
raw/<checkpoint>.png           # 应用内部渲染截图
<checkpoint>-native.png       # 原生窗口截图
```

两边文件名集合必须一一对应。应用截图和原生截图数量不等、哈希重复或尺寸无法独立
解码，都不能通过。

### 9.2 证据文件

| 文件 | 作用 |
|---|---|
| `state.json` | 运行过程状态，用于精确断言 |
| `actions.jsonl` | 时间顺序的输入、截图和断言账本 |
| `native-captures.jsonl` | 原生截图采集记录 |
| `raw/` | 应用渲染截图 |
| `raw-index.json` | 渲染文件名、hash、bytes、宽高索引 |
| `*-native.png` | 原生窗口截图 |
| `contact-sheet.jpg` | 供每回合一次视觉审核的汇总图 |
| `process-cleanup.jsonl` | 四阶段进程回收证据 |
| `vision.json` | 视觉结果摘要 |
| `result.json` | Runner 结果与完整动作/截图元数据 |
| `evidence-manifest.json` | 封印时生成的全文件哈希清单 |

### 9.3 低成本审核规则

GUI 审核每回合最多读取一张 `contact-sheet.jpg`，不批量打开 `raw/`。原始 PNG
是桥接证据，不是日常沟通材料。只有用户明确要求检查某个原图时才单独读取。

## 10. 进程零残留协议

每次成功验收必须出现并按顺序证明四个阶段：

1. `bridge-before-launch`
2. `cards-before-launch`
3. `cards-exit`
4. `bridge-post-job`

每条记录都要包含：

- 当前测试 UID。
- App canonical 路径。
- bundle id `com.codexautomation.loopedvows`。
- 写死的可执行文件名。
- 扫描前 PID 集合。
- 收到 SIGTERM/SIGKILL 的 PID 集合。
- 有界等待结果。
- 最终剩余 PID 集合。
- `verified_zero=true/false`。

### 10.1 为什么不能 `pkill -f LoopedVows`

模糊匹配可能误杀其他 Godot 项目、开发实例或主账户进程。精确 Reaper 必须同时
匹配：

```text
当前 godot-test UID
AND 可执行文件的 canonical 路径
AND canonical App 内的预期 executable
```

发送信号前重新读取 PID 身份；SIGTERM 后最多等待 3 秒，仍未退出才升级 SIGKILL，
再扫描并证明零匹配。

### 10.2 四层防线

- Bridge 启动前清理。
- Runner 启动前清理。
- Runner EXIT trap 清理。
- Bridge 任务结束后清理并验证零匹配。

任何一层失败都不能用“游戏已经测完了”作为通过理由。

## 11. 封印与独立 Validator

### 11.1 封印做什么

对 PASS 结果：

- 计算所有顶层证据文件的 SHA-256 和字节数。
- 绑定 job、task、candidate 和 result hash。
- 将文件设为只读。
- 将 `raw/` 设为不可新增。
- 将结果目录设为不可修改。

封印是本地 tamper evidence，不是签名系统。

### 11.2 Validator 独立检查什么

Validator 不接受裸 `status=passed`，至少交叉验证：

- job/task/demo/candidate 与状态机期望一致。
- build_id、source SHA、binary SHA 和 PCK SHA 一致。
- App、manifest、result 和证据路径没有穿越、符号链接或错误 owner。
- Bridge/App 都运行在期望测试 UID。
- Console user 前后未改变。
- 每个动作使用允许的 native route 和 geometry provenance。
- `reference_layout` 即使混在真实原生动作中也会整体拒绝。
- AX click 有完整回执，key/click 回执绑定正确目标。
- 每张原生截图有 hash、bytes、宽高和只读权限。
- 每个应用渲染都在 `raw-index.json` 中，且与应用/原生配对。
- 动作账本与 `result.json` 完全一致。
- 四阶段进程清理按顺序且最终 PID=0。
- evidence manifest 没有漏文件、加文件或封印后修改。
- completion marker 的 result hash 与当前结果一致。
- capability ticket 的 job id 与 candidate SHA 和结果匹配。

### 11.3 状态机也是验收者

`night-control.sh` 和 `longrun-control.sh` 是最终入口。它们必须调用 Validator；
不能因为目录里存在 `passed` 字样就推进任务状态。

## 12. 从安装到验收的操作手册

### 12.1 首次安装或升级共享 runtime

主账户：

```bash
./automation/godot/prepare-shared.sh
```

随后切到 `godot-test` 账户执行一次：

```bash
/Users/Shared/godot-playtest/runtime/install-bridge-login.command
```

该安装器负责：

- 申请 CuaDriver Accessibility、Screen Recording 和 direct-capture 权限。
- 安装 `com.codex.godot-playtest` LaunchAgent。
- 登录后自动启动 Bridge。

不要给测试账户管理员权限或 Full Disk Access。

### 12.2 构建并发布候选

```bash
candidate_sha="$(git rev-parse HEAD)"
./automation/godot/build.sh
./automation/godot/publish-build.sh /path/to/build/manifest.json
```

manifest 中的 `source_sha` 必须等于 `candidate_sha`，且游戏子树必须干净。

### 12.3 提交真实 GUI 场景

```bash
./automation/godot/submit-card-playtest.sh \
  --demo=detail \
  --task-id=your-task-id \
  --candidate-sha="$candidate_sha"
```

成功输出包含：

```text
QUEUED cards-...
RESULT /Users/Shared/godot-playtest/results/cards-.../result.json
STATUS passed
VISION /Users/Shared/godot-playtest/results/cards-.../vision.json
```

### 12.4 独立验收

```bash
./automation/godot/validate-card-playtest-result.sh \
  --result-dir /Users/Shared/godot-playtest/results/cards-... \
  --task-id your-task-id \
  --developer-commit "$candidate_sha" \
  --expected-demo detail \
  --ticket .game-auto/capability-ticket.json
```

只有出现 `PASS gui-acceptance` 才进入任务状态机。

### 12.5 本地门禁

```bash
bash -n automation/godot/cua-playtest-cards.sh
bash automation/godot/gui-native-path.test.sh
./automation/godot/validate-card-playtest-result.test.sh
./automation/godot/reap-looped-vows-processes.test.sh
```

Reaper 真实测试在 macOS 沙箱中可能因禁止读取进程表失败；这属于测试环境限制，
应在非沙箱宿主重跑，不能把沙箱拒绝误判成实现故障。

## 13. 故障矩阵

| 现象 | 可能原因 | 正确处理 |
|---|---|---|
| `godot-test bridge` 不领取任务 | LaunchAgent/Bridge 未运行 | 检查登录服务和 bridge 日志，重装仅一个安装入口 |
| 权限检查失败 | CuaDriver 权限未授予 | 在 `godot-test` 重新运行安装器并确认权限 |
| background click/key 被拒绝 | 目标窗口不接受 off-space/background | 仅在该窗口返回 exact-window foreground 建议时精确升级 |
| AX button 缺失 | 控件没有可操作语义 | 先修游戏 AX；临时自绘交互才使用实时 Control 矩形 |
| 找不到唯一窗口 | 多窗口、旧进程或尺寸漂移 | 先清理旧进程，再按 PID、AX 和连续截图确认 window |
| Runner 超时 | 场景卡住、输入失败、窗口未就绪 | 看 `bridge.log`、`actions.jsonl` 和末状态；禁止放宽验收掩盖问题 |
| 清理失败 | PID 身份漂移、SIGKILL 后仍存活或扫描失败 | 结果保持失败，检查 `process-cleanup.jsonl` |
| result 有 passed 但无 manifest | 封印失败或 Bridge 未收口 | 拒绝结果，检查 sealer 和 bridge runtime log |
| Validator 拒绝 reference layout | 使用了硬编码坐标 | 改用 AX 或实时 Control 矩形 |
| 结果封印后 hash 改变 | 后置修改证据 | 整体拒绝并重新运行 |
| completion hash 不匹配 | 结果在完成标记后被修改 | 拒绝，检查并发写和 storage |

## 14. 当前保留与已淘汰的路线

保留：

- 原生 AX 语义点击。
- 精确窗口键盘输入。
- 实时 Godot `Control` 矩形单击和拖拽。
- background 明确拒绝后的同 PID/window 前台 HID 升级。
- 应用渲染与原生窗口截图配对。
- 四阶段精确进程回收。
- 独立 sealer、validator 和 capability gate。

淘汰：

- 桌面级点击、移动和按键。
- 按窗口像素坐标硬编码的 reference layout。
- 独立 double-click 和多个重复 pixel-click helper。
- isolated pointer / `CGEventPostToPid`。
- Terminal fallback。
- `viewport.push_input` 作为验收输入。
- 按进程名模糊 kill。
- 看到 `status=passed` 就直接接受。

任何被淘汰的路线都不得通过改名重新加入。确实需要新能力时，必须同时提供：

1. 精确目标绑定。
2. 可验证输入回执。
3. Validator 接受规则。
4. 负向测试。
5. 文档和迁移理由。

## 15. 指标与质量目标

建议持续记录：

| 指标 | 目标 |
|---|---|
| GUI 验收证据完整率 | 100% |
| 四阶段清理 `verified_zero=true` 率 | 100% |
| 封印后证据修改数 | 0 |
| 无目标绑定输入数 | 0 |
| 错 candidate/build 验收数 | 0 |
| 每场景首次通过率 | 趋势指标，不为了高通过率放宽门禁 |
| 中位执行时长、P95、超时次数 | 用于容量和稳定性改进 |
| 输入路线分布 | 监控 AX 覆盖率；AX 下降时先修应用语义 |

不要用“平均看起来正常”掩盖单次清理失败。关键安全项必须是全量 100%。

## 16. 迁移到其他 GUI 项目

可复用的不是 Looped Vows 的固定坐标，而是下面这组约束：

1. 构建产物必须绑定完整源码 SHA 和产物指纹。
2. GUI 必须由非日常账户的常驻 Bridge 启动。
3. 输入必须按“AX -> 精确键盘 -> 实时控件矩形”选择。
4. 每个动作必须有目标身份和回执。
5. 每个视觉状态必须同时有内部渲染和窗口级证据。
6. 清理必须覆盖启动前、退出和任务后，且证明 PID=0。
7. 封印和验收必须独立于被测试 Runner。
8. 所有 fail-open 路径都应改成 fail-closed。

迁移时需要替换的是：

- Bundle id、App 名和 canonical executable。
- Build manifest 与发布指针。
- 场景 locator 和业务断言。
- AX 标签和 Godot Control 读取接口。

不应替换的是账户隔离、目标绑定、证据配对、四阶段清理和独立验收原则。

## 17. 教学路径

建议给新同学安排以下阅读和实操顺序：

1. 先读本文第 1、3、4、6 节，理解为什么要拆角色和证据链。
2. 阅读 `GUI-ACCEPTANCE-CONTRACT.md`，对照机器门禁。
3. 跑一次本地静态门禁和 Validator 负向测试。
4. 用 `contact-sheet.jpg` 观察一个真实结果，不打开 raw PNG。
5. 阅读一次失败运行，找出是构建、窗口、输入、截图、清理还是封印失败。
6. 在模拟 fixture 中新增一个 AX 交互，并补正向和负向测试。
7. 最后才修改场景 Runner；任何新 fallback 都先写验收规则。

新人必须能回答三个问题：

- 为什么 `status=passed` 还不是最终 PASS？
- 为什么 desktop click 和 hardcoded coordinate 不能作为验收证据？
- 为什么进程清理失败时，截图和业务断言都通过也只能判失败？

## 18. Reviewer 清单

### 代码评审

- 是否优先使用 AX。
- 是否仍绑定真实 `pid + window_id`。
- 是否新增了重复的 pixel/desktop fallback。
- key 和 click 回执是否分别校验。
- 失败路径是否仍会清理进程和写 terminal result。
- 是否更新 Validator 和负向测试。

### GUI 证据评审

- candidate SHA 与 build manifest 是否一致。
- 场景是否正确，是否来自真实输入。
- `contact-sheet.jpg` 是否展示预期状态变化。
- 应用渲染和原生截图是否配对。
- 是否有状态断言，而不只是“图片看起来像”。
- 四阶段清理是否全部为零。
- completion 与 evidence manifest 是否存在且 hash 一致。
- capability ticket 是否引用本次 job 和 candidate。

## 19. 相关文档

- 验收条款：[`GUI-ACCEPTANCE-CONTRACT.md`](GUI-ACCEPTANCE-CONTRACT.md)
- 安装与安全边界：[`SECURITY.md`](SECURITY.md)
- 连续自动化运行手册：`automation/nightshift/CONTINUOUS-OPERATIONS-PLAYBOOK.md`

本文是体系总览；字段级准入条件仍以 `GUI-ACCEPTANCE-CONTRACT.md` 和
`validate-card-playtest-result.sh` 为最终事实来源。
