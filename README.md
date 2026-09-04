# MapAdKiller

> 高德 / 百度 / 腾讯地图 去广告 Xposed (LSPosed) 模块
> Ad-blocking LSPosed module for Amap / Baidu Maps / Tencent Maps

**中文** | [English](https://github.com/ldxm666/MapAdKiller#english)

---

## 中文

一个纯本地运行的 LSPosed 模块，针对国内三大地图 App 的**开屏广告、首页运营横幅、信息流广告卡、第三方广告 SDK** 做确定性拦截。所有 hook 点均来自对目标 APK 的静态逆向分析（jadx / apktool / androguard），证据表见 [docs/ANALYSIS.md](https://github.com/ldxm666/MapAdKiller/blob/master/docs/ANALYSIS.md)。

### 功能一览

| 目标 App | 包名 | 拦截内容 |
|---|---|---|
| 高德地图 | `com.autonavi.minimap` | 开屏广告（强制走自家 NO_SPLASH 收尾路径，不卡启动）、实时广告拉取、首页轮播 Banner、后台推送运营弹窗、搜索页模板开屏、AJX 首页联动广告数据 |
| 百度地图 | `com.baidu.BaiduMap` | 开屏广告（Native/OpenAPI/Push 三渠道全灭）、聚合层 6 家 ADN 加载器（聚量/GroMore/美数/章鱼/趣盟/推荐位）、悬浮运营黄条（"做任务领现金"类）、首页中部横幅、BMAd 开放封装 |
| 腾讯地图 | `com.tencent.map` | 广点通 GDT SDK 初始化（`initWith=false`，连带 tangramsplash / 融合 SDK 全部失效）、开屏流水线任务、首页 Banner 数据绑定、POI 列表广告卡（视图层） |

另含**通用视图清扫兜底**（ViewKiller）：每次 Activity resume 后遍历视图树，命中已知广告类名/资源名即 GONE + 移除子树。

### 安装

1. 需要 root + LSPosed（本模块在 KernelSU + ZygiskNext + LSPosed v2.1.1 / Android 16 实测通过）
2. 安装 `MapAdKiller.apk`
3. 在 LSPosed Manager 中启用本模块，勾选作用域：**高德地图、百度地图、腾讯地图**（模块 manifest 已声明 `xposedscope`，LSPosed 安装时会自动预选）
4. 重启目标 App 生效（无需重启手机）

### 构建（无需 Android Studio / Gradle）

纯命令行工具链：JDK + Android SDK build-tools（aapt2/d8/apksigner）。

```powershell
# Windows PowerShell
.\app\build.ps1
# 产物: dist/MapAdKiller.apk
```

依赖路径在 `app/build.ps1` (see source repo) 顶部可改（`$Sdk` / `$Bt` / `$Aj` / `$Jdk`）。Xposed API 使用仓库内 `app/stub-src/` 的 **compile-only 桩**（签名与 API 82 一致，运行时由框架提供，不打进 APK）。

### 已知边界

- 广告为服务端概率下发，"某次没广告"不能作为验证依据；请以 logcat `MapAdKiller:` 日志为准（`adb logcat -s LSPosedFramework | grep MapAdKiller`）
- 百度地图首页"爱去榜/旅游攻略"为百度自家内容推荐（非广告 SDK 下发），不在本模块范围
- 高德"探索"tab 内嵌 AJX 画布渲染的内容卡片无法在原生视图层定位，暂未处理
- 打车 tab "单单省"角标等平台促销徽标未处理
- 目标 App 大版本更新可能导致混淆方法名漂移，hook 未命中时日志会输出 `miss` 行，欢迎提 issue 附日志

### 隐私

模块不联网、不采集任何数据；所有逻辑仅修改目标 App 进程内的广告组件行为。

### License

GPL-3.0-or-later

---

## English

A local-only LSPosed module that deterministically blocks splash ads, home banners, feed ad cards and third-party ad SDKs in the three major Chinese map apps. Every hook point was derived from static reverse-engineering (jadx / apktool / androguard) of the target APKs — see [docs/ANALYSIS.md](https://github.com/ldxm666/MapAdKiller/blob/master/docs/ANALYSIS.md).

- **Amap** (`com.autonavi.minimap`): splash (forced into the app's own `NO_SPLASH` path — no launch hang), real-time fetch, home carousel banner, background push ops-popups, search-page template splash
- **Baidu Maps** (`com.baidu.BaiduMap`): splash across all three channels (native/openapi/push), the mediation layer feeding 6 ad networks (Business/GroMore/Meishu/Octopus/Qumeng/Recommend), floating promo yellow bar, mid-banner
- **Tencent Maps** (`com.tencent.map`): GDT SDK init kill (`initWith=false` neutralizes tangram splash + fusion SDK), splash pipeline tasks, home banner binding, POI list ad cards (view layer)

Plus a generic `ViewKiller` sweep on every `Activity.onResume` matching known ad view classes and resource ids.

Requires root + LSPosed (tested on KernelSU + ZygiskNext + LSPosed v2.1.1, Android 16). Build without Android Studio: `.\app\build.ps1` (JDK + SDK build-tools only).

Ads are probabilistically server-delivered — verify via `logcat -s LSPosedFramework | grep MapAdKiller` (`HOOKED` / `KILLED` / `forced NO_SPLASH` lines), not by one ad-free launch.

Licensed under GPL-3.0-or-later.
