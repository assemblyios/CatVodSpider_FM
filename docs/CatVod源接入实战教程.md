# CatVod 影视源接入实战教程（从选站到播放）

> 目标：教会你**把一个影视网站变成能在播放器里看的源**。
> 全文分两条路线——**路线 A：用 XBPQ 规则（写 JSON）**；**路线 B：网站不适合 XBPQ 时，写 Java / JS 爬虫**。
> 配套文档：`XBPQ站点选型指南.md`（选型）、`XBPQ规则引擎分析.md`（引擎原理 + 接入播放器）、`开发文档.md`（jar 派开发）、`jadehh-TVSpider说明.md`（JS 派）。

---

## 0. 总览：先决策走哪条路

拿到一个网站，先问自己：**这个站适合用 XBPQ 规则吗？**

```
                         你的网站
                            │
            ┌───────────────┴───────────────┐
            │ 源码可见 + URL 有规律 + 直链可截 + 无强反爬？ │
            └───────────────┬───────────────┘
                是 ─────────┴───────── 否
                  │                    │
           路线 A：XBPQ 规则      路线 B：写爬虫
           （写一份 JSON）      ┌─── Java spider（jar 派）
                               │     → FongMi/TV 等 jar 播放器
                               └─── JS spider（JS 派）
                                     → TVBox / CatVodOpen
```

三种"写源"方式对比：

| 方式 | 产物 | 写源成本 | 给谁用 | 典型代表 |
| --- | --- | --- | --- | --- |
| **jar 源** | `custom_spider.jar` | 高（Android/Java 开发） | FongMi/TV 等 jar 派 | **本项目 CatVodSpider** |
| **js 源** | JS 配置分支 | 中 | TVBox / CatVodOpen | jadehh/TVSpider |
| **XBPQ 规则源** | 一份 JSON 规则 | 低（填截取串） | 加载了 XBPQ 引擎的播放器 | XBPQ |

> 一句话：**能规则化的站优先用 XBPQ（最快）；规则搞不定的站，按你的目标播放器选 Java 或 JS。**

---

# 路线 A：适合 XBPQ 的网站（写 JSON 规则）

## A.1 发现与判断（要不要走这条路线）

判断标准看 `XBPQ站点选型指南.md`，这里只给结论：

**四道硬门槛**（全过才适合）：
1. 数据在 HTML 源码里（右键"查看源代码"能搜到片名）
2. 列表 URL 可模板化（如 `/show/{cateId}---{catePg}.html`）
3. 播放地址可截取（源码里能截到 `.m3u8 / .mp4`）
4. 无强反爬（无 Cloudflare 盾 / 验证码 / 强制登录）

**最省事的信号**：源码里出现 `stui-`、`vodlist`、`/vodplay/`、`/detail/` 等字样 → 八成是苹果CMS（maccms）模板站，**直接套模板改改就能用**。

**5 分钟自测**：看源码搜片名 → 翻分类看 URL 规律 → 进详情看是否明文 → 看播放页能否截直链 → 测搜索是 GET 还是 POST。全过即可。

---

## A.2 写一份 XBPQ 规则（核心技能）

XBPQ 规则就是一份 JSON。引擎做的事：**按你给的"左边界 `&&` 右边界"从网页源码里切出想要的内容**，再拼 URL、过滤、替换。

### A.2.1 先懂四个核心符号

| 符号 | 作用 | 例子 |
| --- | --- | --- |
| `&&` | **边界截取**：切出 `左 && 右` 之间的内容 | `"text-right\">&&</span>"` 切出副标题 |
| `&` | **列表分隔**：多个值用 `&` 分开 | `"电影&剧集&动漫"` |
| `#` | **多关键词分隔**：用于过滤指令 | `[不包含:云盘#网盘]` |
| `>>` | **替换分隔**：`[替换:原>>新]` | 把"原"替换成"新" |

