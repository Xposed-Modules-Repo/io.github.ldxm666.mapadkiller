# Changelog

## v1.0.7 (2026-09-18)

### 修复：高德首页「删了标签，悬浮栏没被占满」

藏掉标签只改了槽位宽（weight 平分），但那一层**选中胶囊** `id=tab_bg_layer` 的宽度
是高德自己按 `totalWidth / 标签总数` 算死的，它不知道我们藏了几个。
于是 5 个标签变 2 个之后胶囊还是 201px，首页 / 我的悬在 1/4、3/4，中间和两头全是空的。

现在每个可见槽位的胶囊层会被撑成槽位宽度本身 —— 等价于高德自己按 2 个标签排出来的样子。
真机实证：`TAB-FILL slot=503 visible=2 rowW=1006`，整条悬浮栏占满。

### 修复：「部分隐藏」会把路线规划页的信息模块一起抹掉（严重）

根因是**整棵 decor 树照单全收**：路线页备选卡片的里程文案 `2379公里` / `34米`
命中了首页信息流的「距离标签」规则（`isDistanceLabel`），`collapseHost` 顺着父链
爬到列表宿主，把「24小时22分 / 大众常选 / 封路」那一整排路线信息模块 GONE 掉。

现在加了**页面闸门**（铁律 G）：首页与「我的」页都带底部悬浮标签栏
（`LiteTabBar > TabItemLayoutV2`），路线页 / 搜索页 / 导航页没有。
认不出标签栏就整轮放弃，一个节点都不碰；列表绑定钩子同样受这道闸门约束。
真机实证：进入路线页后 `COLLAPSE` / `BIND-HIDE` 归零，三个备选路线卡片完整。

### 新增：隐藏桌面图标（可开关）

`LAUNCHER` intent-filter 移到 `activity-alias`（`.LauncherAlias`）上，
设置页新增「隐藏桌面图标」开关，切的就是这个组件的 enabled 状态。

- 隐藏时挂一条**常驻通知**当回家的路（点按回设置页），并申请 `POST_NOTIFICATIONS`；
- 手动兜底：`adb shell am start -n io.github.ldxm666.mapadkiller/.MainActivity`；
- 在 LSPosed 管理器里把模块停用不受影响，桌面图标状态存在系统组件表里。

真机实证：`disabledComponents: io.github.ldxm666.mapadkiller.LauncherAlias`，
`resolve-activity -c LAUNCHER` 返回 `No activity found`，通知正常常驻。

### 新增：「收藏夹」独立开关

宫格第 3 行是**轮播格**（景点游玩 / 离线地图 / 通行费助手 / 收藏夹 / 更多工具 …
在几个槽位里换着显示）。「收藏夹」以前被并进「更多工具」的别名，
受「扩展工具页」和「更多工具」双重夹击，不管怎么设都是隐藏的。

- 从 `TOOL_EXTRA_LABELS` 里摘出来，`TOOL_ALIAS` 归一成自己的键；
- 留不留，只看**这一格此刻显示的那个文案自己的开关**，不再看 `cellKey`
  （那是首见文案钉住的键，轮播格钉错了就永远放不出来）；
- 「保过一次就永久保」：轮播轮到收藏夹时会把格子放出来，并把它从隐藏登记表里摘掉；
- 收行判据改成**配置驱动**（`markKeep`）并支持反悔：
  以前按当时的 width/height 判空行，AJX 首帧还没量过时整行会被误判成空行收掉，
  而 `squashedRows` 每轮重申 GONE+height=0 —— 那一行就永久死了。

### 新增：隐藏搜索页金刚区（美食 / 酒店 / 加油站 / 休闲玩乐 / 扫街榜）

搜索页顶部那排分页运营位，设置页新增「高德 · 搜索页」分区独立开关（默认显示）。

识别方式是**结构投票**：只有当某个容器子树里同时挂着 >=4 个分类文案时才认它，
命中从上往下第一个满足的祖先（最小那个），再往上走几层把分页小圆点一起收掉。
单看一个「美食」绝不动手 —— 首页信息流、我的页、宫格里都有它。
这条规则不属于首页，所以跑在页面闸门**之前**。
真机实证：`SEARCHCAT-HIDE Container [31,13 1017x223]`，金刚区与圆点一起消失，
搜索框 → 快捷 chip → 历史记录 之间不留空带。

