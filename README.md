# zztracy.github.io 项目说明

本项目是我的一个博客😆，用来梳理输出自己日常学习的内容～

## 项目结构

### 文章内容
- **Redis 相关文章**：
    - `redis-encoding-sds/index.html`：介绍 Redis 数据编码 SDS，对比了 Redis 自己创建的 SDS 数据结构与 C 语言字符串在字符串长度获取、缓冲区溢出问题以及内存分配效率等方面的差异。
    - `redis-network-model/redis-network-model/index.html`：阐述 Redis 网络模型，分析了 Redis 网络模型的性能瓶颈，并介绍了 Redis 6.0 启用多线程后针对客户端命令解析和写响应结果两个环节采用多线程并发处理的过程。
    - `Redis-encoding-intset/index.md`：关于 Redis 数据编码 IntSet 的相关文章。

### 主题相关
- **PaperMod 主题**：
    - `themes/PaperMod` 目录下包含了 PaperMod 主题的相关文件，包括国际化语言文件（如 `i18n` 目录下的 `eo.yaml`, `zh.yaml`, `cs.yaml`, `en.yaml` 等）、主题说明文档 `README.md` 等。

## PaperMod 主题特性

### 安装与更新 📥
更多详细信息请阅读 Wiki：[PaperMod - Installation](https://github.com/adityatelange/hugo-PaperMod/wiki/Installation)

### 特性/模式 💥
- 默认使用 Hugo 的资产生成器，支持管道处理、指纹识别、捆绑和缩小。
- 3 种模式：
    - [Regular Mode.](https://github.com/adityatelange/hugo-PaperMod/wiki/Features#regular-mode-default-mode)
    - [Home-Info Mode.](https://github.com/adityatelange/hugo-PaperMod/wiki/Features#home-info-mode)
    - [Profile Mode.](https://github.com/adityatelange/hugo-PaperMod/wiki/Features#profile-mode)
- 生成目录（采用新的实现方式）。
- 文章归档。
- 社交图标（适用于主页信息和个人资料模式）。
- 文章中的社交媒体分享按钮。
- 菜单位置指示器。
- 多语言支持（带有语言选择器）。
- 分类法。
- 每篇文章都有封面图片（支持响应式图片）。
- 亮/暗主题（根据浏览器主题自动切换，并有主题切换按钮）。
- 搜索引擎优化（SEO）友好。
- 多作者支持。
- 使用 Fuse.js 的搜索页面。
- 文章下方的其他文章推荐。
- 面包屑导航。
- 代码块复制按钮。

### 常见问题及指南 🙋
更多详细信息请阅读 Wiki：[PaperMod-FAQs](https://github.com/adityatelange/hugo-PaperMod/wiki/FAQs)

### 发布日志 📃
发布日志包含了新增内容的信息：[Releases](https://github.com/adityatelange/hugo-PaperMod/releases)

### 页面速度洞察 👀
[Pagespeed Insights (100% ?)](https://pagespeed.web.dev/report?url=https://adityatelange.github.io/hugo-PaperMod/)

### 支持 🫶
- 给这个仓库点个星 🌟。
- [**Fuse.js**](https://github.com/krisk/fuse)
- [**Feather Icons**](https://github.com/feathericons/feather)
- [**Simple Icons**](https://github.com/simple-icons/simple-icons)
- **All Contributors and Supporters**

## 注意事项
- Redis 开启多线程后不保证多个客户端之间命令的串行性，只保证单个客户端内部命令执行以及响应的串行性（通过为一个客户端分配一个单独的线程串行执行来完成），这种设计满足大部分场景的要求，因为客户端通常只关注自身操作的原子性。
