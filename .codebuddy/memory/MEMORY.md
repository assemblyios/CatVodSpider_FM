# CatVodSpider 长期记忆（跨会话）

> 最后更新：2026-07-16。本文件沉淀稳定、跨会话有用的项目/生态/用户偏好事实。

## 一、项目概况（CatVodSpider）
- 本质：CatVodTVSpider 派生版，jar 派**爬虫(站点解析)开发与调试沙盒**，产物 `custom_spider.jar` 供 FongMi/TV 等 CatVod 系播放器加载。
- 目录：`app/`(Android 工程, `app/src/main/java/com/github/catvod/spider/` 各站点解析器 + `crawler/Spider.java` 契约 + `MainActivity.java` 调试界面)；`jar/`(`genJar.bat`,`checkJar.ps1`,`spider.jar/` 模板, `custom_spider.jar`)；`json/config.json`(sites 用 `csp_类名` 关联)。
- 调试：Android Studio Run app，`MainActivity` 硬编码 `spider = new PTT();` 为调试目标；按钮触发各方法看 JSON + Logcat。
- 打包(Win `build.bat`)：gradlew assembleRelease → apktool 抽 smali → custom_spider.jar → checkJar.ps1 校验。
- macOS：`.bat` 不能直接跑；用 `./gradlew assembleRelease` + `brew install apktool` + (pwsh 未装，曾用 Python 复刻 checkJar 逻辑校验)。`checkJar.ps1` 限制 jar 仅含 `com/github/catvod/{js,spider}`、白名单包、禁 Gson TypeToken 直接构造。

## 二、两套 CatVod 爬虫体系不互通
- **jar 派（本项目）**：Java 写 Spider → custom_spider.jar → FongMi/TV 用 `DexClassLoader` + `csp_类名` 反射加载。
- **JS 派（jadehh/TVSpider）**：JS(QuickJS/Node) 写 Spider → 发布 git 分支(js/dist) → TVBox/CatVodOpen 用 `gitee://Token@...` 配置串导入。
- 实现语言、产物、目标播放器、分发均不同，**不互通**。详见 `docs/jadehh-TVSpider说明.md`。

## 三、XBPQ 完整知识（黑盒规则引擎）
- 规则驱动型 spider（`type=3`, `api:"csp_XBPQ"`），约300KB `xbpq.jar`(内含 classes.dex)，**无公开 Java 源码**，仅以 .jar 分发。主类 `com.github.catvod.spider.xBPQ extends Spider`，重写 home/category/player 等。
- **真实语法**（样本 xBPQ1107 菜妮丝线，以 LIBVIO.json 为准）：`&` 列表分隔、`&&` 边界截取、`#` 多关键词、`>>` 替换分隔、`[替换:原>>新]`/`[追加:..]`/`[不包含:a#b]`、`{cateId}{catePg}{area}{by}{year}{class}` URL 模板、`Base64()`、`json数组` JSON 路径模式。
- **混淆**：`merge.BfT.d(hex)` = hex 解码 + 逐字节 XOR 密钥 `"jDLKJK"`。
- **规则样例 LIBVIO.json**（maccms 站）：主页url/分类/分类值/分类url/影片地区/影片类型/导演/主演/简介/线路数组/线路标题/播放数组/筛选。
- **反编译样本**：`boyss180/gao-xBPQ` 的 `xBPQ1107.jar`。
- **smali 资产**：`jar/xbpq_smali/`(577 文件 = xBPQ*.smali + merge/)，来自反编译，曾计划"精缝"进本项目(用户中途喊停，**未完成**)。若重做精缝：从此目录取 smali，需把 `rxhttp/` 与 `javax/annotation/` 加进 checkJar 白名单方能通过校验。
- XBPQ 接入播放器步骤见 `docs/XBPQ规则引擎分析.md` 第七节（jar 须含 csp_XBPQ + 配置加 `type:3`/`api:csp_XBPQ`/`ext` + 推送 FongMi/TV）。

## 四、CatVod 生态事实
- **框架发明者** = 猫影视TV 作者（**化名，开源后称 TVBox**），定了"壳 + 插件式 spider"契约(`csp_` 反射、`custom_spider.jar` 动态加载)。无具名个人。
- **主要分支**：FongMi/TV(台湾开发者, jar派核心, OK影视内核)、影视仓(安卓哥团队, **闭源/公开无源码仓**, 多仓并行)、小苹果等。技术内核一致(壳源分离)。
- **XBPQ 作者**：菜妮丝(cainisi.cf, 主流说)，另有"小米小"来源说；确定黑盒无源码。
- **JS 派**：jadehh/TVSpider。
- **开源源码仓库**（均可 clone 编译）：`github.com/FongMi/TV`(=OK影视内核, 二开换皮)、`github.com/CatVodTVOfficial/TVBoxOSC`(原版空壳)、`o0HalfLife0o/TVBox`、`takagen99/Box`、`q215613905/TVBoxOS`、`NewOrin/TVBox`、`cyao2q/files`。
- **闭源**：影视仓(仅 APK + 接口分发)；OK影视**无独立源码仓**(=FongMi/TV 二开)。
- **关键区分**：源码仓库(可编译代码) vs 配置/接口仓库(`qist/tvbox`、`zmivip/Tvbox` 等只放 JSON 接口, **非代码**)。

## 五、docs/ 文档产出清单（2026-07-16 集中产出）
- `开发文档.md`：项目说明 + 第九节播放器加载机制(DexClassLoader+csp_) + 第十节 XBPQ 源。
- `jadehh-TVSpider说明.md`：JS 派对比。
- `XBPQ规则引擎分析.md`：反编译 + 原理 + **第七节接入播放器**。
- `XBPQ站点选型指南.md`：四门槛 + 体检清单。
- `CatVod源接入实战教程.md`：路线 A(XBPQ 规则)/路线 B(Java/JS) + 索引。
- `CatVod生态发展史.md`：框架/XBPQ 作者 + 各分支开源状态 + 二开源码仓列表。

## 六、用户偏好 / 约定
- **喜欢把成果整理成文档**：多次主动要求"整理成文档"，已沉淀上述 4~5 份 docs（选型/教程/发展史等）。
- **不主动打包**：`debug-log-viewer` 插件记忆(ID:98605220)——改完代码只跑 `npm run compile` 验证，不主动 `vsce package`，等用户确认。
- **XBPQ 精缝任务曾中途喊停**（"算了这个工具，别干了，你都还原吧"），未继续；如需重做从 `jar/xbpq_smali/` 取 smali。
- **用户想学技术/读源码**：多次问发展史、各分支开源仓库、clone 学习；回答时给公开源码仓库地址(合法)，不给影视源接口地址(灰产)。
