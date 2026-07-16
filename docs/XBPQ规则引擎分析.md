# XBPQ 规则引擎反编译分析

> 本文基于对 `xBPQ.jar`（菜妮丝线 `xBPQ1107.jar`，构建于 2022-11-07）的反编译结果整理。
> 反编译工具：`jadx 1.5.6`（macOS，`brew install jadx`）。
> 样本规则：`boyss180/gao-xBPQ` 仓库的 `xBPQ/LIBVIO.json`。

---

## 一、概述

`xbpq.jar` 是 TVBox / CatVod 生态里**规则驱动型爬虫解析器**的实现包（详见 `开发文档.md` 第十节）。它本身**没有公开的 Java 源码仓库**，所有能找到的仓库里放的都只是编译好的 `.jar`（内部是 `classes.dex`）。本文通过反编译其二进制来还原规则引擎的核心逻辑。

关键事实：
- 二进制形态：`.jar` 内含 `classes.dex`（约 950KB），属 Android/Dalvik 字节码。
- 主类：`com.github.catvod.spider.xBPQ`（类名小写），被播放器以 `api:"csp_XBPQ"` 反射加载。
- 字符串混淆：代码中所有明文字符串都用 `BfT.d("<hex>")` 解密（详见第三节）。
- 同 jar 内还打包了其它 spider（`XPath`、`Bili`、`Alist`、`merge/*` 等）与 parser（`JsonBasic`/`JsonParallel`/`JsonSequence`/`MixDemo`），说明这个 jar 是一个「多功能聚合 spider 包」，XBPQ 规则引擎只是其中一部分。

---

## 二、反编译记录

```bash
# 1. 下载 jar（菜妮丝线，config 里实际引用的版本）
curl -L -o xBPQ.jar "https://raw.githubusercontent.com/boyss180/gao-xBPQ/master/jar/xBPQ1107.jar"

# 2. jadx 反编译为 Java 源码
jadx -d out xBPQ.jar
```

反编译产物包结构（节选）：
```
com/github/catvod/
├─ spider/
│  ├─ xBPQ.java          # ★ 规则引擎主类（2351 行）
│  ├─ XPath.java         # XPath 规则套娃 spider
│  ├─ Init.java / Proxy.java
│  ├─ Bili.java / Alist.java / IQIYI.java / ...   # 其它内置 spider
│  └─ merge/             # 大量混淆类（AZ/Ak/B/...），含字符串解密器 BfT
└─ parser/
   ├─ JsonBasic.java / JsonParallel.java / JsonSequence.java / MixDemo.java
```

> 注：`jadx` 反编译 `xBPQ.java` 时报了 6 个错误（少数方法未完全还原），但核心逻辑可读。

---

## 三、字符串混淆与解密（BfT）

代码里所有字符串常量都是密文，靠 `com.github.catvod.spider.merge.BfT.d()` 解密：

```java
// BfT.java（反编译后）
public class BfT {
    private static final String KEY = "jDLKJK";
    public static String d(String str) {
        // 1) 把 16 进制字符串每 2 字符转 1 字节
        // 2) 每个字节与 KEY 循环异或
        for (int i = 0; i < len; i++) b[i] = (byte)(b[i] ^ KEY.charAt(i % keyLen));
        return new String(b);
    }
}
```

即：**hex 解码 → 逐字节 XOR 密钥 `"jDLKJK"`**。据此可解出引擎内部使用的真实符号：

| 密文(hex) | 解密值 | 用途 |
| --- | --- | --- |
| `4C` | `&` | 截取边界分隔符之一 |
| `49` | `#` | 多关键词列表分隔（如 `[不包含:a#b]`） |
| `3660` | `\$` | 转义 `$`（正则） |
| `366E` | `\*` | 转义 `*` |
| `366F` | `\+` | 转义 `+` |
| `366A` | `\.` | 转义 `.` |
| `3662` | `\&` | 转义 `&` |
| `547A` | `>>` | `[替换:原>>新]` 的分隔符 |
| `31A2D7F4ACC6C8` | `[替换` | 替换指令前缀 |
| `31ACF3F6AFC1CA` | `[追加` | 追加指令前缀 |
| `49186868` | `#\$#` | 占位标记 |
| `4E75` | `$1` | 正则反向引用（replaceAll 用） |
| `00372325ACDEDAA3F7CF` | `json数组` | JSON 数组模式字段 |
| `8DEDF6` / `8DEDF66F6EACC3FE` | `空` / `空$$空` | “空”占位符 |
| `0925382E032F` | `cateId` | URL 模板变量 |
| `8DF5F7AED4C0` / `8FC1E4A2C9E3` / `8FCDEBADC9CE` | `类型` / `全部` / `剧情` | 筛选相关字段 |