### 修复：百度地图卡在开屏，要按返回键才进主页（严重）

**根因**：广告 SDK 自动检索把 `com.meishu.sdk.core.ad.splash.SplashAdLoader#loadAd`
一起吞了（不 proceed、返回默认值）。百度开屏正是等这条回调往下走的，
回调永远不来 → 开屏流程永远收不了尾，屏幕还留一层不消失的暗色蒙层。

**修法**：凡是开屏链路（类名或方法名含 `splash` / `kaiping`）的加载方法一律**不吞**，
原方法照常执行。开屏广告不需要在这一层拦 —— 视图层已经用 alpha=0 兜底，
零曝光且完全不影响开屏自己的收尾节奏。

### 修复：百度开屏广告被 GONE 掉同样会掐死收尾

视图层原来对开屏广告根做 `setVisibility(GONE)`。真机实证：GONE 之后
「跳过 5s」倒计时与收尾回调一起死，同样表现为卡开屏。
改成 **`setAlpha(0f)`** —— 视图照常 measure/layout/跑动画/收回调，只是看不见。
真机实证：日志从 `hidden` 变 `hidden(alpha0)`，倒计时 5s→4s 正常走，
百度冷启动 1 秒内进入 MapsActivity，无蒙层、无广告曝光。

### 修复：首页「去幸福路步行街」快捷卡被压成半张（用户报「半遮住、很难看」）

**根因**：那张卡里也有一个写着「打车」的按钮，`ascendSmallCell` 照样爬到了它的容器
（161x95，父容器 1006x210），于是它被当成工具宫格的一格；`packRow` 随后
按工具格尺寸（158 宽）把整张卡重排 —— 卡片文字被压成 84px 宽，
「去幸福路步行街」只剩「去幸」两个字。

真机日志证据：`TOOL-HIDE 打车 [801,57 161x95]` + `TOOL-REPACK [0,0 1006x210] cells=4`。

**修法**：两道形状闸门。工具格实测 158x147 / 158x164，所以

- `applyTools` 里先问 `looksLikeToolCell()`：宽 120~220 且高 120~200 才算宫格格子；
- `packRow()` 入口再加一道同样的闸门，任何被误判的容器都不会被重排。

真机实证：`TOOL-HIDE 打车 [636,0 158x164]`（真格子），
卡片文案恢复 `w=617`（与停用模块时的基线完全一致），`TOOL-REPACK` 不再误伤。

### 修复：「设置家 / 设置单位 / 常去地点」不再连带整张卡

这一排和「去XX」快捷卡常常在**同一个 AJX item** 里，原来走 `collapseHost`
会把整张 item 一起收掉。现在 `hideChipsRowOnly()` 只摘那一排
（最小、且子树里挂着 ≥2 个 chips 文案的祖先），真机日志 `CHIPS-HIDE`。

### 修复：「更多工具」页白屏 + 收藏夹看不见（严重）

两个症状同一个根因：**搜索页金刚区的规则跑到「更多工具」页上去了**。

`applySearchCats()` 按设计跑在首页闸门**之前**（搜索页没有底部标签栏），
而「更多工具」页的「服务」区里全是同一批分类词 —— 美食 / 洗车养车 / 洗牙 /
休闲玩乐 / 订酒店… 于是整页被当成金刚区收掉，用户点进去就是**一片白**。

同一个根因还让「收藏夹」看着像没生效：那一页的入口全被收没了，
开关开不开当然都看不见（开关本身一直是好的）。

另外标签级即时隐藏（消闪烁那段）以前**不分页面**：只要文案命中就盖掉，
于是「更多工具」页里的「代驾 / 旅游度假 / 景点游玩…」变成"有图标没字"，
就是用户说的「偶尔跳出其他工具出来」。

**修法**（两道）：

- 金刚区规则加**结构护栏**：命中的容器祖先链上必须有 `HorizontalScroller`
  —— 搜索页金刚区是分页横向滚动条里的一排，「更多工具」页的分类宫格不是；
- 标签级即时隐藏加**归属护栏**：工具键必须真的是宫格格子（`looksLikeToolCell`），
  规则键必须真的在首页（`homeCtx`）。

