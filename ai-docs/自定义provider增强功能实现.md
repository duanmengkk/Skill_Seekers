# Custom Provider Enhancement 实现文档

## 需求背景

用户需要支持使用自定义的 LLM API 端点进行技能增强（enhancement），参考的 API 配置如下：

```bash
curl -s "http://your-endpoint.example.com/api/v1/messages" \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -H "x-api-key: YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -H "anthropic-version: 2023-06-01" \
  -d '{"model": "minimax-m2.5", "max_tokens": 10, "messages": [{"role": "user", "content": "hi"}]}'
```

这是一个 MiniMax 兼容 API，使用 Anthropic SDK 兼容接口，需要：
- `base_url`: 自定义 API 端点
- `api_key`: API 密钥（会自动生成 Authorization: Bearer 和 x-api-key 头）
- `model`: 模型名称

## 实现方案

### 环境变量设计

| 变量名 | 必填 | 默认值 | 说明 |
|--------|------|--------|------|
| `CUSTOM_API_KEY` | 是 | - | API 密钥（会自动生成 `Authorization: Bearer` 和 `x-api-key` 头）|
| `CUSTOM_BASE_URL` | 是 | - | API 端点 URL |
| `CUSTOM_MODEL` | 否 | `minimax-m2.5` | 模型名称 |

### 修改文件清单

1. `src/skill_seekers/cli/agent_client.py` - 核心 AI 客户端
2. `src/skill_seekers/cli/enhance_command.py` - 增强命令分发
3. `src/skill_seekers/cli/arguments/enhance.py` - 命令行参数定义
4. `src/skill_seekers/cli/enhance_skill.py` - 增强功能实现

---

## 开发过程

### 第一步：修改 agent_client.py

#### 1.1 添加 DEFAULT_MODELS

```python
DEFAULT_MODELS = {
    "anthropic": "claude-sonnet-4-20250514",
    "moonshot": "moonshot-v1-auto",
    "google": "gemini-2.0-flash",
    "openai": "gpt-4o",
    "custom": "minimax-m2.5",  # 新增
}
```

#### 1.2 添加 API_KEY_MAP

```python
API_KEY_MAP = {
    "ANTHROPIC_API_KEY": "anthropic",
    "ANTHROPIC_AUTH_TOKEN": "anthropic",
    "MOONSHOT_API_KEY": "moonshot",
    "GOOGLE_API_KEY": "google",
    "OPENAI_API_KEY": "openai",
    "CUSTOM_API_KEY": "custom",  # 新增
}
```

#### 1.3 添加 PROVIDER_TARGET_MAP

```python
PROVIDER_TARGET_MAP = {
    "anthropic": "claude",
    "moonshot": "kimi",
    "google": "gemini",
    "openai": "openai",
    "custom": "custom",  # 新增
}
```

#### 1.4 在 _init_api_client() 中添加 custom 分支

```python
elif self.provider == "custom":
    import anthropic

    kwargs = {"api_key": self.api_key}
    base_url = os.environ.get("CUSTOM_BASE_URL")
    if base_url:
        kwargs["base_url"] = base_url
    
    # 自动生成认证头
    kwargs["default_headers"] = {
        "Authorization": f"Bearer {self.api_key}",
        "x-api-key": self.api_key,
    }
    
    return anthropic.Anthropic(**kwargs)
```

#### 1.5 在 get_model() 中添加 custom 映射

```python
provider_env_map = {
    "anthropic": "ANTHROPIC_MODEL",
    "moonshot": "MOONSHOT_MODEL",
    "google": "GOOGLE_MODEL",
    "openai": "OPENAI_MODEL",
    "custom": "CUSTOM_MODEL",  # 新增
}
```

#### 1.6 在 _call_api() 中添加 custom 到调用组

```python
if self.provider in ("anthropic", "moonshot", "custom"):
    response = self.client.messages.create(...)
    # 处理不同的内容块类型 (TextBlock, ThinkingBlock 等)
    for block in response.content:
        if hasattr(block, "text") and block.text:
            return block.text
        elif hasattr(block, "thinking") and block.thinking:
            return block.thinking
    return None
```

---

### 第二步：修改 enhance_command.py

#### 2.1 添加 custom 到 _get_api_keys()

