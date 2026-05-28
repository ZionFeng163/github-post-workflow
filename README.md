# GitHub 开源项目评测 Skill

自动化复现 GitHub 开源项目，体验功能并生成评测文章和推文。

## 功能

- 一键复现 GitHub 开源项目
- Docker / venv 隔离环境，不污染系统
- 自动生成评测文章（中文 + 英文）
- 支持 Web、CLI、GUI 等多种项目类型
- 反 AI 痕迹文风约束

## 安装

### 全局安装（推荐）

```bash
# 克隆仓库
git clone https://github.com/ZionFeng163/github-post-workflow.git

# 复制 skill 到全局目录
cp github-post-workflow/.claude/commands/post.md ~/.claude/commands/post.md
```

### 项目级安装

```bash
# 在你的项目目录下
mkdir -p .claude/commands
cp /path/to/github-post-workflow/.claude/commands/post.md .claude/commands/post.md
```

## 使用方法

```
/post https://github.com/username/project-name
```

## 工作流程

1. **项目准备** - Clone 到临时目录
2. **项目分析** - 读取 README、检查平台兼容性
3. **环境搭建** - Docker 优先，GUI 项目用 venv 隔离
4. **功能体验** - 截图核心功能
5. **内容生成** - 生成中英文评测文章

## 输出位置

```
~/PycharmProjects/GithubProjectPosts/{project-name}/
├── README.md           # 操作流程文档
├── screenshot-*.png    # 截图
├── long-post.md        # 长帖子（中文）
├── long-post-en.md     # 长帖子（英文）
├── short-post.md       # 短帖子（中文）
└── short-post-en.md    # 短帖子（英文）
```

## 环境隔离

| 项目类型 | 方案 |
|---------|------|
| Web 项目 | Docker 优先 |
| GUI 项目 | venv + pip |
| CLI 项目 | venv + pip |

所有依赖都安装在临时目录的 `.venv` 中，完成后删除目录即可清理。

## 文风约束

集成了 [anti-slop-writing](https://github.com/adenaufal/anti-slop-writing) 规则：

- 禁用 AI 味词汇（pivotal, crucial, leverage, seamless...）
- 句子长度变化，禁止三件套
- 禁止破折号（ChatGPT dash）
- 用具体细节替换泛泛声明
- 个人视角叙述，不是说明书

## 设计原则

1. **环境隔离** - 使用 Docker 或临时虚拟环境
2. **用户控制** - 清理时机由用户决定
3. **双语输出** - 中文 + 英文版本
4. **反 AI 痕迹** - 自然人类写作风格

## 支持的项目类型

- Python Web 应用（Flask, FastAPI, Django）
- Node.js 应用
- GUI 桌面应用（Qt, tkinter）
- CLI 工具
- Docker 化项目

## License

MIT
