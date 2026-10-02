# EchoWeave

原作角色平行剧情项目。当前已完成 Summer Pockets 的本地素材整理、局部场景包草案、可重放状态运行时，以及中央 Agent 的模型接口与工具循环。

`agent-play` 接入用户配置的 OpenAI 兼容 Chat Completions API，由模型决定 NPC 工具动作。当前已通过模拟模型和本地 HTTP 服务测试，并核对了用户实际运行的首个真实模型回合；多回合稳定性与生成质量尚待评测。秘密基地初始化采用开发用 IF 设定，原作场景顺序仍待核验。

## 先跑模拟模型

需要 Python 3.10 或以上，无第三方依赖。在项目目录执行：

```powershell
python run.py demo --output .local/demo-result.json
python run.py agent-demo --storypack .local/storypacks/sp-secret-base-0729-draft --output .local/agent-demo-result.json
python run.py environment-demo --db .local/environment-demo.sqlite3 --storypack .local/storypacks/sp-secret-base-0729-draft --output .local/environment-demo-result.json
python run.py play
```

`demo` 会新建三个演示分支，验证私聊、显式转告、祖先分叉截止和状态重放；重复运行不会删除已有存档。
`play` 是离线交互，成功的操作会自动写入 `.local/runtime.sqlite3`，退出后可以继续。
`agent-demo` 使用模拟模型，实际执行检索、工具反馈、候选动作、原子提交和 receipt 重试，不产生外部模型费用。
`environment-demo` 使用模拟模型跑完求助、回应、到场、协作，并验证未求助的另一分支仍受阻。每次演示创建新分支。

## 接入已有 API

首次运行时复制配置模板。如果已经有 `.env`，直接编辑现有文件。

```powershell
if (-not (Test-Path -LiteralPath '.env')) {
    Copy-Item -LiteralPath '.env.example' -Destination '.env'
}
```

在 `.env` 填写 `ECHO_LLM_BASE_URL`、`ECHO_LLM_MODEL`、`ECHO_LLM_API_KEY`。Base URL 包含供应商要求的版本路径，例如 `/v1`，程序会追加 `/chat/completions`。密钥留在本地；`.env` 已排除提交。已有进程环境变量优先于配置文件。

```powershell
python run.py agent-play --environment --storypack .local/storypacks/sp-secret-base-0729-draft
```

模型需支持 function tools。若供应商要求 `max_tokens`，修改 `ECHO_LLM_OUTPUT_TOKEN_FIELD`；若不支持 `strict`，把 `ECHO_LLM_STRICT_TOOLS` 设为 `0`，服务器仍会检查工具参数。其他接口协议需要另外适配。

`agent-play` 默认创建或恢复 `agent-main` 分支，与离线 `main` 分开。剧情回合会调用已配置的 API。研究查询、分叉和切换不调用模型。材料版本变化时请用新的 `--branch`，或恢复匹配原材料的分支。

`--environment` 启用原创 IF 求助推进规则。旧存档通过新增 `environment_enabled` 事件启用，之前的历史、快照和 receipt 保持原样；再次启用不会重复写入。已经打开的交互进程需要先输入 `退出`，再用上面的命令启动。

当前场景包只提供选中的开场证据与局部角色草案；不是通用故事包解释器，也不会把待核验资料升级为已审核原作事实。不提供 `--storypack` 时使用原创机制测试材料。

## 交互示例

在 `agent-play` 或离线 `play` 中逐行输入：

```text
私聊 天善 先寻求协助，暂时不要进入基地。
视角 天善
视角 良一
等待
等待
观察
分支 alternate 0
离开
状态
切换 agent-main
状态
退出
```

`agent-play` 中还可输入 `重试`，用同一 turn_id 恢复上一回合结果。已提交的回合会返回原 receipt，不再次调用模型；未提交的失败回合会重新调用模型，不自动重试 HTTP 请求。

天善的角色视角包含私聊，良一的角色视角不包含；显式转告或公开发言会改变接收范围。`alternate` 从版本 0 分叉，不继承父分支之后发生的私聊或求助。离线 `play` 的默认分支为 `main`，切换时使用对应名称。
启用环境时，求助后第一次玩家等待收到回应，第二次等待协助者到场。到场后，NPC 或玩家调用 `coordinate_assistance` 才能使入口恢复通行；玩家命令为 `协作`。若 NPC 已经协调处理，直接观察入口即可。NPC 连续发言或等待不会推进回应阶段；公开台词也不能直接改动世界事实。

