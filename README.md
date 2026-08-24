# 李文杰个人主页 (imamtom.github.io)

这是李文杰 ([imamtom](https://github.com/imamtom)) 的个人学术主页，基于 Jekyll 构建并部署在 GitHub Pages 上。

本仓库 Fork 自 [RayeRen/acad-homepage.github.io](https://github.com/RayeRen/acad-homepage.github.io)，并在其基础上做了大量定制化修改，包括：

- 新增中文主页 (`/zh/`) 与英文主页 (`/`)
- 自定义 BibTeX → GB/T 7714 参考文献渲染流水线
- 多语言侧边栏配置
- 个人化的版面与招生信息

## 站点结构

- **英文主页**：根路径 `/`，源文件位于 [`_pages/about.md`](_pages/about.md)
- **中文主页**：`/zh/`，源文件位于 [`_pages/about-zh.md`](_pages/about-zh.md)
- 顶部导航：见 [`_data/navigation.yml`](_data/navigation.yml) (英文) 与 [`_data/navigation-zh.yml`](_data/navigation-zh.yml) (中文)
- 侧边栏与 SEO 元数据：见 [`_data/authors.yml`](_data/authors.yml)（包含 `default:` 与 `zh:` 两个块）

## 技术栈

- **Jekyll 3.10.0** + [`github-pages`](https://github.com/github/pages-gem) gem（与 GitHub Pages 部署环境保持一致）
- **Pandoc ≥ 2.11** + `--citeproc`：将 `_data/ref.bibtex` 渲染为符合 GB/T 7714-2015 标准的参考文献列表
- **Liquid** 模板引擎
- **Kramdown** Markdown 解析器
- **SCSS** + 自定义样式
- **jQuery** + **MathJax 2.7**（数学公式渲染）

## 常用命令

```bash
# 安装依赖（Ruby gems）
bundle install

# 启动本地开发服务器（含实时刷新）
# 地址：http://127.0.0.1:4000
bash run_server.sh

# 仅构建（不启动服务器）
bundle exec jekyll build

# 修改 _data/ref.bibtex 后，重新渲染 publications 区块
bash bibtex_build/render.sh
```

Python 爬虫（Google Scholar）单独管理依赖：

```bash
pip install -r google_scholar_crawler/requirements.txt
```

## 内容维护

### 添加 / 修改论文

论文列表是**单一数据源**模式：

1. 编辑 [`_data/ref.bibtex`](_data/ref.bibtex)，在文件**末尾**新增一条 BibTeX 条目（顺序即显示顺序，最新的在前）。
2. 执行 `bash bibtex_build/render.sh` 重新生成 `_includes/publications.html`。
3. 若要显示该论文的 Google Scholar 引用数，可在论文正文中加入：
   ```html
   <span class='show_paper_citations' data='GOOGLE_SCHOLAR_PAPER_ID'></span>
   ```
   论文 ID 来自 Google Scholar 引用 URL 中 `citation_for_view=` 后面的字段。

### 修改个人简介

- **英文**：编辑 [`_pages/about.md`](_pages/about.md)
- **中文**：编辑 [`_pages/about-zh.md`](_pages/about-zh.md)
- **侧边栏与 SEO 元数据**：编辑 [`_data/authors.yml`](_data/authors.yml)

### 修改导航栏

- **英文**：[`_data/navigation.yml`](_data/navigation.yml)
- **中文**：[`_data/navigation-zh.yml`](_data/navigation-zh.yml)

中文页面的导航链接必须以 `/zh/` 开头，否则会跳转到英文页面的对应锚点。

## 自动化流水线

### Google Scholar 引用统计

由 [`.github/workflows/google_scholar_crawler.yaml`](.github/workflows/google_scholar_crawler.yaml) 定时执行：

- **触发条件**：每天 08:00 UTC，以及每次 `page_build` 事件
- **执行内容**：使用 [`scholarly`](https://github.com/scholarly-python-package/scholarly) 库抓取作者与每篇论文的引用数
- **输出**：将 `gs_data.json` 与 `gs_data_shieldsio.json` 推送到 `google-scholar-stats` 分支
- **消费方**：浏览器端 `_includes/fetch_google_scholar_stats.html` 在页面加载时拉取并填充引用数

需要配置 GitHub Secret `GOOGLE_SCHOLAR_ID`（个人 Google Scholar ID）。

### Pandoc → GB/T 7714 参考文献渲染

[`bibtex_build/render.sh`](bibtex_build/render.sh) 调用 Pandoc：

```bash
pandoc --citeproc \
  --bibliography=_data/ref.bibtex \
  --csl=bibtex_build/china-national-standard-gb-t-7714-2015-numeric.csl
```

输出包裹后写入 [`_includes/publications.html`](_includes/publications.html)。

> ⚠️ 该文件由脚本自动生成，**不要手动编辑**——下次运行脚本会被覆盖。
>
> ⚠️ GitHub Pages **不会**运行 Pandoc，只会用已经提交到仓库的 `_includes/publications.html`。因此每次修改 `_data/ref.bibtex` 后都必须：
> 1. 本地执行 `bash bibtex_build/render.sh` 重新生成 publications.html
> 2. 将 `_data/ref.bibtex` **和** `_includes/publications.html` 一起提交

## 目录结构

```
.
├── _config.yml              # Jekyll 配置（构建选项、插件等）
├── _data/
│   ├── authors.yml          # 侧边栏 + SEO 元数据（英文 default / 中文 zh）
│   ├── navigation.yml       # 英文页面导航
│   ├── navigation-zh.yml    # 中文页面导航
│   └── ref.bibtex           # 论文 BibTeX（单一数据源）
├── _includes/
│   ├── head.html            # <head> 内容
│   ├── masthead.html        # 顶部导航栏
│   ├── sidebar.html         # 侧边栏容器
│   ├── author-profile.html  # 侧边栏作者卡片
│   ├── publications.html    # ⚠️ 自动生成（Pandoc 输出）
│   ├── seo.html             # <title> 与 Open Graph 元数据
│   └── scripts.html         # 底部 JS
├── _layouts/
│   └── default.html         # 默认布局
├── _pages/
│   ├── about.md             # 英文主页
│   └── about-zh.md          # 中文主页
├── _sass/                   # SCSS 源文件
├── assets/
│   ├── css/                 # 编译后的 CSS
│   ├── js/                  # JavaScript
│   └── fonts/               # Font Awesome / academicons
├── bibtex_build/
│   ├── render.sh            # Pandoc 渲染脚本
│   └── *.csl                # GB/T 7714-2015 CSL 样式
├── docs/                    # 文档与截图
├── google_scholar_crawler/  # Python 爬虫
├── images/                  # 头像、favicon
└── run_server.sh            # 本地开发服务器启动脚本
```

## 本地调试常见问题

### Windows 上 Ruby 与 Bundler

如果 Microsoft Store 别名导致 `bundle` 找不到真实 Ruby：

```bash
# Git Bash 下
export PATH="/d/Ruby33-x64/bin:$PATH"
```

`run_server.sh` 已内置此处理。

### 端口冲突

`run_server.sh` 会先清理占用 4000 / 35729 端口的旧 Jekyll 进程，再启动新的服务器。

### 修改 `_config.yml` 后

`_config.yml` **不会**被 `jekyll serve` 自动重载，需要重启服务器。

## 致谢

本项目基于以下开源项目（均遵循 MIT 许可证）：

- [RayeRen/acad-homepage.github.io](https://github.com/RayeRen/acad-homepage.github.io)
- [mmistakes/minimal-mistakes](https://github.com/mmistakes/minimal-mistakes)
- [academicpages/academicpages.github.io](https://github.com/academicpages/academicpages.github.io)

字体：Font Awesome（[SIL OFL 1.1](https://scripts.sil.org/OFL) 与 MIT）。

参考文献样式：[GB/T 7714-2015 (numeric)](https://github.com/citation-style-language/styles) — 来自 [citation-style-language/styles](https://github.com/citation-style-language/styles)。