# GitHub 开源项目评测

复现 GitHub 开源项目，体验功能并生成评测文章和推文。

## 输入

用户提供 GitHub 仓库链接。如果未提供，询问用户。

## 工作流程

### 1. 项目准备

```bash
# 创建临时工作目录
REVIEW_DIR="/tmp/review-$(basename $GITHUB_URL)-$(date +%s)"
mkdir -p "$REVIEW_DIR"

# Clone 项目（浅克隆，加快速度）
git clone --depth 1 "$GITHUB_URL" "$REVIEW_DIR/repo"
```

### 2. 项目分析

**第一步：获取项目基本信息**

```bash
cd "$REVIEW_DIR/repo"

# 获取 stars/forks（用于文章中增加可信度）
REPO_NAME=$(basename "$GITHUB_URL" .git)
OWNER=$(echo "$GITHUB_URL" | awk -F'/' '{print $(NF-1)}')
gh api "repos/$OWNER/$REPO_NAME" --jq '{stars: .stargazers_count, forks: .forks_count, description: .description}'
```

**第二步：读取并分析以下文件：**
- `README.md` - 项目介绍和功能
- `package.json` / `pyproject.toml` / `requirements.txt` / `Cargo.toml` - 依赖和语言
- `docker-compose.yml` / `Dockerfile` - 容器配置
- `SKILL.md` - 技能包配置（如果有）
- 其他配置文件

**第三步：平台兼容性检查**

在开始前必须告知用户支持的平台：

```bash
# 检查是否有平台限制
check_platform() {
  local readme=$1
  
  # 检查 Windows 专用
  if grep -qi "windows only\|\.exe\|visual studio\|\.net framework\|powershell required" "$readme" 2>/dev/null; then
    echo "⚠️ 此项目可能仅支持 Windows"
    return 1
  fi
  
  # 检查 macOS 专用
  if grep -qi "macos only\|osx only\|homebrew\|swift\|appkit" "$readme" 2>/dev/null; then
    echo "⚠️ 此项目可能仅支持 macOS"
    return 1
  fi
  
  # 检查 Docker 支持
  if [ -f "Dockerfile" ] || [ -f "docker-compose.yml" ]; then
    echo "✅ 有 Docker 支持，跨平台兼容"
    return 0
  fi
  
  # 检查常见跨平台技术
  if grep -qi "python\|node\|golang\|rust\|java" "$readme" 2>/dev/null; then
    echo "✅ 跨平台兼容（基于跨平台技术栈）"
    return 0
  fi
  
  echo "⚠️ 无法确定平台兼容性，请查看项目文档"
  return 2
}
```

**必须在 clone 完成后、运行前，向用户报告：**
- 支持的平台（Linux/macOS/Windows/Docker）
- 是否有平台限制
- 建议的运行方式
- 项目 stars/forks 数量

**记录：**
- 项目名称和用途
- 主要功能特点
- 技术栈和语言
- 是否有官方 Docker 支持
- 支持的平台
- stars/forks 数量

### 3. 环境搭建与运行

**第一步：检测项目类型**

根据项目特征判断类型：

| 类型 | 特征 | 安装方式 |
|------|------|----------|
| Web 项目 | 有 Dockerfile/docker-compose，或 README 提到 web/server | Docker 优先 |
| CLI (Python) | 有 setup.py/pyproject.toml/requirements.txt | venv + pip |
| CLI (Rust) | 有 Cargo.toml，README 提到 CLI | cargo install |
| CLI (Node) | 有 package.json + bin 字段 | npm install -g |
| npm 包 | 有 package.json + "main" 字段 | npm install（在项目中测试） |
| Skill 包 | 有 SKILL.md 或 npx skills add | npx skills add |
| GUI 项目 | 有 PyQt/PySide/tkinter/wx 依赖 | venv + pip |
| 库/框架 | 被其他项目 import 使用 | pip/cargo/npm install |

**第二步：按类型选择方案**

---

**Web 项目 → Docker 优先：**

```bash
cd "$REVIEW_DIR/repo"

# 有 docker-compose.yml
docker-compose up -d

# 只有 Dockerfile
docker build -t review-app .
docker run -d -p 8080:8080 review-app
```

---

**CLI (Python) 项目 → venv + pip：**

