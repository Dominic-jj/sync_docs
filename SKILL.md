# sync-docs skills
你是项目文档同步助手。对比上一个版本（git tag / commit 历史）与当前代码，自动更新 CHANGELOG.md 和项目内所有 README*.md 文档。

## 执行步骤

### 1. 确定基线版本

按优先级寻找变更基线，找到第一个即停止：

```bash
# 优先级 1：最新的 git tag
LATEST_TAG=$(git describe --tags --abbrev=0 2>/dev/null)

# 优先级 2：CHANGELOG 中最新日期
LATEST_DATE=$(grep -m1 '^## [0-9]' CHANGELOG.md 2>/dev/null | grep -o '[0-9]\{4\}-[0-9]\{2\}-[0-9]\{2\}')

# 优先级 3：仓库首次提交
FIRST_COMMIT=$(git log --reverse --format='%H' | head -1)
```

确定基线 `BASE` 的逻辑：
- 如果有 `LATEST_TAG`：`BASE = LATEST_TAG`
- 否则如果有 `LATEST_DATE`：`BASE = git log --since="$LATEST_DATE 00:00:00" --reverse --format='%H' | head -1`
- 否则：`BASE = FIRST_COMMIT`

### 2. 遍历 commit 历史，构建完整变更记录

遍历从 `BASE`（或首次提交）到 `HEAD` 的所有 commit，按日期分组分析变更。同时将**未提交的工作区变更**也纳入分析。

```bash
# 获取所有 commit 列表（从旧到新），包含日期
git log --reverse --format='%H|%ai|%s' $BASE..HEAD

# 获取每个 commit 的变更文件
git diff-tree --no-commit-id --name-status -r <commit_hash>

# 获取未提交的变更
git status --short
git diff HEAD --stat
```

**历史 commit 分组规则：**
- 同一天的多个 commit 合并到一个日期条目下
- merge commit（`Merge branch ...`）跳过，不单独记录
- 每个 commit 的变更文件用 `git diff-tree` 获取，按目录归类到模块
- 未提交的工作区变更单独作为「待提交」部分，或归入当天日期

**commit message 解析：**
- 从 commit message 提取改动的关键信息
- 如果 message 过于简略（如 `update readme`），结合 diff 内容补充完整描述
- 对比相邻 commit，如果同一文件被多次修改，只记录最终状态的变更意图

### 3. 加载 .gitignore 规则

自动识别并加载项目中的 `.gitignore` 规则，用于后续变更过滤：

```bash
# 读取根目录 .gitignore
cat .gitignore 2>/dev/null

# 读取子目录中的 .gitignore（如存在）
find . -name .gitignore -not -path './.git/*' 2>/dev/null
```

**过滤规则：**
- 将 gitignore 中的模式解析为文件路径前缀列表（如 `build/`、`output/`、`logic/`、`*.lock`）
- 在分析变更文件列表时，排除匹配 gitignore 规则的文件
- 常见过滤目标：构建产物（`build/`、`dist/`）、依赖目录（`node_modules/`、`__pycache__/`）、IDE 配置（`.idea/`）、临时文件（`*.log`、`*.tmp`）

**应用方式：**
- 在步骤 4（CHANGELOG）中：diff 结果先过滤掉 gitignore 文件，再按模块分组
- 如果变更文件**全部**被 gitignore 过滤，跳过该 commit 不记录

### 4. 分析项目结构（自动检测）

扫描项目目录（排除 gitignore 中的路径），识别技术栈和结构，用于后续模块分组：

```bash
# 检测项目类型
ls package.json → 前端/Node 项目
ls requirements.txt / pyproject.toml / setup.py → Python 项目
ls go.mod → Go 项目
ls Cargo.toml → Rust 项目
ls pom.xml / build.gradle → Java 项目

# 扫描源码目录结构（排除 gitignore 路径）
ls -d src/*/ → 列出 src 下的模块
ls -d app/*/ → 列出 app 下的模块（Next.js App Router）
ls -d lib/*/ → 列出 lib 下的模块
ls -d internal/*/ → 列出 internal 下的模块（Go）
```

根据检测结果自动确定模块分组名，规则：
- 将变更文件按所在目录归类（如 `src/components/` → `组件`，`src/app/api/` → `API`）
- 目录名含义不明确时，直接用目录名作为分组名
- 只出现一次的分组合并到 `[其他]`
- gitignore 中的文件不参与分组

### 4.5. 扫描并归类所有 README 文档

在更新文档前，必须递归扫描仓库中的所有 README 类文件，而不是只检查根目录 README：

```bash
# 优先使用 rg
rg --files -g 'README*.md'

# 若 rg 不可用，再退回 find
find . -name 'README*.md' -not -path './.git/*'
```

