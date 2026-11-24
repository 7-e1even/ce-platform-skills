# CE 知识平台技能

给 [Multica](https://multica.ai) 上的 agent 用的技能,让它们充当认知酶(Cognitive Enzyme,CE)的知识平台:
从 CE 收到的对话里沉淀经验,通过 CE 的维护 MCP(ce-maintain)读写。现在有两步:「记事」只记发生过的事,
写成候选记忆(隔离状态、`candidate` 标签),不往线上注入任何东西;「修经验」拿注入后失败的对话,改正已在用的记忆。

| 技能 | 做什么 | 给谁 |
|---|---|---|
| [`ce-recorder`](skills/ce-recorder/SKILL.md) | 记事:增量读对话,给已有记忆加证据,或写成候选 | 记录员 |
| [`ce-reviser`](skills/ce-reviser/SKILL.md) | 修经验:看注入了却失败的那些次,改正写错的那句、加区分条件或把触发条件写窄,压到 150 字以内 | 修订员 |
| [`genius-thinking`](skills/genius-thinking/SKILL.md) | 天才思维:五条法则(拒绝放弃、为什么不、一带百万、开放、加速改进本身)+ 读懂、抓本质、发散、举一反三、检验 | 所有 agent |

`genius-thinking` 以《天才是怎样思考的》的五条法则为骨架,配合《天才的13个思维工具》的具体动作,用自己的话改写成智能体能照做的步骤,没有摘录原文。

## 导入 Multica

技能 → 新建 → 从网址导入,分别填:

```
https://github.com/7-e1even/ce-platform-skills/tree/main/skills/ce-recorder
https://github.com/7-e1even/ce-platform-skills/tree/main/skills/ce-reviser
https://github.com/7-e1even/ce-platform-skills/tree/main/skills/genius-thinking
```

从网址导入的技能记着来源,这里更新后在技能页点「从源更新」(或 `multica skill refresh <技能 id>`)就能拉到新版。

## 配置顺序

1. **工作空间**:新建一个,比如「CE 知识平台」。建好后重启守护进程(重开 Multica 桌面端,或
   `multica daemon restart`),它会给新空间注册本机的运行时。
2. **项目**:建「知识平台」项目;在桌面端的项目 → 资源 → 本地文件夹,绑定 `~/ce-platform`,选原地(in place)。
   Multica 默认每次运行都在新目录里跑,绑定固定目录后 CE 才能按项目名认出平台自己的对话。
3. **CE 那边**:运行参数 → 数据 →「不收这些项目的对话」加上 `ce-platform`。
4. **MCP**:设置 → MCP 服务器 → 添加,名称 `ce-maintain`,配置从 CE 接入页「改记忆 → 更多选项」的 Multica 片段复制,
   令牌用 CE「平台 → 服务令牌」里给 Multica 建的那枚。
5. **技能**:按上面导入三个技能。
6. **agent「记录员」「修订员」**:见下;MCP 页「从工作区添加」ce-maintain,技能分别加 `ce-recorder` / `ce-reviser`,
   都加 `genius-thinking`。
7. **Autopilot「记事」「修经验」**:见下。先各点一次「立即运行」,看报告和 CE 后台(审计日志里有改前改后),再开定时。

第一次跑完检查:CE 后台记忆页筛「待审」看候选和证据;对话原料页看平台自己的对话有没有混进来
(混进来了就把那段对话的项目名补进「不收这些项目的对话」)。

### 记录员

- 运行时:便宜、上下文长的模型就够(比如 pi + DeepSeek),它只读和记,不写代码。
- 并发:1(游标是一条线,两个同时跑会重复处理)。
- 指令:

```
你是认知酶(CE)知识平台的记录员。你只做「记事」:按 ce-recorder 技能,从 CE 新交回的对话里记下发生过的事,
给已有记忆加证据,或写成候选记忆。读对话时用 genius-thinking 的法则四(开放)和第 1 步,写经验时用第 2 步。
不改已有记忆,不转正,不删除。工单里只写报告,不贴对话原文。
```

### 记事 Autopilot

- 名称:记事;指派:记录员;项目:知识平台
- 执行方式:创建工单(报告留在工单里)
- 触发:定时 `0 */6 * * *`,时区 Asia/Shanghai
- Runbook:

```
用 ce-recorder 技能处理 CE 新交回的对话:从上次的游标开始,最多处理 20 段,
处理完存游标,按技能里的模板在本工单写报告。没有新对话就报告「没有新对话」后结束。
```

存量对话较多时,先手动「立即运行」几次把积压处理完,再交给定时。

### 修订员

- 运行时:要会判断失败原因、改写得准,用比记录员强一点的模型;它会改线上在用的记忆。
- 并发:1。
- 指令:

```
你是认知酶(CE)知识平台的修订员。你只做「修经验」:按 ce-reviser 技能,看注入了却失败的那些次,读失败的对话,
判断是记忆写错了、用错了场合,还是别的原因;前两种就改正那句或把触发条件写窄,压到 150 字以内写回。
读对话时用 genius-thinking 的第 1 步,改写时用第 2 步。不新建、不驳回、不删除记忆,不改状态和置信度。
工单里只写报告,不贴对话原文。
```

### 修经验 Autopilot

- 名称:修经验;指派:修订员;项目:知识平台
- 执行方式:创建工单(报告留在工单里)
- 触发:定时 `30 3 * * *`(每天一次),时区 Asia/Shanghai
- Runbook:

```
用 ce-reviser 技能处理近 7 天注入后失败的那些次(list_surprises,min_expected=0):按记忆归拢,
一次最多 10 条记忆,按技能里的模板在本工单写报告。没有失败就报告「没有要修的」后结束。
```

CE 那边要先开着进化记账(注入时随机扣下对照、判定对话结果),才会有「注入后失败」的记录。

## 相关

CE 这边的接口约定(读对话、游标、证据、写回把关)见 Cognitive-Enzyme 仓库的 `docs/knowledge-platform.md`。
