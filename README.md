# 星火燎原 · 抖音自动续火花

> ## ⚠️ 升级警告：本版本是一次大重构，请仔细斟酌后再升级
>
> 这次更新重写了 **脚本输出方式（`ctx.result`）、会话快照的数据结构与生命周期、
> 编辑器内核（CodeMirror 6）以及多处界面交互**，与旧版差异很大，**不是小修小补**。
>
> - **旧脚本可能要改**：`ctx.sandbox.*` 已改名 `ctx.macro.*`；`ctx.template.变量名` 改为
>   `ctx.template.vars.变量名`；`$spark_result$` 仍然兼容，但新写法是 `ctx.result`。
> - **界面变化**：会话列表多了「被隐藏」分组，宏 / 函数 / 沙盒变量改成折叠卡片。
> - **数据变化**：快照改为合并写入（IM 分批加载不会再清空会话）；连续 3 次没出现的会话会隐藏、
>   30 天后连快照一起删除。
>
> **升级前请先「设置 → 配置」导出备份；升级后先核对脚本、已选目标与模板。**
> **不确定就先别升级，或者等一两个小版本稳定后再升。**

一个 LSPosed 模块：注入抖音进程，**自动给「今天还没互动」的会话续火花** —— 发一条你自己写的文案。
文案是一段 JavaScript，由「模板 + 两层变量 + 宏 + 自定义函数」组合出来，目标是「每天自动续，但话说得像真人说的」。

作用域：`com.ss.android.ugc.aweme`（抖音）

---

## 它做什么

抖音的火花需要双方每天有互动。这个模块会在**抖音启动 / 回到前台 / 跨天时**扫一遍会话，把「今天还没互动」的挑出来，
按你配置的脚本生成文案发过去；每个会话每天最多发一条。

- **待发**：今天还没发过 → 会发
- **待对方回复**：今天已经发过了、火花还是灰的（对方还没回） → 不重复发
- **今日已续**：今天已经互动过 → 跳过
- **待重燃**：火花断在恢复期 → 按抖音判定处理

打开抖音、跨天补跑、手动发送之后，模块会把会话快照**自动写回管理界面**，不用打开 App 也能看到最新状态。

## 功能

- **模板**：内置模板（空白 / 按时问候 / 火花天数打卡 / 简短提醒）+ 自定义模板；一级模板可展开看派生模板（跟随它的会话）
- **空白模板 = 不用模板、自己写**：照常发送，只是父模板内容是空的
- **变量（两层）**：模板一套、每个会话还能有自己一套；同名覆盖、空值回落到父模板、都没有就是 `null`；
  变量值是 JS 表达式，能读 `ctx`、能引用排在前面的变量；标题栏「试算」可以看整套变量的当前结果
- **宏**：挂在模板或会话上的小脚本，在模板之后执行；适合节日倒计时这类临时代码，用完删掉不影响模板；
  宏可以用 `ctx.macro` 读写跨次保留的沙盒变量（`ctx.sandbox` 是旧名）
- **自定义函数**：全局定义，执行前注入成 `ctx.functions.<名字>`
- **系统接口**：`ctx.system.getAllConversation(filter?)` 按条件拿本轮会话（可按 `willSend` / `autoSend` /
  `manualSend` / `isGroup` / `days` 等字段筛，也可以传函数）；`mountMacro / unmountMacro / unmountAll` 动态挂载；
  `ctx.registerCallback(fn)` 在发送结束后回调 `{ conversationId, result: "Succeed"|"Skipped"|"Failed", reason? }`
- **编辑器**：CodeMirror 6（离线内置），输入时自动补全 `ctx.*` 与 JS 内置成员、类型推断、悬浮文档；
  **粘贴 / 格式化**（清理零宽字符、智能引号、代码围栏并重排缩进）；用真实会话快照**预览**；
  长按走系统原生选区，动作条是图标（长按看名称）
- **会话快照生命周期**：连续 3 次没在抖音出现的会话收进「被隐藏」折叠卡片，满 30 天连快照一起删除
- **配置按项导出 / 导入**：勾模板会自动带上父模板、宏、脚本里用到的函数；导入是合并
- **使用文档**：App 内可看 / 可导出 / 可复制全文；里面有一节可直接贴给 AI 的提示词

## 脚本 API 速查

```js
// 输出：往列表里 push，多行会被拼成一条消息；空列表 = 这条不发
ctx.result.push("早上好，第 " + ctx.io.days + " 天的火花啦 🔥");

// 内置能力
ctx.builtin.date / time / weekday / timestamp
ctx.builtin.pick(list) / pad / repeat / random / shuffle / countdown / nextNewYear / untilNewYear / formatDate

// 宿主数据
ctx.io.nickname / remark / name / days / state / stateText / peerUid / conversationId

// 模板变量（含两个固定键）
ctx.template.vars.称呼 / ctx.template.template / ctx.template.parent

// 自定义函数 / 宏变量 / 系统
ctx.functions.名字(...)
ctx.macro.getvar(name) / setvar(name, value) / deletevar(name)
ctx.system.getAllConversation({ willSend: true })
ctx.system.mountMacro("宏 id", { conversation: "会话 id" })
```

> 旧写法 `$spark_result$` 和 `ctx.result` 是同一个数组，老脚本不用改。

沙盒示例（宏里记录跑过几次）：

```js
var n = (ctx.macro.getvar("runs") || 0) + 1;
ctx.macro.setvar("runs", n);
ctx.result.push("（第 " + n + " 次加这句）");
```

## 安装

1. 安装 APK，在 **LSPosed** 里启用模块，作用域勾选「抖音」，重启抖音
2. 打开模块界面（桌面图标，或 **抖音 → 我 → 设置 → 续火花**）
3. 设置里打开总开关 → 点一次「刷新会话」
4. 模板页给会话选模板；变量在模板 / 会话的「变量」入口里加；宏与函数在「宏/函数」页里加

## 说明

- 只在抖音进程内工作，不联网上传任何数据；配置保存在模块 App 自己的私有目录
- 每个会话每天最多发一条，发送记录按天保存
- 群聊默认不参与自动发送（手动移进「已选目标」才发）
- 脚本上下文按来源分成 `ctx.builtin` / `ctx.io` / `ctx.template` / `ctx.system` / `ctx.macro` / `ctx.functions`；
  旧写法（`ctx.name`、`ctx.pick(...)`、`ctx.sandbox.*`）不再推荐

## 许可与致谢

- 许可：Apache License 2.0
- 思路来源：[yfishyon/doumcp](https://github.com/yfishyon/doumcp)（抖音逆向与注入基础）、
  WorldMargin 的模板 / 宏 / 函数组合思路与 DouYinSpark 油猴脚本原作