**处理规则：**
- 根目录 `README.md` 视为项目入口文档，描述整个仓库的用途、依赖、结构、构建与调试流程
- 子目录中的 `README*.md` 视为模块级文档，描述该目录当前的职责、关键文件、使用方式和限制
- 即使模块目录本次没有直接出现在 diff 中，也要检查对应 README 是否已经过时
- 若 README 所描述的入口、API、目录结构、构建参数、硬件连接与当前代码不一致，必须同步修正
- 不为没有 README 的目录自动补文档，除非用户明确要求创建

### 5. 更新 CHANGELOG.md

如果 CHANGELOG.md 不存在，先创建：

```markdown
# Changelog

所有有意义的变更都会记录在此文件中。

<!-- 新条目插在下方 -->

```

在 `<!-- 新条目插在下方 -->` 下方插入新条目。如果有 tag，用 tag 名做标题；否则用日期：

```
## [版本号] - YYYY-MM-DD
或
## YYYY-MM-DD
```

条目格式：

```
### [分类] 模块名
- 具体改动描述（简洁，一行一个，参考 commit message 但用更完整的表述）
```

**分类标签（通用）：**
- `[新增]` — 新功能、新文件、新模块
- `[修改]` — 修改已有功能行为
- `[修复]` — bug 修复
- `[重构]` — 代码结构调整、重命名、文件移动
- `[其他]` — 配置、文档、依赖更新、杂项

**注意事项：**
- **日期倒序排列**：最新日期在最上面，最早日期在最下面。`<!-- 新条目插在下方 -->` 紧跟最新条目
- 同一天的多个改动合并到一个日期下
- 日期使用 commit 的作者日期（`%ai`），工作区变更使用当天日期
- 忽略无意义变更（格式调整、空行、lock 文件变化、.gitignore 自身变化）
- 不删除或修改已有记录
- 如果使用语义化版本 tag，标题格式为 `## [vx.y.z] - YYYY-MM-DD`
- **历史 commit 也要完整记录**，按日期倒序排列（最新在上）

### 6. 检查并更新所有 README*.md

README 文档统一只描述“当前状态”，不承载变更历史。变更历史统一由 CHANGELOG.md 管理。

**根目录 README.md 必须包含的章节：**
- 项目简介（一段话描述项目是什么、做什么）
- 依赖（开发环境要求）
- 项目结构（目录树，排除 gitignore 路径）
- 构建命令
- 烧写/上传说明
- 调试说明
- 引脚分配说明（如适用）

**模块 README（如 `src/README.md`、`lib/sdk/README.md`）至少应覆盖：**
- 模块职责：当前目录在整个项目中的定位
- 关键文件：主入口、核心源码、配置头文件、示例文件等
- 使用方式：构建参数、入口切换方式、依赖的硬件连接或外部条件
- 现状说明：哪些接口仍保留、哪些路径已废弃、哪些内容只是兼容/遗留能力

**README 不包含：**
- 变更记录 / changelog / 历史记录 — 这些只在 CHANGELOG.md 中
- 重复的版本历史信息

**更新 README 时遵循：**
- 先扫描所有 README，再逐个判断是否过时
- 检查项目结构是否有变化（新增/删除了目录或核心文件），如有则更新目录树或关键文件列表
- 检查依赖/构建命令/运行方式是否有变化，如有则更新对应章节
- 检查当前代码入口是否变化（如 main.c 切换用途、构建开关新增、示例目录扩展），如有则同步更新相关 README
- 只更新受影响的部分，不重写与当前任务无关的说明
- 保持每个 README 已有的章节结构和语言风格；若原文档主题已经失效，可重构其结构，但必须保留该目录的真实职责说明

**通常不需要更新 README 的情况：**
- 纯 bug 修复
- 代码内部逻辑优化
- 注释/格式调整
- 测试文件变更
- 不影响用户使用方式的内部重构

**即使本次改动主要是代码，也要特别检查 README 是否提到了以下易过时信息：**
- 旧入口文件名或旧 build mode
- 旧引脚定义、旧时钟频率、旧串口路径
- 旧 API 形态（例如从 inline 头文件实现切换到 `.c` 实现）
- 旧示例列表、旧目录树、旧硬件连接说明

### 7. 输出摘要

完成后向用户展示：
- 基线版本信息（tag / 日期 / 首次 commit）
- 遍历了多少个历史 commit
- 新增的 CHANGELOG 条目内容预览
- 扫描到了哪些 README，哪些 README 有更新及更新了哪些部分
- 如果有无法自动判断的变更，列出来让用户确认