```bash
cd "$REVIEW_DIR/repo"

# 创建隔离环境
python3 -m venv .venv

# 安装依赖（使用完整路径，避免激活问题）
"$REVIEW_DIR/repo/.venv/bin/pip" install -r requirements.txt  # 或 pip install -e .

# 运行（使用完整路径）
"$REVIEW_DIR/repo/.venv/bin/python" main.py
```

⚠️ **注意：** 后台任务中 `source .venv/bin/activate` 可能不生效，始终使用完整路径调用。

---

**CLI (Rust) 项目 → cargo install：**

```bash
cd "$REVIEW_DIR/repo"

# 安装 CLI 工具
cargo install --path .

# 或直接运行测试
cargo run -- --help
```

---

**npm 包 → 在测试项目中使用：**

```bash
cd "$REVIEW_DIR/repo"

# 创建测试项目
mkdir -p /tmp/test-npm && cd /tmp/test-npm
npm init -y
npm install "$REVIEW_DIR/repo"

# 测试使用
node -e "const pkg = require('$(basename $GITHUB_URL)'); console.log(pkg)"
```

---

**Skill 包 → npx skills add：**

```bash
cd "$REVIEW_DIR/repo"

# 检查 SKILL.md 内容
cat SKILL.md | head -50

# 安装到测试项目
mkdir -p /tmp/test-skill && cd /tmp/test-skill
npm init -y
npx skills add "$GITHUB_URL"
```

---

**GUI 项目 → venv + pip（隔离环境）：**

```bash
cd "$REVIEW_DIR/repo"

# 创建隔离的虚拟环境
python3 -m venv .venv

# 安装 GUI 依赖
"$REVIEW_DIR/repo/.venv/bin/pip" install pyqt5  # 或 pyqt6、pyside2

# 运行
"$REVIEW_DIR/repo/.venv/bin/python" -m cola
```

⚠️ **GUI 项目禁止使用：**
- `pip install --user`（会污染用户目录）
- `sudo pip install`（会污染系统）
- `brew install pyqt`（会装到 Homebrew）

---

**库/框架 → 安装并测试导入：**

```bash
cd "$REVIEW_DIR/repo"

# Python 库
python3 -m venv .venv
"$REVIEW_DIR/repo/.venv/bin/pip" install -e .
"$REVIEW_DIR/repo/.venv/bin/python" -c "import $(basename $GITHUB_URL); print('OK')"

# Rust 库
cargo build
```

---

**记录：**
- 项目类型（Web/CLI/Skill/库等）
- 使用的运行方式
- 暴露的端口号（Web 项目）
- 运行状态
- 是否需要特殊依赖（如 Tesseract、Docker）

### 4. 功能体验与截图

**优先使用 GitHub 仓库自带的截图/示例图片！**

**第一步：检查仓库自带资源**

```bash
cd "$REVIEW_DIR/repo"

# 检查常见的截图目录
ls -la assets/ images/ screenshots/ demo/ docs/ examples/ public/ static/ 2>/dev/null

# 检查 README 中引用的图片（区分 badge 和演示图）
grep -E '\.(png|jpg|jpeg|gif|webp|svg)' README.md 2>/dev/null | grep -v 'badge' | grep -v 'shields.io' | grep -v 'img.shields.io'

# 检查 .github 目录下的图片
find .github -name "*.png" -o -name "*.jpg" -o -name "*.gif" 2>/dev/null
```

**第二步：下载仓库自带的截图**

如果仓库有高质量截图，优先下载使用：

```bash
# 处理 GitHub user-attachments URL（需要直接下载）
# 例如：https://github.com/user-attachments/assets/xxx
curl -sL "https://github.com/user-attachments/assets/xxx" -o "$OUTPUT_DIR/screenshot-1.png"

# 处理 raw.githubusercontent.com URL
curl -sL "https://raw.githubusercontent.com/{owner}/{repo}/main/assets/demo.png" -o "$OUTPUT_DIR/screenshot-1.png"

# 处理相对路径（需要先确定 base URL）
# 例如：assets/demo.png → https://raw.githubusercontent.com/{owner}/{repo}/main/assets/demo.png
```

**第三步：只有在没有自带截图时才自己截取**

---

**Web 项目：**
- 使用浏览器打开 localhost:端口
- 截图主要界面和功能
- 尝试核心功能流程

