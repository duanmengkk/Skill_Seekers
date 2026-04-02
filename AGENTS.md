# AGENTS.md - Skill Seekers

AI 编码代理的简要参考指南。Skill Seekers 是一个 Python CLI 工具 (v3.3.0)，可将文档站点、GitHub 仓库、PDF、视频、笔记本、Wiki 等转换为适用于 16+ LLM 平台和 RAG 管道的 AI ready 技能。

## 环境配置

```bash
# 运行测试前必须执行（src/ 布局 — 测试会在包未安装时硬退出）
pip install -e .
# 包含开发工具（pytest, ruff, mypy, coverage）
pip install -e ".[dev]"
# 包含所有可选依赖
pip install -e ".[all]"
```

注意：`tests/conftest.py` 会检查 `skill_seekers` 是否可导入，如果不可导入则调用 `sys.exit(1)`。请始终先以可编辑模式安装。

## 构建 / 测试 / 代码检查命令

```bash
# 运行所有测试（不要跳过测试 — 提交前所有测试必须通过）
pytest tests/ -v

# 运行单个测试文件
pytest tests/test_scraper_features.py -v

# 运行单个测试函数
pytest tests/test_scraper_features.py::test_detect_language -v

# 运行单个测试类方法
pytest tests/test_adaptors/test_claude_adaptor.py::TestClaudeAdaptor::test_package -v

# 跳过慢速/集成测试
pytest tests/ -v -m "not slow and not integration"

# 带覆盖率
pytest tests/ --cov=src/skill_seekers --cov-report=term

# 代码检查 (ruff)
ruff check src/ tests/
ruff check src/ tests/ --fix

# 代码格式化 (ruff)
ruff format --check src/ tests/
ruff format src/ tests/

# 类型检查 (mypy)
mypy src/skill_seekers --show-error-codes --pretty
```

**Pytest 配置**（来自 pyproject.toml）：`addopts = "-v --tb=short --strict-markers"`，`asyncio_mode = "auto"`，`asyncio_default_fixture_loop_scope = "function"`。
**测试标记：** `slow`、`integration`、`e2e`、`venv`、`bootstrap`、`benchmark`、`asyncio`。
**异步测试：** 使用 `@pytest.mark.asyncio`；由于 asyncio_mode 是 `auto`，装饰器通常是隐式的。
**测试文件数量：** 123 个测试文件（107 个在 `tests/`，16 个在 `tests/test_adaptors/`）。

## 代码风格

### 格式化规则（ruff — 来自 pyproject.toml）
- **行长度：** 100 个字符
- **目标 Python 版本：** 3.10+
- **启用的检查规则：** E, W, F, I, B, C4, UP, ARG, SIM
- **忽略的规则：** E501（行长度由格式化器处理）、F541（f-string 样式）、ARG002（接口兼容性的未使用方法参数）、B007（故意的未使用循环变量）、I001（格式化器处理导入）、SIM114（可读性偏好）

### 导入排序
- 使用 isort 排序（通过 ruff）；`skill_seekers` 是一方包
- 标准库 → 第三方 → 一方包，用空行分隔
- 仅在需要前向引用时使用 `from __future__ import annotations`
- 使用 try/except ImportError 保护可选导入（参见 `adaptors/__init__.py` 模式）：
  ```python
  try:
      from .claude import ClaudeAdaptor
      from .minimax import MiniMaxAdaptor
  except ImportError:
      ClaudeAdaptor = None
      MiniMaxAdaptor = None
  ```

### 命名规范
- **文件：** `snake_case.py`（例如 `source_detector.py`、`config_validator.py`）
- **类：** `PascalCase`（例如 `SkillAdaptor`、`ClaudeAdaptor`、`SourceDetector`）
- **函数/方法：** `snake_case`（例如 `get_adaptor()`、`detect_language()`）
- **常量：** `UPPER_CASE`（例如 `ADAPTORS`、`DEFAULT_CHUNK_TOKENS`、`VALID_SOURCE_TYPES`）
- **私有：** 前缀 `_`（例如 `_read_existing_content()`、`_validate_unified()`）

### 类型提示
- 渐进式类型 — 在实际可行的地方添加提示，不必强制全面
- 使用现代语法：`str | None` 而非 `Optional[str]`，`list[str]` 而非 `List[str]`
- MyPy 配置：`disallow_untyped_defs = false`、`check_untyped_defs = true`、`ignore_missing_imports = true`
- 测试排除严格类型检查（`disallow_untyped_defs = false`，对 `tests.*` 设置 `check_untyped_defs = false`）

### 文档字符串
- 每个文件都有模块级文档字符串（三引号，描述用途）
- 公共函数/类使用 Google 风格文档字符串
- 在有用时包含 `Args:`、`Returns:`、`Raises:` 部分

### 错误处理
- 使用特定异常，绝不使用裸 `except:`
- 提供带有上下文的帮助性错误消息
- 对无效参数使用 `raise ValueError(...)`，对状态错误使用 `raise RuntimeError(...)`
- 使用 try/except 保护可选依赖导入，并在失败时给出清晰的安装说明
- 包装异常时使用 `raise ... from e`

### 抑制代码检查警告
- 使用内联 `# noqa: XXXX` 注释（例如 `# noqa: F401` 用于重新导出，`# noqa: ARG001` 用于必需但未使用的参数）

## 项目结构

