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

# Clone 项目
git clone "$GITHUB_URL" "$REVIEW_DIR/repo"
```

### 2. 项目分析

**第一步：读取并分析以下文件：**
- `README.md` - 项目介绍和功能
- `package.json` / `pyproject.toml` / `requirements.txt` - 依赖
- `docker-compose.yml` / `Dockerfile` - 容器配置
- 其他配置文件

**第二步：平台兼容性检查**

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

**记录：**
- 项目名称和用途
- 主要功能特点
- 技术栈
- 是否有官方 Docker 支持
- 支持的平台

### 3. 环境搭建与运行

**第一步：检测项目类型**

根据项目特征判断类型：

| 类型 | 特征 |
|------|------|
| Web 项目 | 有 Dockerfile、package.json、requirements.txt + 启动命令含 web/server/app |
| GUI 项目 | 有 PyQt/PySide/tkinter/wx 依赖，或 README 提到桌面应用 |
| CLI 项目 | 命令行工具，无 GUI 依赖 |

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

**GUI 项目 → venv + pip（隔离环境）：**

```bash
cd "$REVIEW_DIR/repo"

# 创建隔离的虚拟环境（不污染系统）
python3 -m venv .venv
source .venv/bin/activate

# 安装 GUI 依赖（只装到 venv 里）
pip install pyqt5  # 或 pyqt6、pyside2，根据项目要求

# 运行（从源码直接运行，不安装到系统）
python -m cola  # 或项目的启动命令
```

⚠️ **GUI 项目禁止使用：**
- `pip install --user`（会污染用户目录）
- `sudo pip install`（会污染系统）
- `brew install pyqt`（会装到 Homebrew）

---

**CLI 项目 → venv + pip：**

```bash
cd "$REVIEW_DIR/repo"

# 创建隔离环境
python3 -m venv .venv
source .venv/bin/activate

# 安装依赖
pip install -r requirements.txt  # 或 pip install -e .

# 运行
python main.py  # 或项目的启动命令
```

---

**记录：**
- 项目类型（Web/GUI/CLI）
- 使用的运行方式
- 暴露的端口号（Web 项目）
- 运行状态

**记录：**
- 使用的运行方式
- 暴露的端口号
- 运行状态

### 4. 功能体验与截图

**Web 项目：**
- 使用浏览器打开 localhost:端口
- 截图主要界面和功能
- 尝试核心功能流程

**CLI 项目：**
- 运行主要命令
- 截图终端输出
- 记录命令和输出

**GUI 项目：**
- 截图应用界面
- 记录交互流程

**截图保存位置：**
直接保存到输出目录：`/Users/zanestear/PycharmProjects/GithubProjectPosts/{project-name}/`

文件命名：
```
screenshot-1.png   # 主界面/首页
screenshot-2.png   # 核心功能1
screenshot-3.png   # 核心功能2
...                # 更多截图
```

**注意：** 不要创建 screenshots 子目录，截图直接放在项目根目录下。

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
6. **我实际体验了什么** - 安装过程、使用感受，3-5 段，配截图
7. **和其他工具比** - 对比分析，2-3 段
8. **优缺点** - 每个点用一段话解释，不是罗列
9. **适合谁** - 分人群讨论，1-2 段
10. **安装难度和推荐度** - 简短总结
11. **FAQ** - 2-3 个常见问题，每个问题一段回答

字数：800-1200 字
风格：像在写产品测评博客，有个人体验，有观点，不是说明书

**短帖子 (`short-post.md` / `short-post-en.md`)：**

必须有段落感，不是罗列要点。像朋友圈推荐一样自然。

**重要：开头两句内必须给出 GitHub 链接。**

结构要求：
1. **开头** - 发现这个工具的契机，1-2 句
2. **GitHub 链接** - 开头两句内给出
3. **这是什么** - 一句话介绍
4. **亮点** - 2-3 个亮点，每个用一句话解释为什么好
5. **截图** - 1 张最吸引人的
6. **总结** - 一句话推荐

字数：150-200 字
风格：口语化，像跟朋友聊天，有起承转合

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

## 环境清理

**不要自动删除！** 等待用户明确指示后再清理。

告知用户：
- 临时目录位置：`$REVIEW_DIR`
- 清理命令：`rm -rf $REVIEW_DIR`
- Docker 容器停止命令：`docker-compose down` 或 `docker stop <container>`

## 注意事项

1. **不要在项目目录安装全局依赖**
2. **优先使用 Docker 隔离**
3. **记录所有步骤**，方便用户复现
4. **截图要清晰**，展示核心功能
5. **文章面向普通用户**，避免过多技术术语

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
- 字数：800-1200 字
- 禁止：纯罗列、空洞形容词、AI 套话、破折号

---

### 短帖子风格（朋友推荐风）

- 像跟朋友发微信推荐东西
- 开头说发现这个东西的契机
- 中间自然地说出亮点，不要刻意罗列
- 用口语化的表达，可以有语气词
- 结尾自然收尾，不要刻意总结
- 字数：150-200 字
- 禁止：官方语气、长句、刻意结构、破折号

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