还有几个能力：
- **URL 模板变量**：`{cateId}{catePg}{area}{by}{year}{class}`，引擎按上下文自动填。
- **过滤指令**：`[不包含:云盘#网盘]` 排除含任一关键词的结果。
- **替换 / 追加**：`[替换:原>>新]`、`[追加:内容]`。
- **Base64 解码**：`Base64(...)` 应对接口返回编码内容。
- **JSON 模式**：规则里出现 `json数组` 字段时走 JSON 路径解析（如 `data.list[1].name`），不再做 HTML 截取。

### A.2.2 一份规则的结构（以真实 maccms 站 LIBVIO 为例）

```json
{
  "站名": "LIBVIO",
  "主页url": "https://libvio.fun",

  "分类": "电影&剧集&动漫&日韩剧&欧美剧",
  "分类值": "1&2&4&15&16",
  "分类url": "https://libvio.fun/show/{cateId}-{area}-{by}-{class}-----{catePg}---{year}.html",

  "副标题": "text-right\">&&</span>",

  "搜索模式": "0",
  "搜索后缀": "/detail/",

  "影片地区": "地区：&&/",
  "影片类型": "类型：&&/",
  "导演": "导演：&&</p>",
  "主演": "主演：&&/",
  "简介": "display: none;\">&&</span>",

  "线路数组": "<h3 class=\"iconfont&&</div>",
  "线路标题": ">&&<[不包含:云盘#网盘]",
  "播放数组": "<ul class=\"stui-content&&</ul>",

  "筛选": { ... }
}
```

### A.2.3 字段怎么写（一步步来）

**第 1 步：骨架（必填）**
- `主页url`：站点域名。
- `分类` / `分类值`：分类名与对应 ID，用 `&` 一一对应（数量必须一致）。
- `分类url`：列表页地址模板，把变化的那段用 `{cateId}`、`{catePg}`（页码）、`{area}`、`{by}`（排序）、`{year}`、`{class}`（子类型）占位。

**第 2 步：列表卡片怎么截**
- 引擎拿 `分类url` 拉列表页，按 `分类` 这些字段切出卡片。若列表结构特殊，可用 `数组` / `链接` 等字段用 `&&` 补截（见 A.3 排错顺序）。

**第 3 步：详情字段（用 `&&` 边界截）**
- 进详情页源码，找"标签文字 + 内容"的固定包裹，写成 `标签前缀&&后缀`：
  - `"导演：&&</p>"` 表示切出 `导演：` 之后、`</p>` 之前的内容。
- 同理写 `影片地区` / `影片类型` / `主演` / `简介` / `副标题`。

**第 4 步：线路与播放（最关键的播放能力）**
- `线路数组`：切出"所有播放线路"的容器（如 `<h3 class="iconfont ... </div>`）。
- `线路标题`：在容器内切出每条线路名，常配合过滤：`>&&<[不包含:云盘#网盘]` 表示切出标题且排除含"云盘/网盘"的线路。
- `播放数组`：切出某条线路下的分集列表容器（如 `<ul class="stui-content ... </ul>`），引擎再从中拆出"第N集$播放id"。

**第 5 步：搜索**
- `搜索模式`：`0` 一般用 GET；具体看站点搜索是 GET 还是 POST。
- `搜索后缀`：搜索结果跳详情页的路径特征，如 `/detail/`，用于把搜索词拼成详情链接。

**第 6 步：筛选（可选）**
- `筛选` 块描述各分类下的可选项（地区/年份/类型/排序），引擎拼进 `分类url` 的 `{area}{by}{year}{class}`。
- 结构：`"分类值": [ {"key":"class","name":"剧情","value":[{"n":"喜剧","v":"喜剧"}, ...]}, ... ]`。

### A.2.4 最小模板（拿来改）

```json
{
  "主页url": "https://你的站.com",
  "分类": "电影&剧集&动漫",
  "分类值": "1&2&3",
  "分类url": "https://你的站.com/show/{cateId}---{catePg}.html",
  "简介": "简介：&&</p>",
  "线路数组": "线路容器左&&线路容器右",
  "线路标题": ">&&<",
  "播放数组": "分集容器左&&分集容器右"
}
```
> 先只填骨架 + 线路播放，能播起来再加详情字段和筛选。