---

## 四、真实规则语法（基于 LIBVIO.json）

> 教程里常写成 `&&` / `$$` 截取，但**本 jar（xBPQ1107）实际语法**以 `&&` 为边界、`&` 为列表分隔。下面的例子是真实规则文件，权威可靠。

```json
{
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
  "筛选": { "1": [ {"key":"class","name":"剧情","value":[...]} ], ... }
}
```

### 语法要点

| 元素 | 写法 | 含义 |
| --- | --- | --- |
| **列表分隔** | `电影&剧集&动漫` | 用单个 `&` 分隔多个名称/值 |
| **截取边界** | `left&&right` | 在原文中截取 `left` 与 `right` **之间**的内容（如 `text-right\">&&</span>`） |
| **URL 模板变量** | `{cateId}{catePg}{area}{by}{year}{class}` | 分类/分页/筛选占位符，引擎按上下文替换 |
| **过滤指令** | `[不包含:云盘#网盘]` | 排除含「云盘」或「网盘」的结果；`#` 分隔多个关键词 |
| **替换指令** | `[替换:原>>新]` | 把结果里的 `原` 替换为 `新`，`>>` 分隔 |
| **追加指令** | `[追加:...]` | 在结果后追加内容 |
| **JSON 模式** | `"json数组": "..."` 字段 | 走 JSON 路径解析而非 HTML 截取 |

> 注意：`[不包含:云盘#网盘]` 里的 `#` 就是代码里解密出的 `strD3='#'`，用作多关键词分隔——这印证了引擎内部确实用 `#` 分隔过滤关键词列表。

---

## 五、规则引擎核心逻辑（基于反编译代码）

`xBPQ extends Spider`，重写了以下接口（`crawler/Spider.java` 契约）：
- `homeContent(boolean)` — 首页/分类
- `homeVideoContent()` — 首页推荐
- `categoryContent(tid, pg, filter, extend)` — 分类列表
- `playerContent(flag, id, vipFlags)` — 真实播放地址

还有一组私有 pipeline 方法承担解析：`E / F / H / Hg / Mm / NF / NW / R / Rn / S / T6`（方法名被混淆，但功能可辨为 列表抽取 / 详情抽取 / 线路抽取 / 播放抽取 / 搜索 等）。

### 1. 边界截取（核心）
规则字符串（如 `text-right\">&&</span>`）在引擎里被定位到原文后，按边界分隔符拆分并取中间段：
```java
// 伪代码（对应 xBPQ.java 的 split 逻辑）
String[] left_right = rule.split("&&");      // 边界：&&
String[] list =  multiKeyword.split("#");    // 多关键词：#
// 抽取：在 html 中找 left 之后、right 之前的内容
```

### 2. 替换（[替换:原>>新]）
代码用正则 `.*\[替换:(.*?)\].*`（解密值 `31A2D7F4ACC6C8`）匹配指令，再用 `String.replaceAll(原, 新)` 替换，分隔符为 `>>`（解密值 `547A`，对应 `$1` 反向引用）。

### 3. 追加（[追加:...]）
同上，正则 `.*\[追加:(.*?)\].*`（解密值 `31ACF3F6AFC1CA`）。

### 4. 包含 / 不包含（[不包含:...#...]）
通过 `String.contains(...)` 判断；多个关键词用 `#` 分隔（解密值 `strD3='#'`）。

### 5. Base64 解码
直接 `import android.util.Base64`，支持整体或局部 `Base64()` 解码（应对接口返回 Base64 编码内容）。

### 6. 二次截取 / 嵌套
字段支持在已抽取内容上再做一次截取（如「线路标题」先抽容器再抽标题），对应代码里多层 `split` 处理。

### 7. JSON 路径模式
当规则含 `json数组` 字段（解密值 `00372325...`）时，走 JSON 路径解析（如 `data.list[1].name`），而非 HTML 文本截取。

### 8. 筛选（筛选 / 类型 / 地区 / 年份 / 排序）
对应解密字段 `类型`/`筛选子分类名称`/`筛选类型名称`/`cateId`，规则 JSON 里的 `筛选` 块描述各分类下的可选项，引擎拼进 `分类url` 的 `{class}{area}{by}{year}` 模板变量。

---

## 六、与本项目（CatVodSpider）的关系