```
src/skill_seekers/           # 主包（src/ 布局）
  cli/                       # CLI 命令和入口点（96 个文件）
    adaptors/                # 平台适配器（策略模式，继承 SkillAdaptor）
    arguments/               # CLI 参数定义（每个源类型一个）
    parsers/                 # 子命令解析器（每个源类型一个）
    storage/                 # 云存储（继承 BaseStorageAdaptor）
    main.py                  # 统一 CLI 入口点（COMMAND_MODULES 字典）
    source_detector.py       # 从用户输入自动检测源类型
    create_command.py        # 统一 `create` 命令路由
    config_validator.py      # VALID_SOURCE_TYPES 集合 + 每类型验证
    unified_scraper.py       # 多源协调器（scraped_data + 分发）
    unified_skill_builder.py # 成对合成 + 通用合并
  mcp/                       # MCP 服务器（FastMCP + 遗留）
    tools/                   # 按类别分组的 MCP 工具实现（10 个文件）
  sync/                      # 同步监控（Pydantic 模型）
  benchmark/                 # 基准测试框架
  embedding/                 # FastAPI 嵌入服务器
  workflows/                 # 67 个 YAML 工作流预设
  _version.py                # 从 pyproject.toml 读取版本
tests/                       # 120 个测试文件（pytest）
configs/                     # 预设 JSON 抓取配置
docs/                        # 文档（指南、集成、架构）
```

## 关键模式

**适配器（策略）模式** — 所有平台逻辑在 `cli/adaptors/` 中。继承 `SkillAdaptor`，实现 `format_skill_md()`、`package()`、`upload()`。在 `adaptors/__init__.py` 的 ADAPTORS 字典中注册。

**抓取器模式** — 每个源类型都有：`cli/<type>_scraper.py`（包含 `<Type>ToSkillConverter` 类 + `main()`）、`arguments/<type>.py`、`parsers/<type>_parser.py`。在 `parsers/__init__.py` 的 PARSERS 列表、`main.py` 的 COMMAND_MODULES 字典、`config_validator.py` 的 VALID_SOURCE_TYPES 集合中注册。

**统一管道** — `unified_scraper.py` 分发到每类型的 `_scrape_<type>()` 方法。`unified_skill_builder.py` 对 docs+github+pdf 组合使用成对合成，对所有其他组合使用 `_generic_merge()`。

**MCP 工具** — 按类别分组在 `mcp/tools/` 中。`scrape_generic_tool` 处理所有新源类型。

**CLI 子命令** — git 风格在 `cli/main.py` 中。每个委托给模块的 `main()` 函数。

**支持的源类型（17 个）：** documentation（web）、github、pdf、word、epub、video、local codebase、jupyter、html、openapi、asciidoc、pptx、rss、manpage、confluence、notion、chat（slack/discord）。每个都由 `source_detector.py` 自动检测。

## 项目约定

### 语言规范
- **中文优先**: 所有的文档都以中文名字命名，内容也以中文为主
- **代码文件**: 英文命名，TypeScript/Python 注释根据上下文选择语言
- **敏感数据**: 永远不要提交 `.env` - 它已在 `.gitignore` 中
- **示例代码安全**: 代码示例中禁止出现明文token、密码、API密钥等敏感凭证，应使用占位符(如 `YOUR_API_TOKEN`、`username:password`)或Base64格式示例，避免敏感信息泄露风险

## Git 工作流

- **`main`** — 生产环境，受保护
- **`development`** — 默认 PR 目标，活跃开发
- 功能分支从 `development` 创建

## 提交前检查清单

```bash
ruff check src/ tests/
ruff format --check src/ tests/
pytest tests/ -v -x   # 首次失败时停止
```

不要提交 API 密钥。使用环境变量：`ANTHROPIC_API_KEY`、`GOOGLE_API_KEY`、`OPENAI_API_KEY`、`GITHUB_TOKEN`。

## CI

GitHub Actions（7 个工作流在 `.github/workflows/`）：
- **tests.yml** — ruff + mypy 检查作业，然后 pytest 矩阵（Ubuntu + macOS，Python 3.10-3.12）并上传 Codecov
- **release.yml** — 标签触发：测试 → 版本验证 → 通过 `uv build` 发布到 PyPI
- **test-vector-dbs.yml** — 测试向量数据库适配器（weaviate、chroma、faiss、qdrant）
- **docker-publish.yml** — 多平台 Docker 构建（amd64、arm64）用于 CLI + MCP 镜像
- **quality-metrics.yml** — 可配置阈值的质量分析
- **scheduled-updates.yml** — 每周热门框架技能更新
- **vector-db-export.yml** — 每周向量数据库导出

## 自定义 LLM Provider 支持

Skill Seekers 支持使用自定义 LLM API 端点（如 MiniMax）进行技能增强。

### 环境变量

| 变量名 | 必填 | 默认值 | 说明 |
|--------|------|--------|------|
| `CUSTOM_API_KEY` | 是 | - | API 密钥（自动生成认证头）|
| `CUSTOM_BASE_URL` | 是 | - | API 端点 URL |
| `CUSTOM_MODEL` | 否 | `minimax-m2.5` | 模型名称 |

### 使用方法

```bash
# 打包构建
python3 -m build

# 安装
pip install dist/skill_seekers-3.4.0-py3-none-any.whl --force-reinstall

# 运行增强（需要设置环境变量）
export CUSTOM_BASE_URL="http://your-endpoint.com/api/v1"
export CUSTOM_API_KEY="your-api-key"
export CUSTOM_MODEL="minimax-m2.5"

skill-seekers enhance /path/to/skill --target custom --api-key "your-api-key"
```

### 实现要点
1. **需要 `--target custom`**：必须显式指定，否则会自动检测为 LOCAL 模式

### 踩坑记录

1. **环境变量不传递**：环境变量在子进程中未正确传递，需使用 `--target custom` 强制指定
2. **pip 和 uv 版本冲突**：开发时使用 `python3 -m skill_seekers.cli.main` 或开发模式安装
3. **ThinkingBlock 没有 text 属性**：需遍历所有内容块兼容处理