---

## A.3 接入播放器（让规则真的能播）

完整步骤见 `XBPQ规则引擎分析.md` 第七节，要点：

1. **前置**：播放器加载的 `custom_spider.jar` 必须含 XBPQ 引擎（`csp_XBPQ`）。
   - 最简：用社区自带 XBPQ 的播放器 / jar；验证看 jar 解出的 `classes.dex` 里是否有 `com/github/catvod/spider/xBPQ`。
   - 没有就补：把 XBPQ 精缝进本项目 jar 再推给播放器（`开发文档.md` 第十节 4），或直接换自带 XBPQ 的 jar。
2. **订阅配置加一条**：
   ```json
   { "key":"myxbpq", "name":"我的XBPQ源", "type":3, "api":"csp_XBPQ", "ext":"你的规则" }
   ```
3. **ext 放规则**：明文 JSON 直接放；含换行/特殊字符就先 `base64 -i 规则.json` 再放（XBPQ 自动解）。
4. **推给播放器**：整份订阅 JSON 放可访问地址（HTTP / Gitee / 对象存储），FongMi TV → 设置 → 配置地址 → 填 URL 刷新；部分支持本地导入。
5. **验证**：能出分类、点进影片有简介/演员、播放出线路与 `.m3u8` 直链即成功。

### XBPQ 规则排错顺序（写错了这样查）
先填 `分类url` → 没数据补 `数组` → 点不开详情补 `链接` → 没播放补 `播放数组` → 播不了补 `播放链接`。可开 `"调试":"1"` 看中间过程。

---

# 路线 B：不适合 XBPQ 的网站（写爬虫）

当网站**数据靠 JS 渲染、URL 靠算参数、播放靠解密**，XBPQ 无能为力，就要写真正的爬虫代码。按**目标播放器**选语言：

| 你的目标播放器 | 写哪种 | 路线 |
| --- | --- | --- |
| FongMi/TV 等 **jar 派** | **Java** spider | 路线 B-1（本项目） |
| TVBox / CatVodOpen（**JS 派**） | **JS** spider | 路线 B-2（jadehh） |

> 注意：**两套产物不互通**。jar 给 jar 派用，JS 给 JS 派用，别混。

---

## B.1 路线 B-1：写 Java spider（给 FongMi/TV 等 jar 播放器）

基于本项目 `CatVodSpider`，详细见 `开发文档.md` 第三/四/五/六/九节。流程：

1. **新建类**：在 `app/src/main/java/com/github/catvod/spider/` 下建类继承 `Spider`，实现关键方法：
   - `homeContent(filter)` 首页分类
   - `categoryContent(tid, pg, filter, extend)` 分类列表
   - `detailContent(ids)` 详情 + 选集
   - `playerContent(flag, id, vipFlags)` 真实播放地址
   - `searchContent(key, quick)` 搜索
   - 参考 `PTT.java`：用 `OkHttp.string(url, header)` 拉页，`Jsoup` 解析 DOM，`Result.string(...)` 组装标准返回。
2. **配置**：在 `json/config.json` 的 `sites` 加一项 `"api": "csp_你的类名"`。
3. **调试**：Android Studio 跑 Demo App，把 `MainActivity` 里 `spider = new PTT();` 改成你的类，点按钮看返回 JSON + Logcat 排错。
4. **打包**：`./gradlew assembleRelease`（macOS/Linux）或 `build.bat`（Windows）→ 产出 `custom_spider.jar`。
5. **播放器加载**：把 `custom_spider.jar` 放进播放器配置目录，配合 `config.json`，播放器用 `DexClassLoader` + `csp_类名` 反射加载（详见 `开发文档.md` 第九节）。

