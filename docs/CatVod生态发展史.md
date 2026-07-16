# CatVod / XBPQ 生态发展史

> 本文讲清两件事：**(1) 这套「壳 + 可插拔爬虫(Spider)」框架是谁建立的；(2) XBPQ 规则引擎是谁写的**。
> 配套：实战教程见 `CatVod源接入实战教程.md`；XBPQ 引擎原理见 `XBPQ规则引擎分析.md`。

---

## 一、先厘清"发明者"有两层

你玩的这套东西里，"spider"其实分两层，所以"发明者"要分开讲：

- **框架层**：CatVod / TVBox 这套「空壳播放器 + 外部数据源 + 插件式爬虫」的**架构与契约**（`csp_` 反射、`custom_spider.jar` 动态加载、`spider`/`api` 配置约定）。
- **应用层**：**XBPQ 规则引擎**——框架之上一个实现了契约的"规则型 spider"，把"写 Java 爬虫"降级成"写 JSON 规则"。

---

## 二、框架层：猫影视TV → TVBox（"壳源分离"模式）

这套 `csp_` 反射约定、`custom_spider.jar` 动态加载、把解析逻辑做成插件——都源自 **「猫影视TV」(CatVodTV)**。

核心思路：**播放器做"空壳"，解析逻辑(爬虫)做成可动态加载的外部包，靠配置(JSON)里的 `spider`/`api` 字段指向**。

- 猫影视TV 作者后来因故停止维护，把源码**开源**，开源后的名字叫 **TVBox**。
- **关键点**：猫影视/TVBox 的原作者长期以**化名 / 项目名**出现，公开资料里没有公认的个人真名（"项目即作者"的社区形态）。所以严格说"发明 spider 框架的那个人"在公开资料里就是"猫影视TV 的作者"，而非某个具名个人。

---

## 三、框架分化出的主要分支与关键人物

TVBox 开源后迅速裂变出大量分支，**技术内核（壳源分离、spider 插件化）完全一致，只是 UI / 功能 / 源策略不同**：

| 分支 / 项目 | 关键人物 | 特点 |
| --- | --- | --- |
| **FongMi 影视（蜜蜂影视）** | 台湾开发者（GitHub `FongMi`） | UI 优秀；`开发文档.md` 第九节分析的加载机制就来自它；**jar 派**主力播放器 |
| **影视仓** | "安卓哥"开发团队 | 主打"多仓并行"架构，市场占有率最高 |
| **小苹果影视** 等 | 各路魔改者 | 各种 UI / 功能定制 |
| **CatVodSpider（本项目）** | CatVodTVOfficial + 派生维护者 | 官方 spider 开发沙盒的改版，培养 jar 派开发者 |

---

## 四、XBPQ 规则引擎：作者说法不一，但确定是"黑盒"

XBPQ 把"写 Java 爬虫"降级成"写 JSON 规则"（详见 `XBPQ规则引擎分析.md`）。关于它的作者，社区有两种流传说法，如实并列：

- **菜妮丝（cainisi.cf）**：流传最广。反编译样本 `xBPQ1107.jar` 注释写"作者：菜妮丝 https://cainisi.cf"；规则仓库 `boyss180/gao-xBPQ` 也标注 jar 与配置来自菜妮丝。
- **小米小**：有分支仓库（如 `Lin5213/tvbox`）在 XBPQ.json 旁标注"jar 包和配置来源于小米小"。

**可以确定的是**：XBPQ 是**黑盒引擎，无公开 Java 源码**，长期只以 `.jar`（内含 `classes.dex`）形式在圈内分发，主类 `com.github.catvod.spider.xBPQ`。它和"猫影视框架"是**两层东西**——框架定契约，XBPQ 只是框架里一个实现了契约的"规则型 spider"。

### 另一条写源路线：JS 派
- **jadehh / TVSpider**：同期活跃的 JS 派代表，教大家用 JS 写 spider，面向 TVBox / CatVodOpen（详见 `jadehh-TVSpider说明.md`）。

---

## 五、发展时间线轮廓

- **2022 及之前**：猫影视TV 建立"壳源分离"架构；XBPQ 早期版本已存在（手头 `xBPQ1107` 构建于 2022-11-07）。
- **2023**：TVBox 开源后分支爆发（影视仓、FongMi、小苹果等）；JS 派代表 jadehh/TVSpider 同期活跃。
- **2024–2026**：进入"多仓时代"，FongMi / OK 影视等成为主流；写源方式三条路线（jar / js / XBPQ 规则）并存，XBPQ 因"写份 JSON 就能接一个站"成为低成本主流。

---

## 六、一句话总结

