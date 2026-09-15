# 棱镜OS给 Codex 的安装说明

优先读取当前 Workspace 内的本说明。安装分为“离线准备”和“联网完成”，离线准备不要求先成功访问服务器。

## 0. 本地检测与离线准备

先检查当前 Workspace 的 .lengjing/workspace-token.txt、.lengjing/bootstrap-input.json、.lengjing/install-state.json 和 PrismOS-install-kit 目录，不读取其他 Workspace。
- 已安装且用户请求升级：走已有 Runner 的升级流程，保留用户配置，不执行首次初始化。
- 有本地资料包：先运行 `python3 PrismOS-install-kit/bootstrap.py --project-dir .`，不联网、不安装系统依赖。包在根目录时相应调整路径。
- 没有资料包：可向 https://prismaigent.com/system/install-for-codex 发起一次有界读取，连接超时10秒、总时长30秒；不要把读取与执行合并成 curl | bash。
- 只有网络已获允许而官网发生DNS/连接失败时，才尝试一次公开备用说明：https://raw.githubusercontent.com/Andytian3/PrismOS-Install/main/INSTALL_FOR_CODEX.md 。
- 权限审核待定、超时或拒绝时，不要反复申请或切换工具/域名绕过；展示“正在等待联网授权，可以先下载本地资料包完成填写”。提供官网资料包 https://prismaigent.com/system/install-kit 和备用 https://github.com/Andytian3/PrismOS-Install/releases/latest ，由用户自行下载，不自动打开浏览器。
- 网站与镜像不匹配时停止混用，优先同一资料包中的manifest版本和文件哈希。哈希用于完整性校验，不代替对下载来源的信任。

## 1. 先填写初始化信息

有本地包时，让用户打开包内 init-form.html；无本地包但网页可用时使用 https://prismaigent.com/system/init-form 。
一张HTML包含所有初始化内容，无需安装Runner即可填写。不要让用户在终端填写，也不要索取服务器FMP Key。
用户点击“检查并复制 JSON”后粘贴到当前Codex。按下面第3节原样保存、简要确认；已经保存的不重复填写。
若尚未有Python3.9+，仍可填写HTML；解释缺少本地运行环境，不把它误报为网站不安全或安装失败。

## 2. 联网完成与恢复

用户确认配置后，在联网权限可用时运行 `python3 PrismOS-install-kit/bootstrap.py --project-dir . --install`。
资料包已包含Runner源码，无需再次下载安装说明和源码；安装Python依赖及注册服务器仍需要联网，不能宣称完全离线安装。
安装完成后按第4节提交 .lengjing/bootstrap-input.json。用户未确认不得注册或提交真实持仓。
失败保留本地资料、JSON和非敏感安装进度；网络恢复后重复同一命令继续，不重新创建Workspace或Token。
错误必须区分：DNS/连接失败、等待授权、服务HTTP错误、本地依赖失败。不能把所有非零退出或404都判断为需要升级。
用户请求升级已有安装时不要覆盖源码；使用当前Runner正式升级入口，不擅自重装或修改系统Python。
用户侧只访问HTTPS域名，不使用服务器IP、SSH或端口。网络权限由宿主控制，脚本不能批准或绕过。

## 3. 保存初始化 JSON

用户粘贴 JSON 后，先确认它是 JSON object，且包含：

```text
workspace, portfolio, watchlist, personal_style, strategy, research
```

然后在当前 workspace 执行：

```bash
mkdir -p .lengjing
```

把用户粘贴的 JSON 原样保存到：

```text
.lengjing/bootstrap-input.json
```

不要把原始 JSON 再完整贴回给用户。请只用中文简要汇总：

- Workspace 名称。
- 持仓数量和关注池数量。
- 个人投资风格模式。
- 做多策略类型。
- 评分权重。
- 研究框架选择。
- 通知渠道是否配置。

然后询问用户是否确认提交。

## 4. 提交初始化

用户确认后执行：