最小示例（`开发文档.md` 第四节）：
```java
public class MySite extends Spider {
    private final String url = "https://example.com/";
    @Override public String homeContent(boolean filter) {
        Document doc = Jsoup.parse(OkHttp.string(url, header()));
        // 解析分类...
        return Result.string(classes);
    }
    @Override public String detailContent(List<String> ids) {
        Vod vod = new Vod();
        vod.setVodPlayFrom("线路1");
        vod.setVodPlayUrl("第1集$playId#第2集$playId2");
        return Result.string(vod);
    }
    @Override public String playerContent(String flag, String id, List<String> vipFlags) {
        return Result.get().url("真实播放地址").string();
    }
}
```

---

## B.2 路线 B-2：写 JS spider（给 TVBox / CatVodOpen）

基于 `jadehh/TVSpider`，详细见 `jadehh-TVSpider说明.md`。流程：

1. **写 Spider**：在 `js/`（QuickJS）或 `nodejs/`（Node.js）目录按接口规范写 JS 爬虫，参考仓库已有 spider + Wiki。
2. **本地验证**：用仓库的 `test.js` / `test.json` 在本地 Node 跑通逻辑。
3. **发布**：把分支（`js` 或 `dist`）导入**私有仓库**，用配置串加载：
   - TVBox / CatVodOpen V1.1.3+：`gitee://Token@gitee.com/jadehh_743/TVSpider/dist/index.js.md5`
   - 老版 CatVodOpen（QuickJS）：`gitee://Token@gitee.com/jadehh_743/TVSpider/js/open_config.json`
4. **终端用户导入**：TVBox 用户直接导入 `https://gh.con.sh/.../TVSpider/js/tv_config.json` 这类配置地址即可。

> 选 JS 还是 Java：**目标播放器决定**。你用 FongMi/TV 就走 B-1（Java），用 TVBox/CatVodOpen 就走 B-2（JS）。成本上 JS 中等、Java 高，但 Java 的产物能直接喂给 jar 派播放器，无需第三方仓库。

---

# 第三部分：速查与排错

## 速查：选站决策表

| 现象 | 结论 | 走 |
| --- | --- | --- |
| 源码能搜到片名 + URL 有规律 + 直链可截 | 适合 XBPQ | 路线 A |
| 源码只有空 `<div id="app">`（SPA） | 不适合 XBPQ | B-1 / B-2（看播放器） |
| 翻页/播放要靠 JS 计算或解密 | 不适合 XBPQ | B-1 / B-2 |
| 有 Cloudflare 盾 / 验证码 | 不适合 XBPQ | B-1 / B-2（且需代理/逆向） |

## XBPQ 规则体检清单
- [ ] 首页源码能搜到已知片名
- [ ] 分类 / 翻页 URL 有规律、可模板化
- [ ] 详情页影片名 / 简介 / 演员为明文，可定位 `&&` 边界
- [ ] 播放页可截到 `.m3u8 / .mp4` 直链
- [ ] 搜索为 GET 请求
- [ ] 无 Cloudflare / 验证码 / 强制登录
- [ ] （可选）源码含 `stui- / vodplay / detail` 等 maccms 特征

**勾满前 6 项 → 用 XBPQ；勾中"可选"项 → 直接套 maccms 模板。**

## 接入播放器常见失败
- 源为空 / 报错：① jar 是否含 `csp_XBPQ`；② `ext` 是明文还是 base64 是否对；③ 规则符号是否按真实语法（`&&` / `{cateId}` / `[不包含:..#..]`）。
- Java spider 不生效：④ `config.json` 里 `api` 是否 `csp_类名` 且类名拼写一致；⑤ jar 是否成功打包并放到播放器目录。

---

# 附：文档索引
- `XBPQ站点选型指南.md` —— 怎么判断网站适不适合 XBPQ（四门槛 + 体检清单）
- `XBPQ规则引擎分析.md` —— 引擎原理 + **第七节：接入 FongMi TV 使用步骤**
- `开发文档.md` —— 本项目 jar 派开发（第四/五/六节）、播放器加载机制（第九节）、XBPQ 源（第十节）
- `jadehh-TVSpider说明.md` —— JS 派开发（路线 B-2）
- `CatVod生态发展史.md` —— 框架(猫影视TV→TVBox)与 XBPQ(菜妮丝/小米小)作者与发展脉络