真机实证（A/B 对照）：

```
停用模块：更多工具页 30 条文案正常渲染，「最近使用」里 收藏夹 / 驾车 / 租车 都在
启用模块：同样 30 条文案，收藏夹在，白屏消失
模块动作全部落在首页（TOOL-HIDE 几何全是 158x147 / 158x164）
搜索页：SEARCHCAT-HIDE Container [31,13 1017x223] 照常命中
```

### 新增：首页运营推广卡槽位开关（「去XX」/ AI叫车 / 顺风车…）

首页底部那个槽位是高德的**轮播推广位**，标题一天能换好几张。实测同一台机器上
连着几十分钟就换了五张：

```
去幸福路步行街    / 有座不拥挤 · 行程有保障     / 打车
帮我预约车辆      / 通勤高峰担心拥堵、叫不到车 / AI叫车
帮我叫一辆空气清新的车 / 怕叫到臭车？试试个性化叫车 / 去打车
预约顺风车，一口价超便宜 / …                 / 去预约
顺路搭模式上线了  / 拼座7折起 · 乘客邀请直接拼入 / 去预约
```

盯文案永远追不上，所以最后落到**结构**上。真机 uiautomator 实测这五张卡版式完全一致：

```
标题  [185,2093] w≈580~617 h=55
副标  [185,2161] w=150~179 h=39   （两个 chip）
按钮  [845,2130] w=112 h=47       （右侧药丸）
```

判据：**整屏宽（≥90%）+ 高 180~320 + 一个 ≥5 字的标题 + 一个 2~7 字的短按钮**，
并且这个节点本身就是列表 item（父级是 AjxList2）。

三条护栏：

- 宫格 item 是 1080x508，高度直接出界；
- 「我的」页那些整屏宽的行里必有 MY_* 文案，一律排除；
- 天气 / 限行卡形状也撞得上，按「°C / 天气 / 限行 / 气温 / 降雨」排除。

两个实现要点（都踩过）：

1. 扫文案时**直接读视图**（AJX 的 Label 把文字挂在 contentDescription 上），
   不能查 anchors 索引 —— 索引里只有"被规则认领过"的文案，新面孔永远等不到；
2. 绑定那一刻 item 还是 0x0，判不了几何 —— 所以除了 bind 钩子，
   每一轮 applyPass 在布局稳定后还会再找一遍（`PROMO-HIDE`）。

真机实证：`BIND-HIDE home_quick_card [0,0 0x0]` —— 命中的正是**没写进任何文案表**的
「顺路搭模式上线了 / 去预约」那一版。

### 新增：「去XX」快捷打车卡单独开关

首页底部那张「去幸福路步行街 / 有座不拥挤 / 行程有保障」的智能目的地卡，
设置页新增独立开关（默认显示）。

判定用的是「标题形状 + 旁证」两条：标题以「去」开头、长度 2~16、不是工具键也不是
chips 键；**并且**往上 8 层内能找到挂着「有座不拥挤 / 行程有保障 / 打车…」的小块。
只有形状命中就乐观登记，旁证由每一轮 applyPass 再判 —— 因为标题 setText 的时候，
旁边那行小字往往还没写进去，当场判定必然落空。

真机实证：`BIND-HIDE home_quick_card [37,507 1006x242]`，卡片整条消失，
页面收得干净，不留空带。

### 增强：常驻重申心跳

resume 时只排 6 轮 APPLY（最晚 4.2s），之后 applyPass 停手；而 AJX 在滚动 /
重渲染时会把被压掉的 item 复活。现在加了一条只跑 `reassert()` 的 300ms 心跳，
并在 `onBindViewHolder` 里对已判定要收的 item 立刻重新按回去。

### 修复：隐藏不再写 `lp.height = 0`

AJX 重绑时会 `setVisibility(VISIBLE)` 但保留我们写的 `height=0`，
那张卡就变成"半张、文字被切"的残片。改成**只 GONE**：GONE 的节点根本不会绘制，
从机制上不可能产生残片。

### 其它

- 页面闸门同时让「我的」页之外的误伤面归零；
- `versionCode 7 → 8`。

## v1.0.6 (2026-09-13)

### 新增：广告 SDK 自动检索 + 学习 + 持久化