- **同一契约层**：`xBPQ` 和本项目的 `spider/PTT.java` 等都继承 `crawler.Spider`，被播放器以 `csp_类名` 反射加载——所以 xBPQ 能像本项目打的 `custom_spider.jar` 一样被 FongMi/TV 系播放器识别。
- **范式差异**：本项目是「每站一个 Java 类」；XBPQ 是「一个规则引擎类 + 每站一份 JSON 规则」。XBPQ 把写爬虫降级为写配置。
- **可集成（精缝）**：把 `xBPQ.jar` 的 smali 与本项目的 spider smali 合并，再走 `genJar.bat` / `apktool` 重打包，即可在同一 `custom_spider.jar` 里同时拥有「内置 Java spider」和「XBPQ 规则引擎」。集成后新增站点可只写 JSON 规则，不必写 Java。

> ⚠️ 集成涉及反编译/修改第三方 jar 并重打包，务必确认来源可信，避免引入安全隐患或导致编译失败。

---

## 七、把 XBPQ 规则接入 FongMi TV（使用步骤）

写好的 XBPQ 规则是一份 **JSON**，要在 FongMi TV（CatVod 系播放器）里用，需同时满足两件事：① 播放器加载的 `custom_spider.jar` 内含 **XBPQ 引擎**（`csp_XBPQ`）；② 订阅配置里把站点指向 `csp_XBPQ` 并把规则放进取 `ext`。

### 1. 前置：确认播放器有 XBPQ 引擎
- **最简**：用社区自带 XBPQ 的播放器 / jar（很多 CatVod 系如 FongMi TV 配套 jar 已含 `csp_XBPQ`）。验证：看其 `custom_spider.jar` 解出的 `classes.dex` 里是否有 `com/github/catvod/spider/xBPQ`。
- **没有就补**：把 XBPQ smali 精缝进本项目打的 jar 再推给播放器（见第六节），或换用自带 XBPQ 的 jar。
- 若 jar 无 XBPQ，配 `api:"csp_XBPQ"` 会找不到类，源直接为空。

### 2. 配置里加一条站点
在 TVBox 订阅 JSON 的 `sites`（有的版本字段名是 `video` / `spider`）数组里加一条：

```json
{
  "key": "myxbpq",
  "name": "我的XBPQ源",
  "type": 3,
  "api": "csp_XBPQ",
  "ext": "这里放你的规则内容"
}
```

- `type: 3` 表示自定义 spider 源；
- `api: "csp_XBPQ"` 让播放器反射实例化 XBPQ 引擎，并把 `ext` 作为规则喂给它；
- 多站点就加多条，每条 `api:"csp_XBPQ"` + 各自 `ext`，最稳。

### 3. ext 放规则（两种写法，以播放器约定为准）
- **明文 JSON**：直接把规则 JSON 字符串整体放进 `ext`（字段如 `主页url` / `分类url` / `分类` 等保持原样）。
- **Base64(JSON)**：规则含换行 / 特殊字符时，先 base64 再放，XBPQ 会自动解：
  ```bash
  base64 -i 你的规则.json   # macOS/Linux，把输出贴进 ext
  ```

### 4. 推给播放器
把整份订阅 JSON 放到可访问地址（自己的 HTTP 服务 / Gitee / 对象存储），FongMi TV → 设置 → 配置地址 → 填 URL 刷新订阅；部分播放器支持本地导入 JSON。

### 5. 验证
能出分类、点进影片有简介 / 演员、播放出线路与 `.m3u8` 直链即成功。失败优先查：① jar 是否含 XBPQ；② ext 是明文还是 base64 是否对；③ 规则符号是否按第四节真实语法（`&&` / `{cateId}` / `[不包含:..#..]` 等）写。

> 一句话：**jar 有 XBPQ + 配置里 `api:csp_XBPQ` + `ext` 放规则**，三样齐即可用。若直接用社区现成带 XBPQ 的播放器，连精缝都不用做，只改配置即可。

---

## 八、小结

- `xbpq.jar` 是**黑盒式规则引擎**：无公开源码，以 `.jar`（classes.dex）形式分发。
- 反编译后确认：主类 `xBPQ`、字符串经 `BfT.d()`（hex+XOR `jDLKJK`）混淆、重写了 `home/category/player` 等 Spider 接口。
- 真实规则语法：`&` 列表分隔、`&&` 边界截取、`{var}` 模板、`[不包含:..#..]`/`[替换:..>>..]`/`[追加:..]` 指令、`Base64()`、`json数组` JSON 模式。
- 与本项目的 spider 同契约，可「精缝」进同一 jar 协同工作。
