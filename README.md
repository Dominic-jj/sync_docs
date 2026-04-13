# sync_docs

> Claude Code Skill — Auto-sync project documentation (CHANGELOG.md & README.md)

`sync_docs` is a [Claude Code](https://docs.anthropic.com/en/docs/claude-code) skill that analyzes your git commit history and workspace changes to automatically keep your `CHANGELOG.md` and `README.md` up to date.

## Features

- **Language Auto-Detection** — Automatically detects your project's documentation language (Chinese/English) and generates content to match
- **Smart Baseline Detection** — Determines the change starting point from git tags, CHANGELOG dates, or first commit
- **Conventional Commits Support** — Parses `feat:` / `fix:` / `refactor:` prefixes and maps them to categorized labels
- **Breaking Change Detection** — Identifies `BREAKING CHANGE` and `!:` suffixes, highlights breaking changes
- **gitignore Integration** — Loads and applies filter rules to exclude build artifacts, dependencies, and temp files
- **Project Type Detection** — Supports Node.js, Python, Go, Rust, Java, C/C++, Flutter, .NET, and more
- **Module Grouping** — Groups changes by directory structure (Components, API, Routes, etc.)
- **Monorepo Support** — Detects monorepo structures (pnpm workspaces, lerna, nx, turbo) and groups changes by package
- **Workspace Change Analysis** — Includes uncommitted changes (staged and unstaged) in the analysis
- **Large Repository Optimization** — Auto-switches to summary mode for repos with 100+ commits
- **Minimal Intervention** — Only modifies what needs to change, preserves existing style and structure

## Installation

### Option 1: Global Skill (recommended)

```bash
cp SKILL.md ~/.claude/skills/sync-docs/SKILL.md
```

### Option 2: Project-Level Skill

```bash
cp SKILL.md your-project/.claude/skills/sync-docs/SKILL.md
```

### Option 3: Embed in CLAUDE.md

Append the content of `SKILL.md` to your project's `CLAUDE.md` file.

## Usage

Open Claude Code in any git repository and run:

```
请根据 SKILL.md 同步项目文档
```

Or in English:

```
Sync project docs based on SKILL.md
```

Or simply describe what you need:

```
帮我更新 CHANGELOG 和 README
```

## Workflow

```
┌──────────────────────────────────────────────────┐
│  0. Environment Detection & Language Detection    │
│     git check → language detection (CN/EN)        │
├──────────────────────────────────────────────────┤
│  1. Determine Baseline Version                    │
│     git tag → CHANGELOG date → first commit       │
├──────────────────────────────────────────────────┤
│  2. Traverse Commit History                      │
│     git log → diff-tree → group by date           │
│     + workspace changes (staged & unstaged)       │
├──────────────────────────────────────────────────┤
│  3. Load .gitignore Rules                        │
│     Filter build artifacts, deps, temp files       │
├──────────────────────────────────────────────────┤
│  4. Analyze Project Structure                    │
│     Detect tech stack → map module groups          │
├──────────────────────────────────────────────────┤
│  5. Update CHANGELOG.md                          │
│     Categorized labels + module groups + desc     │
├──────────────────────────────────────────────────┤
│  6. Check & Update README.md                     │
│     Read first → update only affected sections    │
├──────────────────────────────────────────────────┤
│  7. Output Summary                               │
│     Baseline info + change preview + confirmations│
└──────────────────────────────────────────────────┘
```

## CHANGELOG Output Format

### Chinese

```markdown
# Changelog

所有有意义的变更都会记录在此文件中。

<!-- 新条目插在下方 -->

## [v1.2.0] - 2025-01-15

### [新增] API
- 添加用户认证接口，支持 JWT token 刷新
- 新增请求频率限制中间件

### [修复] 组件
- 修复日期选择器在 Safari 下的显示异常

### [修改] 配置
- 更新 Node.js 最低版本要求至 18.x
```

### English

```markdown
# Changelog

All notable changes to this project will be documented in this file.

<!-- new entries below -->

## [v1.2.0] - 2025-01-15

### [Added] API
- Add user authentication endpoint with JWT token refresh support
- Add rate limiting middleware

### [Fixed] Components
- Fix date picker display issue in Safari

### [Changed] Config
- Update minimum Node.js version requirement to 18.x
```

## Category Labels

| Chinese | English | Usage |
|---------|---------|-------|
| `[新增]` | `[Added]` | New features, files, modules |
| `[修改]` | `[Changed]` | Modifications to existing behavior |
| `[修复]` | `[Fixed]` | Bug fixes |
| `[重构]` | `[Refactored]` | Code restructuring, renames, file moves |
| `[其他]` | `[Misc]` | Config, docs, dependency updates, etc. |

## Conventional Commits Mapping

| Prefix | Chinese | English |
|--------|---------|---------|
| `feat:` | `[新增]` | `[Added]` |
| `fix:` | `[修复]` | `[Fixed]` |
| `refactor:` | `[重构]` | `[Refactored]` |
| `perf:` | `[修改]` | `[Changed]` |
| `docs:` | `[其他]` | `[Misc]` |
| `chore:` / `build:` / `ci:` | `[其他]` | `[Misc]` |
| `test:` / `style:` | Skip | Skip |

## Supported Project Types

| Type | Indicator Files |
|------|----------------|
| Node.js / Frontend | `package.json` |
| Python | `requirements.txt` / `pyproject.toml` / `setup.py` |
| Go | `go.mod` |
| Rust | `Cargo.toml` |
| Java | `pom.xml` / `build.gradle` |
| C/C++ | `CMakeLists.txt` / `Makefile` |
| Flutter / Dart | `pubspec.yaml` |
| .NET | `*.csproj` / `*.sln` |

## Requirements

- [Claude Code](https://docs.anthropic.com/en/docs/claude-code) CLI, Desktop, or IDE extension
- A git repository with at least one commit

## License

[MIT](LICENSE)
