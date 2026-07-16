# jadehh/TVSpider 项目说明（含与本项目对比）

> 仓库：<https://github.com/jadehh/TVSpider>
> 简介：影视 / 猫影视（TVBox、CatVodOpen）爬虫仓库，主打「授人以渔」，教大家用 JS 编写 Spider。
> 语言：JavaScript 99.5% + Python 0.5%，许可证 GPL-3.0。
> 作者：jadehh（贡献者 xhybb、jiandehui），最近 Release v1.0.1（2023-02-16），约 670 Stars / 375 Forks。

---

## 一、与现有 CatVodSpider 项目的核心区别

| 维度 | 本项目（CatVodSpider，本地） | jadehh/TVSpider |
| --- | --- | --- |
| 实现语言 | **Java**（Android 工程） | **JavaScript**（早期 QuickJS，后改 Node.js）+ Python 脚本 |
| Spider 形态 | Java 类，继承 `crawler/Spider` 抽象类 | JS 文件（每个站点一个 spider 脚本） |
| 产物 | `custom_spider.jar`（含 `classes.dex`） | `js`/`dist` 分支下的配置 + JS 代码（如 `index.js.md5`） |
| 加载方式 | 播放器用 `DexClassLoader` 动态加载 jar，按 `csp_类名` 反射 | 播放器通过 `gitee://Token@...` 配置串导入私有仓库分支 |
| 目标播放器 | FongMi/TV 等 CatVod 系（jar 派） | **TVBox** + **CatVodOpen**（JS 派） |
| 分发机制 | 本地文件 / HTTP URL 的 jar | GitHub Actions 自动发布到 `js`/`dist` 分支，用户导入配置地址 |
| 调试手段 | Android Studio 跑 Demo App，按钮触发各方法看 JSON | `test.js` / `test.json` 本地验证；Wiki 教程 |
| 配置生成 | `json/config.json` 手写 | `python build.py --aliToken ...` 生成（依赖阿里云盘 Token） |
| 分支策略 | 单一 main | `main`（测试）/ `js`（QuickJS 配置）/ `dist`（Node.js 配置） |

**一句话总结**：两者都是「CatVod / TVBox 生态的爬虫开发项目」，但本项目是 **Java 写、出 jar 包**；`jadehh/TVSpider` 是 **JS 写、出配置分支**，面向的是 TVBox / CatVodOpen 而非 FongMi/TV 的 jar 加载体系。

---

## 二、jadehh/TVSpider 怎么用（终端用户视角）

### 1. 给 TVBox 用
- 软件发布地址：FongMi/Release 的 GitHub 页面。
- 配置地址（导入播放器即可）：
  ```
  https://gh.con.sh/https://raw.githubusercontent.com/jadehh/TVSpider/js/tv_config.json
  ```
  （配置来自仓库的 `js` 分支）

### 2. 给 CatVodOpen 用
- **V1.1.3 及以上**（已改用 Node.js）：
  ```
  gitee://Token@gitee.com/jadehh_743/TVSpider/dist/index.js.md5
  ```
  对应 `dist` 分支。
- **V1.1.2 及以下**（QuickJS）：
  ```
  gitee://Token@gitee.com/jadehh_743/TVSpider/js/open_config.json
  ```
  对应 `js` 分支。
- ⚠️ CatVodOpen 要求把 `dist` / `js` 分支**导入为私有仓库**（仅支持私有仓库）后再用上面的配置串加载。

### 3. 直播源
参考作者的另一个仓库 `jadehh/LiveSpider`。

---

## 三、怎么调试

- **官方 Wiki**：<https://github.com/jadehh/TVSpider/wiki> 提供使用教程，鼓励提 Issue 共同学习。
- **本地测试文件**：仓库内含 `test.js` 和 `test.json`，可在本地 Node 环境跑来验证 Spider 逻辑是否正确。
- **Fork 注意**：Fork 时务必取消「仅复制 main 分支」，才能拿到 `js` / `dist` 全部分支。
- **版本兼容坑**：
  - 老版 CatVodOpen 的 `cfg` 参数是 `string`，TV 影视是 `object`，初始化时要注意用 `this.cfgObj` 区分。
  - CatVodOpen 移除 quickjs 后，旧 `js` 配置无法使用，需切回旧版播放器或改用 `nodejs` 目录编译发布 `dist`。
- **已知问题（排错参考）**：
  - m3u8 跨域：可用代理加载；但若本无跨域又走代理会死循环。
  - 虎牙弹幕：不支持 WebSocket，无法实现。
  - SP360 启用了嗅探，但 CatVodOpen 暂不支持嗅探。
  - TV 影视暂不支持 B 站 DASH 文件播放。
  - 玩偶姐姐资源需切换 VPN 节点。

---

## 四、怎么开发（贡献 / 自写 Spider）

1. **写 Spider**：在 `js/` 或 `nodejs/` 目录下按播放器接口规范编写 JS 爬虫（参考仓库已有 spider 与 Wiki）。
2. **本地验证**：用 `test.js` / `test.json` 跑通逻辑。
3. **分支管理**：
   - `main`：日常代码测试，**不含配置信息**。
   - `js`：发布支持 QuickJS 的爬虫配置。
   - `dist`：发布支持 Node.js 的爬虫配置。
4. **编译 / 发布**：
   - Node.js 部分只生成代码，**需手动 build**，且要区分 Node 18+ 版本。
   - 所有配置通过 **GitHub Actions** 在创建 tag 时自动生成并发布到对应分支；把分支导入私有仓库后即可在播放器加载。

### 配置文件生成（可选）
```bash
python build.py --aliToken <你的阿里云盘Token>
```
- 需要先从 alist.nn.ci 指引获取**阿里云盘 Token**，Token 失效需重新获取。
- ⚠️ 该命令会执行第三方 Python 脚本、读取并写入本地文件、涉及你的私有凭据，**请勿在不可信环境运行，也不要把 Token 提交到公开仓库**。

---

## 五、目录结构速览

```
TVSpider/
├─ .github/workflows/    # GitHub Actions 自动发布
├─ js/                   # QuickJS 爬虫脚本与配置（对应 js 分支）
├─ nodejs/               # Node.js 版爬虫代码（对应 dist 分支）
├─ json/                 # JSON 配置 / 规则
├─ lib/                  # 公共依赖库
├─ wrapper/              # 包装/适配
├─ resources/ assets/    # 资源、图片
├─ build.py              # Python 配置生成（需阿里 Token）
├─ test.js / test.json   # 本地测试
├─ jiguang.json          # 多线路 / 推送配置
├─ package.json          # Node 依赖
├─ requirements.txt      # Python 依赖
└─ README.md / CONTRIBUTING.md / LICENSE.md
```

---

## 六、重要提示

- **免责声明**：资源仅供测试与学习研究，禁止商业用途，使用第三方软硬件后果自负，24 小时内需删除。
- **不要混淆两套体系**：本项目（CatVodSpider）产出的 `custom_spider.jar` 给 **FongMi/TV 类（jar 派）** 用；`jadehh/TVSpider` 的产物给 **TVBox / CatVodOpen（JS 派）** 用，两者**不互通**。
- **凭据安全**：`build.py` 需要阿里云盘 Token，属于私人敏感凭据，切勿泄露或公开提交。