> **框架发明者 = 猫影视TV 的作者（化名，开源后称 TVBox）**，他定了"壳 + 插件式 spider"这套契约；**XBPQ 规则引擎作者 = 菜妮丝（主流说法，另有"小米小"来源说）**，是框架之上的一个黑盒规则型 spider。两者是"平台"与"平台上一个热门应用"的关系。

三条写源路线对比（同源不同成本）：

| 路线 | 产物 | 成本 | 给谁用 | 代表 |
| --- | --- | --- | --- | --- |
| jar 源 | `custom_spider.jar` | 高 | FongMi/TV 等 jar 派 | 本项目 CatVodSpider |
| js 源 | JS 配置分支 | 中 | TVBox / CatVodOpen | jadehh/TVSpider |
| XBPQ 规则源 | 一份 JSON 规则 | 低 | 加载了 XBPQ 引擎的播放器 | XBPQ（菜妮丝线） |

---

## 七、去哪学：各分支开源状态与源码仓库

想读源码学习，先分清两类仓库：**源码仓库**（播放器 / 框架代码，可编译、可读逻辑）vs **配置 / 接口仓库**（只放 JSON 接口地址，**不是代码**）。

| 项目 | 开源？ | 源码仓库 | 说明 |
| --- | --- | --- | --- |
| **FongMi/TV（蜂蜜影视）** | ✅ 开源 | `github.com/FongMi/TV` | **OK影视的播放器内核就是它**；jar 派核心，与本项目 `开发文档.md` 第九节加载机制对应，最值得学 |
| **OK影视** | ✅（=FongMi/TV 二开） | 同上 `FongMi/TV` | 网上 `qist/tvbox`、`dahu315/tvbox` 等是**配置仓库**（放接口 json），不是播放器源码。**多次检索未发现独立"OK影视"源码仓——它只是 FongMi/TV 的换皮二开打包，想学其代码直接 clone `FongMi/TV` 即可** |
| **原版 TVBox** | ✅ 开源 | `github.com/CatVodTVOfficial/TVBox`（或 `o0HalfLife0o/TVBox`） | 猫影视开源后的原版空壳 |
| **CatVodTVSpider（本项目上游）** | ✅ 开源 | `github.com/CatVodTVOfficial/CatVodTVSpider` | 官方 spider 开发沙盒，本项目即其派生 |
| **影视仓** | ❌ 公开找不到源码仓库 | — | 通常以闭源 APK + 多仓配置接口分发；想学"多仓 / 壳源分离"架构看 FongMi/TV 或原版 TVBox 即可 |

### 学习路径建议
1. **jar 派爬虫 / 加载机制**（与本项目最对口）：clone `FongMi/TV` + 本项目，对照 `开发文档.md` 第九节。
2. **壳源分离框架本质**：读原版 TVBox / `CatVodTVSpider`。
3. **XBPQ 规则**：看 `XBPQ规则引擎分析.md` + `CatVod源接入实战教程.md`（无需读播放器源码）。

> 影视仓的"多仓并行"仍是壳源分离架构的延伸，在 FongMi/TV、原版 TVBox 的 config 解析里即可看到同样思路，不必死磕其闭源 APK。

---

## 八、可参照的其他二开源码仓（均源自 TVBox 原版）

这些仓库都是**公开源码**、可 clone 编译，且都从同一开源 TVBox 原版（猫影视开源后）派生。按"想学什么"挑：

| 仓库 | 说明 / 特点 |
| --- | --- |
| **`github.com/CatVodTVOfficial/TVBoxOSC`** | TVBox 原版官方仓库（OSC 版），最贴近"猫影视开源原版"的空壳 |
| **`github.com/o0HalfLife0o/TVBox`** | 常被引用的原版 APP 地址，与原版同源 |
| **`github.com/FongMi/TV`** | 蜂蜜影视 / OK影视 内核（jar 派核心，见第七、九节） |
| **`github.com/takagen99/Box`** | takagen99 版，教程里常作为"Android Studio 编译二开"范例 |
| **`github.com/q215613905/TVBoxOS`** | q215613905 版，支持直播回看等 |
| **`github.com/NewOrin/TVBox`** | 标注"TVbox 开源版（空壳-自行配置）" |
| **`github.com/cyao2q/files`** | 收录多个 TVBox 开源项目代码仓，便于一次性浏览各分支 |

> 区分：**以上均为可编译的源码仓**；而 `qist/tvbox`、`zmivip/Tvbox`、`dahu315/tvbox` 这类是**配置 / 接口仓库**（只放 JSON 接口地址），不是播放器源码，别混。

### 快速 clone 建议
```bash
# 想学 jar 派加载机制（与本项目最对口）
git clone https://github.com/FongMi/TV

# 想看最贴近原版的空壳
git clone https://github.com/CatVodTVOfficial/TVBoxOSC

# 想照教程做 AS 二开编译
git clone https://github.com/takagen99/Box
```