不带 `--environment` 时继续沿用原有的求助记录模式。也可以在交互中输入 `启用环境`，或用 `python run.py enable-environment --branch agent-main` 单独启用，无需调用模型。

角色视角、完整历史和权威状态查询属于开发研究入口。中央控制器只接收被激活角色的合法视角，角色标签明确区分；完整研究历史不会作为模型输入。本版中央模型仍共享多个角色视角，不能声称已经解决跨角色的语义泄露。

## 已实现的机制

- SQLite 保存事件、快照、分支及 turn receipt，进程重启后可恢复。
- 私聊、公开发言和地点观察具有不同接收范围；离场角色不会自动听见原地点的新发言。
- 分支逐层继承祖先截止点以内的事件，可以从历史快照继续，父分支后来发生的事不污染子分支。
- 动作先在内存裁决，事件、快照、分支版本和 receipt 在同一个事务中提交。
- 相同请求重试返回原 receipt；相同 turn_id 的不同请求或过期状态版本被拒绝。
- 同一 reducer 用于提交和重放，历史快照可逐版本核对。
- 模型调用位于数据库写事务外；玩家输入和 NPC 动作先形成候选，模型结束后再按版本原子提交。
- 工具参数在服务器校验，不允许替玩家行动、修改任意状态或选择另一分支；错误反馈可供模型修正。
- 每回合默认最多 4 次模型调用、8 次工具调用、7 个 NPC 动作、60,000 字符上下文；输出 Token 上限可配置。
- HTTP 协议错误、预算耗尽、取消或过期候选不提交。研究轨迹记录工具结果与用量，不记录模型内部推理；缺失用量保持未知。
- 正式剧情只展示玩家观察到的已提交事件，模型最后的自由文本不会直接改写世界。
- 环境规则与版本指纹写入存档；自动回应和到场事件共用候选／提交／重放路径，按现场角色确定可见范围。
- 模型视角只提供一份当前事实、最近 12 条消息和最近 6 条非消息观察；动作反馈提供本次可见增量。被裁剪的旧消息可通过 `recall_history` 按角色与分支检索。

这里的裁剪与历史查询是确定性处理，没有生成摘要或使用向量缓存。权限机制保证上下文材料的范围，模型接入后的语义泄密和幻觉需要另外评测。

## 验证

```powershell
python -m unittest discover -s tests -v
```

测试覆盖私聊与转告、离场观察、祖先和嵌套分支截止、失败回滚、跨重启幂等、并发版本冲突、台词与事实的区别，以及逐版本历史重放。

2026-10-02 最新验证：93 项测试通过，覆盖状态运行时、Agent 工具循环、中文交互、localhost HTTP 模拟、环境因果规则和上下文裁剪。环境演示的 5 项检查通过，结果在 `.local/environment-demo-result.json`。早期离线与模拟 Agent 验证记录仍保留。这些结果不包含真实模型生成质量或多 Agent 对照实验成绩。

用户在此前版本实际跑通首个模型回合，存档核对：4 次模型调用、6 次合法工具调用，保存玩家消息及 4 个 NPC 动作；供应商报告总计 17,425 Token，提交前耗时约 26.2 秒。角色视角检查与重放一致性通过，记录在 `.local/validation/2026-10-02-first-model-turn.json`。该旧回合没有启用环境规则，历史仍按原状重放。

使用该回合保存的 5 份角色视角比较，紧凑表示的序列化字符数减少约 53%～59%，记录在 `.local/validation/2026-10-02-context-compaction.json`。这只衡量角色视角，不含系统提示、工具定义和完整请求；不能当成真实 Token、费用或耗时降幅。新版尚未额外调用真实 API 验证性能。

45 秒回合截止与取消采用协作式检查：模型调用前后及提交阶段检查；HTTP 有单次 I/O 超时，目前不能即时打断正在进行的网络请求。并发重复请求只提交一份结果，但可能已各自调用模型；丢弃尝试的调用量单独记录。当前为可信本地开发入口，未实现生产身份认证、摘要或向量检索。

## 接下来

启用环境后跑约 10 回合，核对真实模型能否依据回应、到场和工具前置条件继续剧情，并记录新版请求大小与实际用量；随后让两位角色使用独立上下文，再做私聊与分支污染评测、同预算中央单 Agent 对照。当前关键词检索只是证据接口的起点，尚未实现完整人设、信念更新、语义校验或公平对照实验。

接入边界与验证记录见 `docs/Agent-接入与验证.md`；素材准备记录在 `docs/data/`，局部场景任务在 `docs/scenarios/`。原作文本、包文件、数据库与研究轨迹保存在本地。
