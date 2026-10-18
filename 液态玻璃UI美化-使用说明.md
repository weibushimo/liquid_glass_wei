# 液态玻璃UI美化 · 使用说明
(本md由Deepseek V4.1Flash1017生成，人工辅助修改)
一个 **LSPosed / Xposed 模块**，给 B 站客户端（手机版与 HD 版）的底部 dock、顶栏等位置绘制 macOS 风格的液态玻璃（折射 + 高光 + 水滴滑块）。

- 包名：`io.github.liuran001.mmliquidglass`
- 版本：`0.3.0-netease-dock`（versionCode 21）
- 签名：debug key（SHA-256 `6779781c…58d`）
- APK：`液态玻璃UI美化.apk`（与本说明同目录）

---

## 一、必须配合的东西（缺一不可）

| 依赖 | 要求 | 说明 |
|---|---|---|
| **Xposed 框架** | LSPosed / Vector 等，**Xposed API 102** | 模块的 `module.prop` 声明 `minApiVersion=102`、`targetApiVersion=102`。低于 102 的旧框架装不了 |
| **作用域** | `tv.danmaku.bili`（手机版）或 `tv.danmaku.bilibilihd`（HD 版） | 必须**手动勾选**作用域，否则模块不会注入 |
| **Android 版本** | **13（API 33）及以上**才有真玻璃 | ≥33 用 `RuntimeShader` 做折射；**12（API 31–32）**只有基础面层、没有折射；<31 不工作 |
| **B 站客户端** | 需要 dock 栏结构未大改的版本 | 见第四节「版本敏感点」 |(测试使用v9.15.0(HD为2.5.1))
| **模块内开关** | 顶栏玻璃默认关闭，需 `setprop` 打开 | 见第三节 |

> 签名一致才能覆盖安装。若你之前装过别的来源版本，需先卸载（会丢失模块设置，不影响 App 数据）。

---

## 二、安装步骤

1. 安装 `液态玻璃UI美化.apk`（普通安装即可）。
2. 打开 **LSPosed 管理器 → 模块**，启用「液态玻璃UI美化」。
3. 进入该模块的 **作用域**，勾选 **`哔哩哔哩`**（`tv.danmaku.bili`）；若要 HD 版则勾 `哔哩哔哩HD`。
4. **强制停止** B 站（或重启手机），再打开 B 站。
5. 生效后底部 dock 会出现玻璃胶囊 + 水滴滑块。

验证方法（需 root / adb）：

```
adb logcat | grep LiquidGlass/BiliPhone
```

看到 `glass attached: ... capsule=...` 即说明注入成功。

---

## 三、可调开关（`setprop`，重启失效）

> ⚠️ **清空属性必须用 `null`，不能写 `""`** —— 在部分 ColorOS/ROM 上 `setprop x ""` 会报 usage 失败且**不改变原值**。
> ⚠️ 属性**不持久**，重启设备后回到默认值；改完需**重启 App**（不是重启手机）才生效。

### 常用

| 属性 | 默认 | 作用 |
|---|---|---|
| `debug.biliphone.topglass` | 关 | 顶栏液态玻璃面层。**只有设 `=1` 才建** |
| `debug.biliphone.topglassfade` | `2` | 顶栏顶部黑→透明渐隐高度（dp），`0` 关闭 |
| `debug.biliphone.topglassalpha` | `33` | 玻璃表面涂层浓度（hex 字节，越小越透） |
| `debug.biliphone.topglassrim` | `8` | 顶栏描边边掩码：`1`上 `2`左 `4`右 `8`下，`0`全关，`f`整圈 |
| `debug.biliphone.topbarplate` | 开 | 清除顶栏纯色底；`=0` 恢复原样 |
| `debug.biliphone.paperstretch` | 开 | content 层随滚动上移；`=0` 完全不干预 |

### 调试 / 对比

| 属性 | 作用 |
|---|---|
| `debug.biliphone.step` | 分步隔离，只跑某一步（排查时用） |
| `debug.biliphone.fullwidth` | dock 玻璃用整条背景（A/B 对比） |
| `debug.biliphone.composelift` | Compose 子页（热门/动画等）的滚动驱动；默认路径会带来频闪 |
| `debug.biliphone.homeliftgain` / `followease` / `followheight` / `followlift` / `followsnap` / `followstep` / `followtopglass` | 首页 lift 的增益与缓动等细调 |
| `debug.biliphone.hlcolor` | 高亮块颜色覆盖 |

示例：