**CLI 项目：**
- 运行主要命令
- 截图终端输出（使用 `script` 或 `asciinema`）
- 记录命令和输出

**GUI 项目：**
- 截图应用界面
- 记录交互流程

---

**截图保存位置：**
直接保存到输出目录：`/Users/zanestear/PycharmProjects/GithubProjectPosts/{project-name}/`

文件命名：
```
screenshot-1.png   # 主界面/首页（优先用仓库自带）
screenshot-2.png   # 核心功能1
screenshot-3.png   # 核心功能2
...                # 更多截图
```

**注意：** 不要创建 screenshots 子目录，截图直接放在项目根目录下。

**截图来源优先级：**
1. GitHub 仓库自带的 assets/images 目录
2. README 中引用的演示图片（排除 badge）
3. 自己截取的运行效果

**截图质量要求：**
- 清晰可读
- 展示核心功能
- 避免空白/无意义内容（如空终端、空页面）
- 如果是文档类项目，截取有实际内容的页面

### 5. 内容生成

生成三个输出文件：

**操作流程文档 (`README.md`)：**
- 项目名称和简介（一句话）
- 项目链接
- 环境要求（Python 版本、Docker 等）
- 完整的复现步骤：
  1. 克隆命令
  2. 进入目录
  3. 启动命令（Docker 或本地）
  4. 访问地址（如果是 Web 项目）
- 截图说明（每张截图对应什么功能）
- 常见问题和解决方案
- 停止/清理命令

**长帖子 (`long-post.md` / `long-post-en.md`)：**

必须是完整的文章，有段落、有叙述、有个人观点。禁止纯罗列。

**重要：标题后第一行必须是 GitHub 链接。**

结构要求：
1. **标题** - 吸引人的标题，不是"XX评测"
2. **GitHub 链接** - 标题后直接给出
3. **开头** - 用第一人称叙述发现这个工具的契机，2-3 段
4. **这是什么** - 简单介绍项目背景，1-2 段
5. **为什么我会注意到它** - 个人视角，1 段
6. **我实际体验了什么** - 安装过程、使用感受，4-6 段，配截图
7. **和其他工具比** - 对比分析，2-3 段
8. **优缺点** - 每个点用一段话解释，不是罗列
9. **适合谁** - 分人群讨论，1-2 段
10. **安装难度和推荐度** - 简短总结
11. **FAQ** - 2-3 个常见问题，每个问题一段回答

字数：1000-1500 字
风格：像在写产品测评博客，有个人体验，有观点，不是说明书

**短帖子 (`short-post.md` / `short-post-en.md`)：**

三段式结构，专为推特 Thread 设计。每段对应一条推文，可直接复制发布。

**格式规范：**
- 每段用 `---` 分隔，表示一条推文的边界
- 每段控制在 250-280 字符（推特限制 280）
- 段内不要有空行
- 截图放在对应的段后面（用 markdown 图片语法）

**三段式结构：**

**第 1 段：钩子 + 简短介绍（必须）**
- 开头用一个痛点或场景吸引注意力（1 句）
- 给出 GitHub 链接
- 一句话说这是什么、解决什么问题
- 一个最吸引人的数字或事实（star 数、用户量、速度等）

**第 2 段：具体功能 + 使用细节**
- 实际用了哪些核心功能（2-3 个）
- 每个功能说一句"怎么用"或"注意什么"
- 可以放 1 张截图
- 这段是干货，让读者知道这东西具体能干什么

**第 3 段：优缺点 + 评分 + 适用人群**
- 优点 1-2 个（具体说为什么好）
- 缺点 1 个（诚实说局限）
- 推荐评分（X/5 或 X/10）
- 适合什么人用（1 句）
- 可以加 hashtag

字数：每段 250-280 字符，总计 750-840 字符
风格：口语化，像在推特上跟人聊天，有起承转合，不要官方语气

---

**双语输出要求：**

写完中文版后，必须写英文版（`long-post-en.md` / `short-post-en.md`）。
英文版要求：
- 内容与中文版一致
- 文风约束相同（反 AI 痕迹规则同样适用）
- 不是直译，是重新用英文写作

### 6. 输出位置