不再只靠内置的厂商前缀表 —— 内置表永远只能找到**已经写进去的**厂商。现在：

- **自动发现**：扫描目标 App **自身 APK 里的 dex 字符串表**（纯 Java 解析 string_ids，
  只读短字符串、不遍历全文件，无 DexKit / 无 native 依赖），既匹配内置前缀，
  也按**广告特征词**找出可疑类，再从类名反推厂商根包名；
- **安全闸门**（重要）：广告特征词必须落在**类名前 4 段命名空间**内 + 基础设施根黑名单。
  否则会误学 —— 实测高德曾被学出 `com.alipay` / `com.google` / `com.huawei` /
  `com.taobao` / `com.alibaba` 等 15 个**非广告**包，真拦下去会把登录、支付、地图本体一起打死；
- **持久化**：hook 侧拿到的 RemotePreferences 是**只读实现**
  （`UnsupportedOperationException: Read only implementation`），
  且超时/广播跨进程唤起在 HyperOS 上会被拦 → 改用 **ContentProvider** 通道
  （`ContentResolver.call()` 按需唤起设置 App 进程）把学到的东西交给 App 侧落盘，
  并加了一层本地 prefs 暂存兜底；
- **下次直接拦**：学到的厂商前缀跨进程保存，三家地图下次启动读到即生效。

设置页新增「已捕获广告 SDK：N 个」一行，点开可看清单、可一键清空重新学习。

### 修复：框架类被全局 hook 导致百度闪退 / 腾讯 ANR（严重）

SDK 自动 hook 最初沿**父类链**往上走，从 SDK 类一路升到 `android.view.View` /
`android.app.Dialog` / `android.content.ContextWrapper` 并挂上了钩子：

```
SDK BLOCK android.app.Dialog#show          ← 全局 Dialog 被拦
SDK CTX   android.app.Activity#startActivityIfNeeded
SDK CTX   android.content.ContextWrapper#startService
→ ANR in com.tencent.map / 百度地图闪退
```

这与 v1.0.2 白屏事故同源，是本模块第一红线。已改为**只处理声明在 SDK 类自己身上**的
方法，并加框架包硬闸门（`android.` / `java.` / `dalvik.` / `androidx.` / `kotlin` 一律不动）。
实测三家 App `FATAL=0 / ANR=0`，框架类 hook 残留 0。

### 修复：状态显示与实际不符

原来用 `StatusCheck.amEnabled()` 自检 —— 它依赖"模块被注入自己的进程"，
实测 LSPosed 不会这么做，于是**明明已激活却显示未激活**。
改为两个可验证的真实信号，并配红/绿点：

```
● 已激活 · LSPosed 2.1.1                 （LSPosed 服务可达）
● 作用域已勾选 · 高德 / 百度 / 腾讯（3/3） （向服务查询 getScope()）
```

### 修复：设置 App 启动即崩

```
NoClassDefFoundError: io.github.libxposed.api.XposedInterface$Hooker
  at Config.prefs → SdkAutoBlock.loadLearned → MainActivity.sdkSummary
```

`io.github.libxposed.api.*` 是**只编译不打包**的 stub，App 进程里没有这个包。
已加 `hookSide` 闸门：App 侧一律不触碰 `H` / `Config.prefs()`。

### 优化：百度开屏广告隐藏提速 3 倍

原来只有等"跳过"文字出现才认得出广告，广告先亮 1 秒。补上实测命中的 token
（`qumeng` / `advlib` / `splashcountdown`）并加 onPreDraw 探针：

```
旧: BMAP splash ad hidden t=1000ms hit=skip-btn:...SplashCountdownView
新: BMAP splash ad hidden at=350ms hit=qumeng:com.qumeng.advlib...
```

### 优化：高德首页内容流

信息流卡片改为挂列表适配器的 `onBindViewHolder`，在**绑定完成的同一帧内**判定并隐藏；
并补挂 onPreDraw 复查 —— 新帖的文字往往在 `onBindViewHolder` 返回后才写入，
只在绑定时刻判会漏。实测滚动过程中 100+ 次 `BIND-HIDE`，几何全为 `0x0`（从未被绘制）。

### 其他

