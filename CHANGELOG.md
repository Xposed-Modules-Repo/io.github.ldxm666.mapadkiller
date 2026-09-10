# Changelog

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