```bash
adb shell setprop debug.biliphone.topglass 1
adb shell setprop debug.biliphone.topglassfade 2
adb shell am force-stop tv.danmaku.bili
adb shell am start -n tv.danmaku.bili/.MainActivityV2
```

---

## 四、版本敏感点（升级 B 站后可能失效）

模块依赖的是宿主**未混淆的类名与 id**，B 站改版可能让某一部分失效：

| 依赖点 | 用途 |
|---|---|
| `tv.danmaku.bili.khome.widget.bottomtab.*` | dock 栏容器与 tab 列表 |
| `tv.danmaku.bili.khome.widget.topbar.*` | 顶栏纯色底的绘制调用（要清掉的是 `#17181a`） |
| `id/content`、`id/pager` | 页面层与翻页容器 |
| `PegasusTintRecyclerView`（`id/recycler_view`） | 首页信息流，玻璃的折射来源 |
| `androidx.compose.foundation.BackgroundKt#background-*` | 纯色底拦截点 |

**症状与对策**：

| 现象 | 多半是 |
|---|---|
| 完全没反应 | 作用域没勾 / 框架 API 版本不够 / App 被强制停止过但没重启 |
| dock 有玻璃但滑块不跟手 | dock 结构或 tab 列表类名变了 |
| 顶栏还是纯色 | 顶栏绘制调用变了（`topbar.m` 那一路），或 `topbarplate=0` |
| 玻璃一片发灰 | 顶栏底下没有内容可折射 —— 检查 `paperstretch` 与顶栏高度是否变化 |
| 有玻璃但完全没有折射 | 系统低于 Android 13，`RuntimeShader` 不可用（只有平面面层） |

---

## 五、已知问题
1. **浅色 / 深色热切换不跟随** —— 切换主题后玻璃仍用旧配色，**重启 App** 即可恢复。
   根因已定位（宿主的 `onConfigurationChanged` 拿不到 Activity、视图侧 `Configuration` 不更新），
   修复方案未采纳。详见工程文档 `BILI_PHONE_ALL.md` 的「已知问题」一节。
2. **切换主题后冷启动，首页 tab 的浅色块可能跑到左下角** —— 点按一次 dock 栏即恢复。
   尚未定位。
3. **慢速拖动偶发频闪** —— Compose 子页的滚动驱动（`composelift`）与"原生滚动 + 平移"两套位移有关，
   默认已规避；`composelift` 打开时更明显。

人工记录bug:
1.旋转屏幕会导致滑块失效（请清理进程后在对应横竖屏模式冷启动应用）
2.会员购界面下滑浏览一段距离后底部会出现纯色块
3.'我的'界面里启用'创作中心/我的服务/更多服务'相关子服务后会出现部分闪屏现象
4.滑块滑经'关注'时不会显示具体红标数字
5.热切换浅色/深色模式不会使液态玻璃切换到对应状态(切换完模式后重新启动应用解决)同时在这之后冷启动应用，首页Tab键的浅色背景块会跑到左下角(点按一次dock栏即可解决)
6.dock栏左右两侧虽然没有显示dock内容，但若点击两侧就会直接让滑块移动到最左边或者最右边
7.在首页滑动content时若content未到达顶部则在滑动没完全禁止之前content会持续上下小幅度跳动（已解决）
8.竖屏模式下首页content在下滑后会有一部分不显示（content值在过程中只加大到1670，没有到竖屏全屏的标准）
9.在浅色模式下首页顶部会有一块黑色（为原顶部背景色设为透明后带来的后果）
10.在首页时将顶部'推荐'tab切换到别的tab后会发现别的tab没做和首页一样的适配，再切换回'推荐'tab会使原作用于'推荐'的content顶部处理方案也失效
11.未对首页-直播/S16/新征程做适配
---

## 六、卸载 / 回滚

1. LSPosed 里关闭模块并重启 App —— 立即恢复原样（不改动 App 本身）。
2. 或直接卸载模块 APK。

模块**不修改 B 站 APK**，只做运行时注入，卸载后无残留。

---

## 七、附：本说明对应的验证环境

```
设备    OPD2101（ColorOS，Android 14，2560×1600 横屏，density 331）
宿主    tv.danmaku.bili（手机版）
框架    Vector（Xposed API 102）
模块    0.3.0-netease-dock
开关    topglass=1、topglassfade=2
```

补充:本模块暂时只适配oppo pad横屏模式，未适配竖屏，若出现content铺不满页面请自行修改content值