所有输出保存到：`/Users/zanestear/PycharmProjects/GithubProjectPosts/{project-name}/`

```
/Users/zanestear/PycharmProjects/GithubProjectPosts/{project-name}/
├── README.md           # 操作流程文档
├── screenshot-1.png    # 截图1
├── screenshot-2.png    # 截图2
├── ...                 # 更多截图
├── long-post.md        # 长帖子（中文）
├── long-post-en.md     # 长帖子（英文）
├── short-post.md       # 短帖子（中文）
└── short-post-en.md    # 短帖子（英文）
```

**注意：** 截图直接放在项目目录下，不要子目录。

### 7. 发布到博客

内容生成完毕后，询问用户是否发布到博客。

**第一步：让用户选择要发布的内容**

用 AskUserQuestion 询问：
1. 选择要发布的帖子：中文长帖 / 英文长帖 / 中文短帖 / 英文短帖（多选）
2. 选择要上传的截图：列出所有截图文件，让用户勾选（多选）

**第二步：读取源文件**

```bash
# 读取用户选择的 markdown 文件
POST_FILE="/Users/zanestear/PycharmProjects/GithubProjectPosts/{project-name}/{selected-post}.md"

# 获取源目录（用于查找图片）
SOURCE_DIR=$(dirname "$POST_FILE")
```

**第三步：提取文章信息**

从 markdown 文件中提取：
- **标题**：第一行 `#` 开头的内容
- **GitHub 链接**：文中出现的 GitHub 仓库链接
- **内容**：除去标题后的正文内容

**第四步：查找图片**

```bash
# 检查源目录中的图片（只复制用户选择的）
ls -la "$SOURCE_DIR" | grep -E "\.(png|jpg|jpeg|gif|webp)$"
```

图片命名规则：
- `screenshot-1.png` → 封面图 + 文中插图
- `screenshot-2.png` → 文中插图
- 其他图片按顺序编号

**第五步：上传图片到 VPS**

博客服务器信息：
- 地址：`root@45.61.135.162`
- 博客项目路径：`/var/www/blog`
- 文章目录：`/var/www/blog/src/content/posts/{slug}/`

```bash
# 从文件名或标题生成目录名（小写，用连字符分隔）
POST_DIR_NAME=$(echo "$POST_TITLE" | tr '[:upper:]' '[:lower:]' | sed 's/ /-/g' | sed 's/[^a-z0-9-]//g')

# 在 VPS 上创建文章目录
ssh -o StrictHostKeyChecking=no root@45.61.135.162 "mkdir -p /var/www/blog/src/content/posts/$POST_DIR_NAME"

# 上传用户选择的图片到 VPS
scp -o StrictHostKeyChecking=no "$SOURCE_DIR/screenshot-1.png" root@45.61.135.162:/var/www/blog/src/content/posts/$POST_DIR_NAME/ 2>/dev/null
scp -o StrictHostKeyChecking=no "$SOURCE_DIR/screenshot-2.png" root@45.61.135.162:/var/www/blog/src/content/posts/$POST_DIR_NAME/ 2>/dev/null
# 按用户选择上传其他图片
```

slug 规则：只使用小写字母、数字和连字符，如 `understand-anything`

**第七步：在 VPS 上生成文章**

直接在服务器上创建 index.md 文件（包含 frontmatter 和正文）：

```bash
# 在 VPS 上创建 index.md
ssh -o StrictHostKeyChecking=no root@45.61.135.162 << EOF
cat > /var/www/blog/src/content/posts/$POST_DIR_NAME/index.md << 'ARTICLE'
---
title: "$POST_TITLE"
published: $(date +%Y-%m-%d)
description: "$DESCRIPTION"
image: "./screenshot-1.png"
tags: [$TAGS]
category: "$CATEGORY"
draft: false
---

$ARTICLE_CONTENT
ARTICLE
EOF
```

**Frontmatter 字段说明：**

| 字段 | 格式 | 示例 |
|------|------|------|
| title | 字符串 | `"代码库太大看不懂？..."` |
| published | YYYY-MM-DD | `2026-05-30` |
| description | 一句话 | `"Understand-Anything 是..."` |
| image | 相对路径 | `"./screenshot-1.png"` |
| tags | 数组 | `["AI", "工具", "开源"]` |
| category | 字符串 | `"工具推荐"` |
| draft | 布尔 | `false` |

