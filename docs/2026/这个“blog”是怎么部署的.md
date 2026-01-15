Gemini 太强了, 它手把手教了我这些东西, 下面基本上全是 Gemini 写的

### 第一阶段：本地环境搭建（一次性工作）

确保你电脑里装了 Python（Mac通常自带，Windows去官网装一个）。

**1. 安装核心工具**
打开终端（Terminal / CMD），输入以下命令安装 MkDocs 和 Material 主题：

```bash
pip install mkdocs-material
```

**2. 创建博客项目**
找个你喜欢的地方，创建博客目录：

```bash
mkdocs new my-blog
cd my-blog
```

此时你的目录结构是这样的：
```text
my-blog/
├── mkdocs.yml    # 这是唯一的配置文件
└── docs/         # 这里面放你的 markdown 文章
    └── index.md  # 这是你的博客首页
```

---

### 第二阶段：配置神级主题（关键一步）

默认的 MkDocs 很丑，我们需要启用 Material 主题。
用你喜欢的编辑器（VS Code / 记事本）打开 `mkdocs.yml`，**清空里面的内容**，复制粘贴下面的配置：

```yaml
site_name: 我的个人博客  # 你的博客名字

theme:
  name: material
  language: zh-CN      # 把界面设为中文
  palette:             # 启用自动深色/浅色模式切换
    - scheme: default
      toggle:
        icon: material/brightness-7 
        name: Switch to dark mode
    - scheme: slate
      toggle:
        icon: material/brightness-4
        name: Switch to light mode
  features:
    - navigation.instant   # 像单页应用一样加载，超快
    - navigation.tracking  # 侧边栏自动跟随滚动
    - content.code.copy    # 代码块显示复制按钮

markdown_extensions:
  - admonition          # 漂亮的提示框
  - pymdownx.highlight  # 代码高亮
  - pymdownx.superfences
```
*(保存关闭，配置结束。以后几乎不用动它了。)*

---

### 第三阶段：开始写作（你的日常流程）

这部分完全不用动脑子，**只管写 Markdown**。

**1. 你的文件夹结构 = 网站菜单**
进入 `docs` 文件夹，随便新建文件夹和文件。比如：

```text
docs/
├── index.md             # 首页（必须保留）
├── 技术笔记/             # 网站上会出现 "技术笔记" 的菜单
│   ├── Python学习.md    # 点击菜单会出现这篇文章
│   └── 踩坑记录.md
└── 生活随感/             # 网站上会出现 "生活随感" 的菜单
    ├── 2023书单.md
    └── 旅行照片.md
```

**2. 它是怎么识别标题的？**

*   **方法A（推荐）：** 只要你的 `.md` 文件第一行是 `# 我是标题`，MkDocs 就会自动提取这几个字作为侧边栏的标题。
*   **方法B（懒人）：** 如果你连 `#` 都不写，它会直接用文件名（去掉.md）作为标题。

**3. 本地预览**

在终端里输入：
```bash
mkdocs serve
```
浏览器打开 `http://127.0.0.1:8000`，你就能看到效果了。改动文件保存后，网页会自动刷新。

---

### 第四阶段：部署到 Vercel（自动上线）

为了让别人能访问，我们需要把代码推送到 GitHub，然后用 Vercel 免费托管。

**1. 准备依赖文件（这是坑点，一定要做！）**

Vercel 需要知道你的博客是用什么生成的。在 `my-blog` 根目录下，新建一个名为 `requirements.txt` 的文件，内容只有一行：

```text
mkdocs-material
```

**2. 推送到 GitHub** 

在 `my-blog` 目录下执行：
```bash
git init
git add .
git commit -m "Initial commit"
# 接下来去 GitHub 创建个空仓库，按 GitHub 的提示把本地代码 push 上去
# git remote add origin https://github.com/xxx/xxx.git
# git push -u origin main
```

**3. 在 Vercel 上导入** 

1.  去 [Vercel 官网](https://vercel.com) 注册登录。 
2.  点击 **"Add New..."** -> **"Project"**。
3.  找到你刚才上传的 GitHub 仓库，点击 **Import**。
4.  **关键设置**：
    *   Build Command: 填 `python -m mkdocs build`。
    *   Output Directory: 填 `site`，不用改。
5.  点击 **Deploy**。

等待 30 秒，Vercel 会给你一个 `https://your-blog.vercel.app` 的网址。恭喜，博客上线了！

---

### 以后怎么更新博客？

这就到了你最想要的“极简”环节：

1.  在电脑上 `docs` 文件夹里写个新文件 `今天吃了顿好的.md`。
2.  打开终端：
    ```bash
    git add .
    git commit -m "加了篇文章"
    git push
    ```
3.  **结束。**
    Vercel 会检测到你 Push 了代码，自动在云端帮你安装 Python 环境、编译成 HTML、发布更新。你哪怕在手机上用 GitHub App 修改了文件，博客也会自动更新。