```python
def _get_api_keys() -> dict[str, str | None]:
    return {
        "claude": os.environ.get("ANTHROPIC_API_KEY") or os.environ.get("ANTHROPIC_AUTH_TOKEN"),
        "gemini": os.environ.get("GOOGLE_API_KEY"),
        "openai": os.environ.get("OPENAI_API_KEY"),
        "kimi": os.environ.get("MOONSHOT_API_KEY"),
        "custom": os.environ.get("CUSTOM_API_KEY"),  # 新增
    }
```

#### 2.2 添加 custom 到自动检测优先级

```python
# 优先级: Anthropic > Custom > Gemini > OpenAI
if api_keys["claude"]:
    return "api", "claude"
if api_keys["custom"]:
    return "api", "custom"
if api_keys["gemini"]:
    return "api", "gemini"
```

#### 2.3 添加 custom 到 API key 映射

```python
env_map = {
    "claude": api_keys["claude"],
    "gemini": api_keys["gemini"],
    "openai": api_keys["openai"],
    "custom": api_keys["custom"],  # 新增
}
```

---

### 第三步：修改 arguments/enhance.py

```python
"target": {
    "flags": ("--target",),
    "kwargs": {
        "type": str,
        "choices": ["claude", "gemini", "openai", "kimi", "custom"],  # 添加 custom
        "help": "...",
    },
}
```

---

### 第四步：修改 enhance_skill.py

#### 4.1 添加 custom 到 choices

```python
parser.add_argument(
    "--target",
    choices=["claude", "gemini", "openai", "kimi", "custom"],  # 添加 custom
    ...
)
```

#### 4.2 处理 custom provider

```python
if args.target == "custom":
    _run_custom_enhancement(args, skill_dir)
    return
```

#### 4.3 实现 _run_custom_enhancement() 函数

这是完整的实现函数，包含：
- 读取 reference 文件
- 构建增强 prompt
- 使用 AgentClient 调用 custom API
- 保存增强结果

#### 4.4 实现 _build_custom_prompt() 函数

构建发送给 AI 的完整 prompt，包含：
- 当前 SKILL.md 内容
- Reference 文档内容
- 增强指令

---

## 测试过程

### 单元测试

```bash
pytest tests/test_agent_client.py -v
# 结果：71 passed
```

### 手动测试

#### 创建测试技能目录

```bash
mkdir -p /tmp/test-skill/references
echo "# Test Skill" > /tmp/test-skill/SKILL.md
echo "# Reference" > /tmp/test-skill/references/test.md
```

#### 运行增强命令

```bash
CUSTOM_BASE_URL="http://your-endpoint.example.com/api/v1" \
CUSTOM_API_KEY="YOUR_API_KEY" \
CUSTOM_MODEL="minimax-m2.5" \
python3 -m skill_seekers.cli.main enhance /tmp/test-skill --target custom --api-key "YOUR_API_KEY"
```

#### 测试输出

```
🤖 Enhancement mode: API (custom)
Reading reference documentation...
  Read 1 reference files
  Total size: 32 characters

  Found existing SKILL.md (22467 chars)

============================================================
ENHANCING SKILL: /tmp/test-skill
Platform: Custom API
Model: minimax-m2.5
============================================================

  Input: 23,963 characters
  Generated enhanced SKILL.md (2467 chars)

  Backed up original to: SKILL.md.backup
  Saved enhanced SKILL.md

✅ Enhancement complete!
```

---

## 打包部署测试

### 构建包

```bash
cd /Users/duanmeng/github-repo/Skill_Seekers
python3 -m build
# 输出：dist/skill_seekers-3.4.0-py3-none-any.whl
```

### 安装包

```bash
pip install dist/skill_seekers-3.4.0-py3-none-any.whl --force-reinstall
```

### 测试已安装的包

```bash
# 验证 AgentClient 可以正确初始化
python3 -c "
from skill_seekers.cli.agent_client import AgentClient, DEFAULT_MODELS, API_KEY_MAP
print('DEFAULT_MODELS:', DEFAULT_MODELS)
print('API_KEY_MAP:', API_KEY_MAP)
"
# 输出正确显示 custom 相关配置
```

---

## 踩坑记录

### 坑1：环境变量在子进程中没有传递