- 品牌统一：设置页标题 MapAdKiller，分类标题标明作用域（`高德 · 首页工具宫格` 等）
- 修复「已捕获广告 SDK」计数不刷新（`onCreate` 时的空快照 vs 点开时的实时值不一致）
- 替换模块图标

## v1.0.5 (2026-09-11)

### 新增：高德首页 / 「我的」页 UI 自定义（并入原 AmapEnhancer 引擎）

把独立模块 AmapEnhancer 的整套 UI 引擎合并进本模块（包名仍是
`io.github.ldxm666.mapadkiller`，百度 / 腾讯 hook 原样保留）。模块 App 现在带有设置页，
改动写入 LSPosed RemotePreferences，Hook 侧同名 group 读取，写入即生效。

可配置项：
- **主页标签栏**：逐 tab 显示/隐藏（首页 / 探索 / 长按说话 / 打车 / 我的），
  剩余 tab 自动均分重排，不变形
- **首页工具宫格**：10 个工具逐格显示/隐藏，剩余格子行内左对齐补空；
  另加「扩展工具页」开关（景点游玩 / 离线地图 / 通行费助手 / 收藏夹 / 旅游度假，默认隐藏）
- **首页推荐内容**：天气卡 / 周边景区 / 榜单帖 / 距离卡（公里与**米**）/ 精选榜单 /
  攻略内容流 / 问问 AI / 推荐频道栏 / 设置家等 chips
- **「我的」页**：订单栏 / 车辆服务栏 / 达人任务 / 运营卡栏 / 猜你喜欢 / 资质信息与协议中心

### 关键修复：一条推荐区规则把**整个工具宫格**抹掉

旧引擎让「文本锚点」直接决定「收缩到哪个祖先」。宫格第 3 行轮播出的 `景点游玩`
命中了推荐区锚点里的「景点」子串 → 被当成 `feed_scenic` → 沿父链爬到 AJX 列表 item
——而那个 item **就是整个工具宫格** → GONE + 高度归零。整排工具因此消失。

修复：**先判定归属面，再执行动作**；工具格/行/宫格登记为受保护节点，
任何「收缩 item」的爬升路径一旦穿过受保护节点立即中止。

### 关键修复：排版重叠错乱

曾尝试跨行把 `更多工具`（宫格第 3 行）搬进第 1 行的空槽，
实测会被 AJX 自有模型的下一次布局覆盖，两套几何叠在一起 → 标签重叠。
现已退回安全实现：**只做行内左对齐压实 + 空行压 0 高，绝不改格子的父节点**。

### 根治：信息流卡片改为「绑定即隐藏」，不再渲染后再删

实测 AJX 列表适配器 `...ajx3.widget.view.list.a` 是混淆的，字段只有
`IAjxContext / zr / nv0` —— **native 侧不存在条目数据列表**，数据在 JS 引擎里，
Java 层没有可直接过滤的数据结构。因此改为挂载该适配器的 `onBindViewHolder`，
在**条目绑定完成的瞬间**（仍在本次 layout pass 内、这一帧还没绘制）判定并隐藏：

- 真机实测 29 次 `BIND-HIDE`，几何全是 `0x0` —— 卡片**一次都没有被绘制出来**
- 不再依赖「下一帧再扫全树」，也不再 600ms 常驻轮询

### 其他修复

- **配置不保存**：`edit().apply()` 是异步落盘，切完开关立刻退出设置页会丢写入 → 改 `commit()`
- **设置页开关显示错**：LSPosed 服务是异步绑定的，绑定前 `readState()` 只能回退默认 true，
  重开设置页会看到「全开」→ 现在绑定成功后按真实存储重刷，绑定前开关禁用
- **频道栏摘不掉**：它的文案既不走 `Label.setText(String)` 也不走 `Label.setAttribute("text",…)`，
  文本锚点完全失效 → 改为**结构识别**（HorizontalScroller + ≥3 个高 60~170、宽 ≤250 的 chip；
  注意要忽略 53x11 的下划线指示条，否则永远匹配不到）
- **锚点丢事件**：`车辆服务` 这类锚点在 setText 时其 HorizontalScroller 尚未 attach，
  旧实现重试 3.5 秒后放弃 → 改为常驻 WeakHashMap 索引，每轮重试到挂载为止
