# Changelog

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