**分类规则：**
- 工具推荐 → `category: 工具推荐`
- 教程类 → `category: 教程`
- 思考类 → `category: 随笔`

**标签提取：**
- 从文章内容中提取关键词
- 从 GitHub 仓库的 topics 中提取
- 常用标签：`AI`, `工具`, `开源`, `开发效率`, `Git`, `Python`, `前端`

**图片引用规则：**
- 封面图：frontmatter 的 `image` 字段自动显示在文章列表
- 文中插图：在描述功能或界面时插入 `![描述](./screenshot-1.png)`
- 每张图片配一句说明文字

**第八步：在 VPS 上构建并推送**

```bash
# SSH 到 VPS 执行构建和推送
ssh -o StrictHostKeyChecking=no root@45.61.135.162 << 'EOF'
cd /var/www/blog

# Git 提交并推送
git add -A
git commit -m "feat: add $POST_TITLE blog post"
git push

# 构建博客
rm -rf .astro dist
NODE_OPTIONS='--max-old-space-size=512' pnpm build
systemctl reload nginx
EOF
```

**第九步：输出结果**

成功后输出：
```
✅ 博客文章已发布！

📝 标题: {POST_TITLE}
📁 目录: src/content/posts/{POST_DIR_NAME}/
🖼️ 封面: screenshot-1.png
🏷️ 标签: {tags}
📂 分类: {category}

🌐 服务器已构建
📤 已推送到 GitHub

🔗 访问: https://zionfeng.org/posts/{POST_DIR_NAME}/
```

**错误处理：**

图片缺失：
1. 询问用户是否需要添加封面图
2. 如果需要，等待用户提供图片路径
3. 如果不需要，使用默认封面或不设置

上传失败：
1. 检查网络连接
2. 检查 SSH 密钥配置
3. 尝试手动上传

构建失败：
1. 检查错误日志
2. 常见问题：图片路径错误、frontmatter 格式错误
3. 修复后重新构建

### 8. 环境清理

**等所有内容生成完毕后，最后才清理！**

**重要：生成的文章和输出目录永远不要删除！**
- 输出目录：`/Users/zanestear/PycharmProjects/GithubProjectPosts/{project-name}/`
- 这个目录和里面的所有文件（文章、截图）是最终成果，不属于临时环境

**只清理临时目录：**
- 项目临时目录：`$REVIEW_DIR`（/tmp/review-*）
- 博客临时目录：`/tmp/blog-post-{slug}`（发布时创建的）
- Docker 容器停止命令：`docker-compose down` 或 `docker stop <container>`
- 清理命令：`rm -rf $REVIEW_DIR /tmp/blog-post-{slug}`

**区分：**
- ✅ 可以删除：`/tmp/review-*`、`/tmp/blog-post-*`（临时文件）
- ❌ 不能删除：`/Users/zanestear/PycharmProjects/GithubProjectPosts/{project-name}/`（生成的文章）

**清理时机：**
1. 确认所有文件已生成到输出目录
2. 确认截图已复制到输出目录
3. 确认文章内容完整
4. 最后才执行清理

## 注意事项

1. **不要在项目目录安装全局依赖**
2. **优先使用 Docker 隔离**
3. **记录所有步骤**，方便用户复现
4. **截图要清晰**，展示核心功能，避免无意义内容
5. **文章面向普通用户**，避免过多技术术语
6. **Clone 用 --depth 1**，加快速度
7. **Python venv 用完整路径调用**，避免激活问题
8. **检查仓库自带截图**，优先使用，没有再自己截

## 文风约束（反 AI 痕迹规则）

**核心原则：** AI 写作失败是因为它优化统计概率。人类写作有历史、观点和具体细节。禁止以下内容：

---

### 词汇禁用列表

**必须禁用的词类：**

| 类别 | 禁用词示例 |
|------|-----------|
| 重要性膨胀词 | pivotal, crucial, paramount, quintessential, indispensable |
| 分析性动词 | underscore, leverage, facilitate, elucidate, spearhead, harness |
| 诗意名词 | tapestry, landscape（比喻用法）, paradigm, ecosystem（比喻用法） |
| 营销形容词 | vibrant, robust, seamless, innovative, holistic, comprehensive |
| 膨胀副词 | seamlessly, profoundly, inherently, relentlessly, fundamentally |