- 工具名与实际文案不一致：`火车票机票`→`火车票`、`高德扫街榜`→`高德扫街`（别名归一）
- 我的页条目带装饰字符（实际是 `-资质信息 ` / `协议中心-`）导致精确匹配永远失败 → 归一后匹配
- 距离卡只认「公里」，漏掉同城卡的「597米」→ 补上米

### 性能

- 每个锚点不再各自 `postDelayed(500ms)` + `150ms×20` 重试，隐藏/重排**合并到下一帧执行一次**
- 配置改为每轮 `getAll()` 快照一次，锚点判定读内存（原先每次判定都走一次 RemotePreferences IPC）
- 去掉 600ms 常驻全树轮询，改为 resume 后按 100/350/800/1500/2600/4200ms 收敛几轮

### 其他

- 替换模块图标
- `module.prop` 补齐收录所需标准字段（name / version / versionCode / author / description）

## v1.0.4 (2026-09-10)

- 重新发布：修复 v1.0.3 发布说明在收录站点的编码乱码（发布流程按非 UTF-8 编码传输请求体所致）
- 代码与 v1.0.3 完全一致，无功能变更；已装 v1.0.3 用户可自愿更新

## v1.0.3 (2026-09-10)

### 关键修复：API 102 Hooker 接口方法名错误导致全部 Hook 失效

- **症状**：v1.0.2 起模块能正常加载、Hook 显示"安装成功"，但运行时每次调用抛
  `AbstractMethodError: XposedInterface$Hooker.intercept(XposedInterface$Chain)`，
  被 `exceptionMode=protective` 静默吞掉 → 所有地图去广告全部无效（高德 16.x/17.x、
  Android 17 设备均有用户反馈；实际影响所有 Android 版本）
- **根因**：编译期 stub 中 `XposedInterface.Hooker` 的抽象方法被误写为 `hook(Chain)`，
  真实 libxposed API 102 为 `intercept(Chain)`。匿名类实现可正常通过编译，
  框架运行时按真实接口回调 `intercept()` 找不到实现 → AbstractMethodError
- **修复**：stub 与全部 8 处匿名 Hooker 实现（H.VOID/H.FALSE/H.TRUE、
  MainHook 自检、AmapHooks 开屏闸门、BmapHooks 遮罩摘除/开屏容器、Sweeper）统一改为
  `intercept(Chain)`；与上游 `io.github.libxposed:api:102` 逐签名核对
- 顺带：`hookAll` 跳过 abstract 方法（此前对抽象方法尝试 hook 必然
  `IllegalArgumentException: Cannot hook abstract methods` 并刷 hook_error 日志）
- 真机回归：高德 hook 命中恢复、日志无 AbstractMethodError

### 百度地图启动白屏/卡开屏修复（本批真机回归发现）

- 背景：v1.0.2 的全部 Hooker 因 AbstractMethodError **从未实际执行**，其中
  "开屏根治"等新增改动未经真机验证；v1.0.3 修复 Hooker 方法名后整套 Hook
  首次真实运行，在 21.20.30 上暴露两类启动问题：
  白屏（单次启动 30+ 个 `SplashViewContainer` 被盲杀）与
  卡品牌开屏（`SplashAdManager.G/H/n/m`、`SplashAdProvider.k/m` 吞调用
  拦截开屏必经路径，完成事件永不到达）
- 处置：
  - 删除 `SplashViewContainer` 移除 Hook 及 ViewKiller 类名盲杀（多实例复用容器）
  - 删除 `HomeSplashPresenter.n` 遮罩摘除
  - 放开 `SplashAdManager.G/H/n/m`、`SplashAdProvider.k/m` 的 VOID 吞调用；
    恢复 `HomeMidBannerPresenter.onCreateView`（返回 View，VOID 会产生 null）
  - 保留 F/z 布尔闸门（FALSE=不等广告快速进首页）+ ADN Loader + BMAd 数据层拦截
  - 真机 21.20.30 回归：约 2s 进首页，无白屏/卡屏，首屏无广告
- 观测增强：`hookMethod` 增加首火日志（每个 Hook 首次触发记录 `HIT <id>`），
  便于用户反馈时定位"Hook 已装但未触发"类问题

### 百度 21.20.50（versionCode 1645）适配