```bash
.venv/bin/python .lengjing/prismos-source/apps/runner/scripts/install_local_bot.py \
  --project-dir "$PWD" \
  --bootstrap-file .lengjing/bootstrap-input.json
```

初始化会完成：

- 首次安装自动生成 Workspace Token；迁移或重绑时使用用户提供的旧 token。
- 写入 `.env` 和 `.lengjing/bootstrap-*.json`。
- 创建本地可见目录 `棱镜OS/99_运行日志`。
- 创建或恢复两个固定 Codex session：
  - `棱镜OS运行监督`：同步系统正在做什么、完成了什么、是否失败。
  - `棱镜OS用户确认`：只处理需要用户明确确认的任务。
- 通过 HTTPS 域名提交初始化到棱镜OS服务器。
- 为初始化持仓和关注池 ticker 创建初始研究信息包，并交给任务编排器。
- macOS 默认安装每 5 分钟执行一次的后台同步任务。

Workspace Token 会明文登记到服务器，并保存到本地 `.lengjing/workspace-token.txt`。以后新 workspace 输入同一个 token，即可绑定到服务器上的同一组历史状态和认知。

## 5. 初始化提交后的继续流程

提交初始化后，不能只回复“安装完成”。必须检查命令输出：

- 如果用户选择“使用已有个人投资风格 ID”，确认 `personal_style_status` 为 `ACTIVE`；成功后不要再进入问答流程。
- 如果服务器返回 `PERSONAL_STYLE_ID_INVALID` 或 `PERSONAL_STYLE_SNAPSHOT_MISSING`，立即停止并让用户从原 Workspace 的 Dashboard 重新复制 ID，不得自动降级为创建新画像或跳过。
- 如果 `server_status` 是 `WAITING_PERSONAL_STYLE`，或 `next_required_actions` 包含 `ANSWER_PERSONAL_STYLE_QUESTIONS`，继续处理个人投资风格初始化。
- 先执行一次待办拉取：

```bash
.venv/bin/lengjing-runner --remind-user-actions --action-type PERSONAL_STYLE_INITIALIZATION --limit 20
```

找到 `PERSONAL_STYLE_INITIALIZATION` 的 `task_id` 后，在当前对话里继续问用户，不要让用户去终端填写。

开始提问前，必须先说明：

- 本轮覆盖 8 个主题模块。
- 大约 30-40 个具体问题。
- 包含选择题、区间判断题和少量开放描述题。
- 建议按真实习惯回答，不确定可以说“大概”“不确定”或给范围。
- 预计需要 15-25 分钟。
- 个人投资风格非常重要，系统了解个人投资风格后，才能减少投资决策里的犯错。

个人投资风格问答至少覆盖：

- 投资周期。
- 风险承受。
- 最大回撤。
- 仓位集中度。
- 偏好公司类型。
- 买入触发。
- 卖出或减仓规则。
- 不碰的行业或情形。

收集完成后，整理为 `.lengjing/personal-style-submit.json`，并执行：

```bash
.venv/bin/lengjing-runner --submit-user-action \
  --task-id "PERSONAL_STYLE_TASK_ID" \
  --action-type PERSONAL_STYLE_INITIALIZATION \
  --payload-file .lengjing/personal-style-submit.json
```

如果没有个人风格待办，说明系统已进入初始持仓和关注池研究同步阶段，并提示可以手动同步一次：

```bash
.venv/bin/lengjing-runner --sync-local --workspace-id "$LENGJING_USER_ACTION_WORKSPACE_ID" --worker-task-limit 5 --limit 20
```

## 6. 日常修改：白名单和持仓

### 6.1 投资研究和投资决策意图

用户表达“我准备买入/卖出/加仓/减仓某个股票”“帮我研究某个标的并做投资决策”“我有一个关于某股票的投资想法”时，必须先登记到棱镜OS服务器并触发全流程。不要直接调用 Codex 公共股票研究 Skill 独立生成报告，也不要只把结果写到本地文件。

