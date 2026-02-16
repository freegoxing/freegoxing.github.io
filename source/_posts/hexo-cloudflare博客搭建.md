---
title: 使用 Hexo 和 Cloudflare Pages 搭建免费个人博客
date: 2026-02-15 19:56:15
tags: [ Hexo, Cloudflare, Blog, CI/CD ]
---

# 前言

本文将详细介绍如何从零开始，使用 Hexo 博客框架和 Cloudflare Pages 服务，免费搭建一个支持自动化部署、全球快速访问的个人博客网站。

作为一个个人博客网站，我们希望这个网站具有下面的优点：

1. **免费**: 无需购买服务器，（可选）无需购买域名。
2. **维护简单**: 通过 Git 推送文章即可自动完成网站更新，专注于内容创作。
3. **访问流畅**: 借助 Cloudflare 的全球 CDN 网络，在世界各地都有良好的访问速度。

## 技术栈

本方案主要由以下几个部分组成：

- **本地环境**:
    - **Node.js**: Hexo 运行所需的环境。
    - **Hexo CLI**: Hexo 命令行工具，用于生成和管理博客。
    - **Typora/Vscode**: 撰写 **Markdown** 文章的软件
- **云端服务**:
    - **GitHub**: 用于托管你的博客源代码。
    - **Cloudflare Pages**: 提供自动化构建、部署以及全球 CDN 加速服务。

整个工作流程是：

1. 在本地用 Markdown 写文章
2. 推送到 GitHub
3. Cloudflare Pages 自动拉取、构建并发布到全球网络。

---

# 一、 环境准备

在开始之前，请确保你已经安装好了必要的软件并拥有相应的账号。

## 1.1 安装必备软件

