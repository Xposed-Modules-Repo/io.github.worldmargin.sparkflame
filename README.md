# 星火燎原 · 抖音自动续火花

一个 LSPosed 模块：注入抖音进程，**自动给「今天还没互动」的会话续火花** —— 发一条你自己写的文案。
文案是一段 JavaScript，由「模板 + 两层变量 + 宏 + 自定义函数」组合出来，目标是「每天自动续，但话说得像真人说的」。

作用域：`com.ss.android.ugc.aweme`（抖音）

---

## 它做什么

抖音的火花需要双方每天有互动。这个模块会在**抖音启动 / 跨天时**扫一遍会话，把「今天还没互动」的那几个挑出来，
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
- **宏**：挂在模板或会话上的小脚本，在模板之后执行；适合节日倒计时这类临时代码，用完删掉不影响模板
- **自定义函数**：全局定义，执行前注入成 `ctx.functions.<名字>`
- **沙盒变量（只给宏）**：`ctx.sandbox.getvar / setvar / deletevar`，值以 JSON 存进配置、跨次运行保留；
  归属就是宏自己，模板读不到也写不了，删宏时它的键一起清掉
- **系统接口**：`ctx.system.getAllConversation()` 拿本轮会话列表；`ctx.system.mountMacro / unmountMacro` 动态挂载 / 摘离宏
- **编辑器**：Monaco（VSCode 同款，离线内置），`ctx.*` 与 JS 内置成员的补全、类型检查、悬浮文档、格式化、双指缩放；
  长按弹悬浮图标条（全选 / 复制 / 粘贴 / 撤销）；也可以切回原生编辑器
- **预览**：用真实会话快照渲染一遍，与实际发送走同一套组装
- **配置按项导出 / 导入**：勾模板会自动带上父模板、宏、脚本里用到的函数；导入是合并
- **使用文档**：App 内可看 / 可导出，最后一节是「给 AI 的模块说明书」（把整篇丢给 AI，它就能照着设计写模板）

## 脚本 API 速查

```js
// 内置能力
ctx.builtin.date / time / weekday / timestamp
ctx.builtin.pick(list) / pad / repeat / random / shuffle / countdown / nextNewYear / untilNewYear / formatDate

// 宿主数据
ctx.io.nickname / remark / name / days / state / stateText / peerUid / conversationId

// 模板变量（含两个固定键）
ctx.template.称呼、ctx.template.template、ctx.template.parent_template

// 自定义函数 / 沙盒 / 系统
ctx.functions.名字(...)
ctx.sandbox.getvar(name) / setvar(name, value) / deletevar(name)
ctx.system.getAllConversation() / mountMacro(name) / unmountMacro(name)

// 输出：往列表里 push，多行会被拼成一条消息；空列表 = 这条不发
$spark_result$.push("早上好，第 " + ctx.io.days + " 天的火花啦 🔥");
```

沙盒示例（宏里记录跑过几次）：

```js
var n = (ctx.sandbox.getvar("runs") || 0) + 1;
ctx.sandbox.setvar("runs", n);
$spark_result$.push("（第 " + n + " 次加这句）");
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
- 升级到 v1.4.0 起：`ctx` 按来源分成 `ctx.builtin` / `ctx.io` / `ctx.template` / `ctx.functions`，
  旧写法（`ctx.name`、`ctx.pick(...)`）不再可用

## 许可与致谢

- 许可：Apache License 2.0
- 思路来源：[yfishyon/doumcp](https://github.com/yfishyon/doumcp)（抖音逆向与注入基础）、
  WorldMargin 的模板 / 宏 / 函数组合思路与 DouYinSpark 油猴脚本原作