动作映射：

- 准备买入、加仓、建立仓位：`--intended-action BUY`
- 准备卖出、减仓、清仓：`--intended-action SELL`
- 只要求研究但没有交易倾向：`--intended-action RESEARCH`

仓位语义必须严格区分：

- “再买入 1% / 加仓 1% / 卖出 1%”是计划交易增量，使用 `--planned-trade-weight-pct 1`。
- “买到 5% / 目标仓位 5% / 调整到 5%”是最终目标仓位，使用 `--target-weight-pct 5`。
- 不要把“想买入/准备买入/考虑加仓”解释成已成交后的当前持仓。

示例：

```bash
.venv/bin/lengjing-runner --request-investment-research AAPL \
  --intended-action BUY \
  --planned-trade-weight-pct 1 \
  --thesis "用户准备再买入1%仓位，希望完成研究并形成投资决策" \
  --change-note "用户在当前对话提出AAPL 1%加仓意图"
```

默认会：

- 把用户投资意图登记为服务器信息入口。
- 由服务器判断 30 天内是否已有可用上游；有可用仓位建议就直接进入投资决策，有可用综合分析就进入仓位建议，缺综合但四类底层分析齐全就重组综合，底层缺什么才补什么。
- 若用户明确提出买入或卖出意图，即使机械仓位建议不是 `need_action`，也进入投资决策建议任务，用来回答“是否应该执行、修改或拒绝这个用户意图”。
- 立即执行一次本地同步并领取后续任务。

用户要求“加入白名单 / 加入关注池 / 观察某个股票”时，不要重新提交完整初始化配置。直接调用单项白名单接口：

```bash
.venv/bin/lengjing-runner --add-watchlist-ticker ORCL --change-note "用户要求加入关注池"
```

默认会：

- 把 ticker 加入当前 workspace 的关注池。
- 注册到研究触发器，默认新闻触发阈值 8.5 分、价格异动 20%、单日异动 10%。
- 为新关注标的创建初始研究链路。
- 立即执行一次本地同步，最多领取 10 个 worker 任务；信息处理完成后会继续领取后续基本面、宏观、估值和技术面分析，不等待 5 分钟后台定时器。

如果用户只是调整关注池但不想立刻研究，增加：

```bash
--no-initial-research
```

如果用户只想提交变更、不想本轮立即领取任务，增加：

```bash
--no-post-change-sync
```

只有用户明确表示“已经成交 / 已买入 / 已卖出 / 当前持仓就是 X% / 把系统里的当前仓位改成 X%”时，才允许更新持仓快照。按当前快照提交：

```bash
.venv/bin/lengjing-runner --upsert-position-ticker MNSO --current-weight-pct 3 --avg-cost 20 --confirmed-executed-snapshot --change-note "用户确认已成交，更新当前真实持仓"
```

规则：

- `--current-weight-pct` 是当前账户里该股票占总资产的百分比；清仓填 `0`。
- 新持仓通常需要 `--avg-cost`；已有持仓可以不填成本价，系统沿用服务器已有成本价。
- `--current-price` 可不填；系统日常行情更新会维护现价。
- `--confirmed-executed-snapshot` 表示这不是计划交易，而是已成交后的真实持仓快照。缺少这个确认时，命令会拒绝执行。
- 修改持仓后会同步研究触发器，持仓标的优先级高于普通白名单。
- 修改持仓后也会立即执行一次本地同步，用于领取因此产生的研究、修复或用户确认任务。

## 7. 错误处理

如果命令输出里出现 IP 地址、SSH 连接、端口、服务器内部路径或公网主机错误，不要原样展示给用户。统一说明：

```text
服务器同步暂时不可用，请稍后重试或联系支持。
```

表单网页不可达时优先使用资料包内 init-form.html，不需要等待网页恢复。仅在没有本地表单且用户明确选择逐项问答时，使用旧 lengjing-init-flow；不得要求已填写的用户重新输入。