**问题**：最初测试时，使用以下命令：

```bash
CUSTOM_BASE_URL="..." skill-seekers enhance /tmp/test-skill
```

结果是进入了 LOCAL 模式（使用 Claude Code CLI），而不是 API 模式。

**原因**：环境变量在命令中设置但在子进程中没有正确传递。

**解决**：使用 `--target custom` 强制指定 API 模式。

---

### 坑2：pip 安装多个版本冲突

**问题**：安装 wheel 包后，Python 找到了旧版本的 CLI（通过 uv 安装的），而不是刚安装的版本。

```bash
which skill-seekers
# 输出：/Users/duanmeng/.local/bin/skill-seekers -> ...uv/tools/...
```

**原因**：uv 安装的版本优先级高于 pip 安装的版本。

**解决**：使用 `python3 -m skill_seekers.cli.main` 直接运行开发版本，或者先卸载再安装：

```bash
pip uninstall skill-seekers -y
pip install -e .
```

---

### 坑3：修改文件后没有生效

**问题**：修改了 `arguments/enhance.py` 和 `enhance_skill.py`，但运行 `skill-seekers enhance --help` 时 `--target` 选项仍然不包含 `custom`。

**原因**：安装的是打包好的 wheel 包，而不是开发版本。

**解决**：
```bash
# 方案1：使用开发版本
pip install -e .

# 方案2：使用 python -m 直接运行
python3 -m skill_seekers.cli.main enhance ...
```

---

### 坑4：API 认证失败

**问题**：初次调用 custom API 时报错：
```
custom API authentication failed: Request denied by Apikey Extract check. No Bearer Authentication information found.
```

**原因**：API 需要 Bearer 认证，但默认只传递了 `api_key`。

**解决**：在代码中自动从 `CUSTOM_API_KEY` 生成认证头：

```python
kwargs["default_headers"] = {
    "Authorization": f"Bearer {self.api_key}",
    "x-api-key": self.api_key,
}
```

---

### 坑5：ThinkingBlock 没有 text 属性

**问题**：MiniMax API 返回的响应包含 `ThinkingBlock`，直接访问 `response.content[0].text` 会报错：
```
'ThinkingBlock' object has no attribute 'text'
```

**原因**：不同 SDK 版本或 API 返回的内容块类型不同。

**解决**：遍历所有内容块，兼容不同类型：

```python
for block in response.content:
    if hasattr(block, "text") and block.text:
        return block.text
    elif hasattr(block, "thinking") and block.thinking:
        return block.thinking
```

---

### 坑6：自动检测模式不生效

**问题**：设置了 `CUSTOM_API_KEY` 等环境变量，但 `skill-seekers enhance` 命令仍然使用 LOCAL 模式。

**原因**：`_pick_mode()` 函数中的自动检测逻辑只检查了特定的几个 API key 环境变量，没有包含 `CUSTOM_API_KEY`。

**解决**：修改 `enhance_command.py` 中的 `_get_api_keys()` 和 `_pick_mode()` 函数，添加对 `custom` 的支持。

---

## 最终使用方式

### 方式1：使用 python -m 运行（推荐开发时）

```bash
CUSTOM_BASE_URL="http://your-endpoint.example.com/api/v1" \
CUSTOM_API_KEY="your-key" \
CUSTOM_MODEL="minimax-m2.5" \
python3 -m skill_seekers.cli.main enhance /path/to/skill --target custom --api-key "your-key"
```

### 方式2：安装打包版本后使用

```bash
# 构建
python3 -m build

# 安装
pip install dist/skill_seekers-3.4.0-py3-none-any.whl --force-reinstall

# 使用
CUSTOM_BASE_URL="..." CUSTOM_API_KEY="..." CUSTOM_MODEL="..." \
skill-seekers enhance /path/to/skill --target custom --api-key "..."
```

---

## 相关文件

- `ai-docs/custom-provider-enhancement.md` - 功能使用文档
- `src/skill_seekers/cli/agent_client.py` - 核心 AI 客户端
- `src/skill_seekers/cli/enhance_command.py` - 增强命令分发
- `src/skill_seekers/cli/enhance_skill.py` - 增强功能实现
- `src/skill_seekers/cli/arguments/enhance.py` - 命令行参数定义