- 静态分析（apktool smali）确认：`SplashAdManager` Kotlin 重写后 `F()/z()/G()/H()`
  仍在，但自营运营开屏（`fetchBizSplashAd`，如淘宝闪购）**不再经过 F/z 闸门**，
  数据层拦截对其失效；该 App 打包 ADN SDK：优量汇（com.qq.e）、穿山甲
  （com.bytedance.sdk.openadsdk）、sigmob、kwai
- 新增 `SplashViewContainer.addView` 探测 Hooker：该容器为品牌/广告复用 FrameLayout
  （21.20.50 已不覆写 `onAttachedToWindow`——v1.0.2 实际误 Hook 了 `android.view.View`
  的全局 attach，即白屏元凶）；新增子树命中打包 AD SDK 类名特征或「跳过」按钮时
  整个子树 GONE，广告倒计时由 Handler 驱动照常完成（onAdFinish/onSkip 正常回调），
  品牌层不受影响 → 不白屏、不卡开屏、广告零曝光
- 真机 21.20.50 回归：启动约 1-2s 进首页，无白屏/卡屏，首页信息流/中部横幅/黄条无广告

## v1.0.2 (2026-09-08)

### 框架重构：legacy → libxposed API 102
- 入口从 `IXposedHookLoadPackage`/`assets/xposed_init` 迁移到 `io.github.libxposed.api.XposedModule`
- 元数据改用 `META-INF/xposed/{java_init.list, module.prop, scope.list}`（`staticScope=true`，安装即固定作用域）
- 移除全部 `de.robv.*` 依赖与 `xposedmodule` meta-data；Hook 链使用 `hook(Executable).setId().intercept(Chain)`
- 编译期 stub 仅 `io.github.libxposed.api`（compileOnly，不打包）；实测 `api=102 framework=LSPosed`

### 三 App 同步至最新版
- 高德 17.00：开屏数据闸门 `za6`→`u96`（双名兼容尝试），其余 hook 点名称稳定，14/14 命中
- 百度 21.20.30：`SplashAdManager` F/z/G/H/n/m 字母闸未漂移；`YellowBannerPresenter` 新方法集（`showViewSwitcher`/`showYbBannerAnim`/`onResumeForYB`）；35 命中
- 腾讯 11.5.0：GDT + 开屏流水线 + 首页绑定全部稳定，8/8 命中

### 百度开屏广告根治
- 真机发现：缓存 SDK 广告（淘宝闪购等）经 `com.baidu.baidumaps.splash.view.SplashViewContainer`
  直接 `addContentView` 展示，绕过 `SplashAdManager.G/n` 加载闸
- 修复：`SplashViewContainer.onAttachedToWindow` 挂载即 `removeView`（即时、覆盖 brand+SDK 两条路径），
  另保留 `HomeSplashPresenter.n` 摘除 + ViewKiller 类名兜底三保险
- 开屏卡需按返回键问题一并终结（容器不再存在 → 无等待）

### App 内广告扩展
- tmap：POI 列表 `AdCardView`×2、`NoticeBanner`、POI 详情 `EtcpBannerCardView`、
  路线 `CarRouteSubPoiBannerView`、地图浮层推广气泡 `ExBannerWidget`/`ExMutableBannerView`
- ViewKiller 新增**无障碍"广告"角标识别**：contentDescription/text 为"广告"/"Ad"的小角标
  自动上溯到列表项容器整体移除（bubbleToAdContainer）
- Sweeper 增加 3500ms 第三次清扫（懒加载信息流广告）

## v1.0.1 (2026-09-03)
- 包名迁移 `cn.eni.mapadkiller` → `io.github.ldxm666.mapadkiller`（Xposed 模块仓库自动审批命名空间）

## v1.0.0 (2026-09-03)

Initial release. Verified on KernelSU + ZygiskNext + LSPosed v2.1.1 (Android 16).

- Amap 16.23: splash / real-time fetch / home banner / background push / search template splash — 14 hook points
- Baidu Maps 21.18: splash (3 channels) / mediation loaders ×6 / BMAd providers / mid banner / floating promo bar — 34 hook points, fast entry via `F()/z()` gate
- Tencent Maps 11.4: GDT init kill / splash pipeline tasks ×3 / home banner binding / POI ad cards (view layer) — 8 hook points + ViewKiller
