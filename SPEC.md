# PoseWear 定制需求

本文记录 iSpace 视角下对上游开源项目 posewear（github.com/atuizz/posewear）的定制项。上游是一个纯静态的姿势、服装、人像参考图库加 AI 试穿工作台，本仓库在它之上按本地工程习惯调整。

文件分两部分：定制需求本身，以及一次具体工作的内部记录。文中的 v0.1、v0.2 是内部条目编号，只用于在本文件内指代某次工作，不是对外发布版本号，项目本身也不需要版本号支持（依据见 v0.2）。

## v0.1 迁移到 uv

### 背景

项目原先没有依赖清单，README 里只写一句「pip install pillow」。第三方依赖只有 Pillow 一个，但它是构建链上唯一的外部依赖：换台机器、换个 Python 版本，能不能装出同一个 Pillow，全靠运气。构建产物和页面生成都依赖它的图片处理行为。

### 做法

- 新增 `pyproject.toml`：用 PEP 621 声明项目元数据与依赖，`requires-python = ">=3.9"` 保持 README 原先声明的支持范围，`[tool.uv] package = false` 表示这是仓库根目录下的脚本集合，不打包成可安装的库
- 新增 `uv.lock`：锁定 Pillow 的具体版本，按 Python 版本分档（3.9 → 11.3.0，3.10 → 12.3.0，3.11 及以上 → 13.0.0），每档都有对应的预编译 wheel，不需要源码编译
- 新增 `.python-version`：固定开发用解释器 3.13，本机没有时 uv 会自动下载
- `.gitignore` 增加 `.venv/`，`uv.lock` 入库以保证依赖可复现
- README、README.en.md、`depth/README.md`、`scripts/fetch-assets.py`、`scripts/indexnow.py` 里的用法提示统一改成 `uv sync` / `uv run`
- `package.json` 的 `build`、`serve` 脚本改成走 `uv run`，npm 侧仍是 Playwright 与 wrangler，未改

### 现在的开发流程

```bash
uv sync                                 # 建 .venv 并装依赖
uv run scripts/fetch-assets.py         # 下载素材到 assets-raw（数 GB）
uv run build.py                         # 构建到 dist/
uv run python -m http.server 8787 -d dist
```

增删依赖走 `uv add` / `uv remove`，改完把 `uv.lock` 一起提交。

### 验证

`uv run depthpack.py` 是仓库自带的自检，迁移后通过。Pillow 的图片流水线（thumbnail 缩放加 WEBP、JPEG 编码）在 uv 环境下可用。完整构建未跑，因为素材图片数 GB，本次只验证到依赖与脚本层面。

## v0.2 版本号

### 结论

项目不需要版本号支持，v0.1、v0.2 只是本文件里的内部条目编号。

### 依据

- 站点是纯静态页面，没有自有后端接口。唯一的服务端是 `functions/api/proxy.js` 转发代理，它透传的是上游服务商的接口路径（OpenAI 的 `/v1`、Gemini 的 `/v1beta`、火山方舟的 `/api/v3`），不存在自有接口需要跨版本演进的问题
- 页面没有版本号展示需求。站点是内容库加工具，版本变化由 README 的「更新日志」按日期记录就够了
- `package.json` 是 `private` 且不带 `version` 字段，不发布 npm 包，没有可发布的版本号
- 仓库里已有的两处版本语义都是内部实现标记，不对外：`src/sw.js` 的缓存名 `posewear-v9`，改前端资源时手动改，作用是让旧缓存失效；`functions/api/proxy.js` 注释里的 `v10`，是代理脚本自身的修订号
- 仓库没有打 git tag，也没有 CHANGELOG 文件

### 因此

不改前端页面显示、不加版本号字段、不打 release tag。v0.1、v0.2 只作内部记录用途。