- **Node.js**: Hexo 基于 Node.js 运行。请从 [Node.js 官网](https://nodejs.org/) 下载并安装 LTS 版本。安装后，`npm`
  包管理器会一同被安装。
  ```bash
  # 检查 Node.js 和 npm 版本
  node -v
  npm -v
  ```
![Node.js 安装](https://img.556756.xyz/PicGo/blogs/2026/02/20260216172236161.png)
  {% note info %}
  了 npm，你也可以使用 pnpm 作为包管理器。pnpm 是一个高性能、节省磁盘空间的 Node.js 包管理工具，在大型项目中优势更加明显。
  ```bash
  # 安装 pnpm
  npm install -g pnpm
  
  # 验证安装
  pnpm -v
  ```
  使用建议：
    - 如果只是简单使用 Hexo，npm 已完全够用。
    - 如果你希望获得更快的依赖安装速度，或者未来会维护多个 Node.js 项目，建议使用 pnpm。
      {% endnote %}

- **Git**: 用于版本控制和代码托管。请从 [Git 官网](https://git-scm.com/) 下载并安装。
  ```bash
  # 检查 Git 版本
  git --version
  ```
![Git 安装](https://img.556756.xyz/PicGo/blogs/2026/02/20260216172236159.png)

## 1.2 注册云服务账号

- **GitHub 账号**: 用于托管博客的源代码。
- **Cloudflare 账号**: 用于构建和部署博客网站。

---

# 二、 搭建 Hexo 博客

## 2.1 安装 Hexo CLI

打开你的终端，使用 npm 或 pnpm 全局安装 `hexo-cli`。

```bash
# 使用 npm
npm install -g hexo-cli
```

## 2.2 初始化项目

选择一个合适的文件夹，执行以下命令来初始化 Hexo 项目。

```bash
hexo init my-blog
cd my-blog
```

{% note info %}
```bash
.
├── _config.yml      # 【核心】站点配置文件
├── package.json     # 依赖包管理文件
├── scaffolds/       # 模版文件夹 (新建文章时的模板)
├── source/          # 【核心】资源文件夹 (你的文章和页面)
│   ├── _posts/      # 存放博客文章的 Markdown 文件
│   └── about/       # (示例) 自定义页面，如“关于我”
├── themes/          # 主题文件夹 (存放 Fluid, Next 等主题)
├── node_modules/    # (自动生成) 项目依赖库，无需手动管理
├── public/          # (自动生成) 最终生成的静态网页文件
└── db.json          # (自动生成) 缓存文件，加快生成速度
```
{% endnote %}

## 2.3 创建内容

我们起一个合适的名字，利用 Hexo 命令可以进行文章的创建

```bash 
hexo new post "我的第一篇博客"
```

## 2.4 本地预览

执行以下命令，在本地启动一个服务器来预览你的博客。

```bash
hexo clean && hexo generate && hexo server
```

{% note info %}
- `hexo clean`: (简写 `hexo c`) 清除缓存文件 (`db.json`) 和已生成的静态文件 (`public` 目录)。这相当于“重置”，通常在更换主题、修改配置文件或遇到渲染异常时使用。
- `hexo generate`: (简写 `hexo g`) 编译源码。将你的 Markdown 文章和主题配置渲染成 HTML 静态网页。
- `hexo server`: (简写 `hexo s`) 启动本地预览服务器。默认地址是 `http://localhost:4000`，让你在发布前检查博客的效果。
{% endnote %}

浏览器访问 `http://localhost:4000`，你应该能看到 Hexo 的默认页面。

![Hexo 的默认界面以及新博客](https://img.556756.xyz/PicGo/blogs/2026/02/20260216172236150.png)

---

# 三、 托管代码到 GitHub

## 3.1 创建 GitHub 仓库

1. 登录 GitHub，创建一个新的**公开**仓库（例如，`my-hexo-blog`）。
2. **不要**勾选 "Initialize this repository with a README" 或其他文件。

## 3.2 关联并推送代码

在本地的 `my-blog` 文件夹中，执行以下命令，将你的代码推送到 GitHub 仓库。

```bash
# 请将 <YOUR_GITHUB_USERNAME> 和 <YOUR_REPO_NAME> 替换为你的实际信息
git init
git remote add origin https://github.com/<YOUR_GITHUB_USERNAME>/<YOUR_REPO_NAME>.git
git branch -M main
git push -u origin main
```

---

# 四、 使用 Cloudflare Pages 实现自动化部署

这是整个流程的核心，它会将你的 GitHub 仓库与 Cloudflare 连接，实现 `git push` 后自动构建和部署。

## 4.1 创建 Cloudflare Pages 项目

1. 登录 Cloudflare 控制台，进入 `构建` > `Workers 和 Pages` > `添加` > `页面` 这个界面

  ![cloudflare page 界面](https://img.556756.xyz/PicGo/blogs/2026/02/20260216172236151.png)

2. 选择 `导入现有 Git 存储库`

  ![cloudflare page 导入界面](https://img.556756.xyz/PicGo/blogs/2026/02/20260216172236156.png)

3. 第一次使用会出现与 GitHub 授权，授权后，我们选择同步到 GitHub 上的博客仓库

  ![GitHub 授权仓库选择界面](https://img.556756.xyz/PicGo/blogs/2026/02/20260216172236160.png)

4. 我们授权完成后，回到 Cloudflare 选择我们的账户和存储库，点击开始设置

  ![GitHub 仓库选择界面](https://img.556756.xyz/PicGo/blogs/2026/02/20260216172236158.png)

## 4.2 配置构建和部署

在 "设置构建和部署" 页面，进行如下配置：

- **构建命令**: `npx hexo c && npx hexo g`
- **构建输出目录**: `public`

{% note success %}
**配置说明：**
- 这里我们使用 `npx` 来运行 Hexo，确保使用项目内安装的 Hexo 版本，避免依赖全局环境。
- `public` 是 Hexo 默认生成的静态文件目录，Cloudflare 会将该目录下的所有内容发布到互联网。

![Cloudflare 构建设置界面](https://img.556756.xyz/PicGo/blogs/2026/02/20260216172236157.png)
{% endnote %}

点击 "保存并部署"，Cloudflare 将会开始第一次构建和部署。完成后，你将获得一个 `*.pages.dev` 的免费域名（在 `生产` > `域` ），通过该域名即可访问你的博客。

![Cloudflare 部署成功界面](https://img.556756.xyz/PicGo/blogs/2026/02/20260216172236155.png)

## 4.3 自动化流程 (CI/CD)

现在，整个自动化流程已经建立。该流程可以概括为以下几个步骤：

1. **本地创作**: 你在本地计算机上撰写或修改 Markdown 文章。
2. **代码推送**: 完成修改后，使用 `git push` 命令将代码推送到 GitHub 仓库。
3. **自动触发**: GitHub 仓库的更新会自动通知 Cloudflare Pages。
4. **云端构建**: Cloudflare Pages 拉取最新的代码，并执行构建命令 (`hexo generate`)，生成静态网站文件。
5. **全球部署**: Cloudflare Pages 将生成好的 `public` 目录部署到其全球 CDN 网络上。
6. **访问更新**: 几分钟后，访问者就可以通过域名看到你博客的最新内容。

你只需要在本地写完 Markdown 文章，然后 `git add .`, `git commit`, `git push`，剩下的部署工作都将自动完成。

---

# 五、 (可选) 绑定自定义域名

如果拥有自己的域名（已经是托管到了 Cloudflare 上面），可以在 Cloudflare Pages 项目的 `自定义域` 设置中轻松绑定。

1. 点击进入 `Workers 和 Pages` > `部署的项目名称` > `自定义域`

![Cloudflare 自定义域](https://img.556756.xyz/PicGo/blogs/2026/02/20260216172236152.png)

2. 点击 `设置自定义域`，进入添加自定义域界面，输入自己的自定义的域名，点击 `继续` 

![Cloudflare 自定义域](https://img.556756.xyz/PicGo/blogs/2026/02/20260216172236154.png)

3. 到达 DNS 记录设置，我们继续点击 `激活域` 就成功的绑定了自定义域名

![Cloudflare 配置DNS](https://img.556756.xyz/PicGo/blogs/2026/02/20260216172236153.png)

# 总结

通过本教程，我们成功搭建了一个功能完善、性能出色且完全免费的个人博客。整个流程充分利用了静态网站生成器和云服务的优势，让我们可以更专注于内容创作本身。