**必须替换的连接词：**
- furthermore → also
- consequently → so
- nonetheless → still
- notwithstanding → despite
- hitherto → until now

**禁止使用的开头/结尾套话：**
- "在当今快速发展的..."
- "作为我们导航复杂性的..."
- "总而言之"
- "最后但同样重要的是"

**禁止使用的虚假权威：**
- "专家认为..."（没有具体来源）
- "研究表明..."（没有具体研究）
- "人们普遍认为..."

**禁止使用的营销套话：**
- "对卓越的承诺"
- "铺平道路"
- "游戏规则改变者"
- "革命性"

---

### 结构规则

1. **句子长度剧烈变化** — 混合 3 词短句和 25+ 词长句；禁止 3 个以上连续相似长度句子
2. **打破三件套** — 列表不要默认三个项目
3. **禁止否定平行结构** — 避免 "不仅是 X，更是 Y"
4. **禁止虚假范围** — "从 X 到 Y" 只用于实际可量化范围
5. **禁止分词结尾** — 句子不要以 "，强调了重要性..." 结尾
6. **禁止套话结论** — 不要 "挑战与未来展望" 这样的章节
7. **禁止强迫总结** — 段落不要以 "总的来说" 或 "总之" 开头
8. **段落节奏** — 长度不规则；单句段落用于强调
9. **禁止垂直列表加粗标题** — 优先用散文而不是加粗要点列表
10. **禁止破折号** — 破折号是 "ChatGPT 破折号"；用句号、逗号、冒号、分号或括号代替
11. **混合句子类型** — 陈述句、疑问句、祈使句和片段混合
12. **打破段落可预测性** — 不要总是以论点开头
13. **变化句法深度** — 在浅层 SVO 和深层多从句结构之间切换
14. **多样化功能词** — 变化连词、介词和冠词
15. **增加词汇多样性** — 使用更多独特词汇、领域术语、专有名词

---

### 内容规则

- 用具体细节替换泛泛声明
- 不要使用模糊归因
- 跳过不增加意义的肤浅分析
- 不要过度强调遗产或历史意义
- 表达真实观点，不要虚假平衡
- 适当展示真正的不确定性

---

### 声音和质感

- 添加人类不完美（冗余、片段、自我纠正）
- 使用语域转换
- 引用具体的真实事件和日期
- 使用第一人称
- 使用英语话语标记（"Well," "Look," "Honestly," "Or rather"）
- 展示情感质感
- 发展独特风格
- 展示知识不对称

---

### 长帖子风格（产品评测博客风）

- 像在写个人博客，有观点、有体验
- 开头用个人故事或发现契机，不要直接介绍产品
- 每个功能点用一段话叙述体验，不要罗列
- 优缺点要具体说明为什么，不要只列关键词
- 对比其他工具时要说出真实感受
- 结尾要有个人推荐，而不是泛泛总结
- 字数：1000-1500 字
- 禁止：纯罗列、空洞形容词、AI 套话、破折号

---

### 短帖子风格（推特 Thread 风）

- 三段式结构，每段对应一条推文
- 每段 250-280 字符，可直接复制到推特发布
- 段与段之间用 `---` 分隔
- 第 1 段：痛点场景 + GitHub 链接 + 一句话介绍 + 数字/事实
- 第 2 段：具体功能使用 + 注意事项 + 可选截图（干货段）
- 第 3 段：优点 + 缺点 + 评分（X/5 或 X/10）+ 适用人群
- 用口语化表达，像在推特上跟人聊天
- 可以有语气词、emoji
- 禁止：官方语气、长句、刻意结构、破折号、罗列要点

---

### 通用原则

- 说人话，不要说"该工具提供了..."
- 用具体数字和日期，不要模糊表述
- 句子长度要变化，不要都一样长
- 可以有口语化的表达
- 不要每段都以主语开头
- 可以有个人观点和情绪
- 截图要配上具体的说明文字

## 错误处理

如果项目无法运行：
1. 记录错误信息
2. 尝试替代方案（如换 Python 版本、安装系统依赖）
3. 如果仍然失败，生成"预览版"文章（基于 README 分析）
4. 明确告知用户哪些功能未能实际测试
