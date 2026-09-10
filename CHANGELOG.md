# Changelog

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
- 真机回归：高德 hook 命中恢复、日志无 AbstractMethodError

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
