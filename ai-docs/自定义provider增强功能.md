# 自定义 Provider 增强功能支持

## 概述

本文档介绍如何配置 Skill Seekers 使用自定义 LLM provider 进行 AI 增强。自定义 provider 使用 Anthropic SDK 兼容的 API，支持配置端点、API 密钥、模型和额外请求头。

## 使用场景

适用于需要使用内部或私有 LLM 端点（如 MiniMax、企业 API）而非公共 provider（如 Anthropic、Google 或 OpenAI）的用户。

## 环境变量

| 变量名 | 必填 | 默认值 | 说明 |
|--------|------|--------|------|
| `CUSTOM_API_KEY` | 是 | - | 认证 API 密钥（会自动生成 `Authorization: Bearer` 和 `x-api-key` 头）|
| `CUSTOM_BASE_URL` | 是 | - | API 端点 URL |
| `CUSTOM_MODEL` | 否 | `minimax-m2.5` | 模型名称 |

## 使用方法

### CLI 命令

```bash
# 设置环境变量
export CUSTOM_BASE_URL="http://your-endpoint.com/api/v1"
export CUSTOM_API_KEY="your-api-key"
export CUSTOM_MODEL="minimax-m2.5"

# 使用 --target custom 运行增强
skill-seekers enhance /path/to/skill --target custom

# 或显式指定 API 密钥
skill-seekers enhance /path/to/skill --target custom --api-key "your-api-key"
```

### 示例：MiniMax API

```bash
export CUSTOM_BASE_URL="http://your-endpoint.example.com/api/v1"
export CUSTOM_API_KEY="your-api-key"
export CUSTOM_MODEL="minimax-m2.5"

skill-seekers enhance /path/to/skill --target custom
```

## 实现细节

### 修改的文件

1. **`src/skill_seekers/cli/agent_client.py`**
   - 在 `DEFAULT_MODELS` 中添加 `custom` provider
   - 在 `API_KEY_MAP` 中添加 `CUSTOM_API_KEY`
   - 在 `PROVIDER_TARGET_MAP` 中添加 `custom`
    - 在 `_init_api_client()` 中添加自定义 provider 初始化（自动生成 Authorization 和 x-api-key 头）
    - 在 `get_model()` 中添加自定义 provider 支持
    - 在 `_call_api()` 中为 Anthropic SDK 添加自定义 provider

2. **`src/skill_seekers/cli/enhance_command.py`**
   - 在 `_get_api_keys()` 中添加 `custom`
   - 在自动检测优先级中添加 `custom`
   - 在 API key 映射中添加 `custom`

3. **`src/skill_seekers/cli/arguments/enhance.py`**
   - 在 `--target` 选项中添加 `custom`

4. **`src/skill_seekers/cli/enhance_skill.py`**
   - 在 `--target` 选项中添加 `custom`
   - 添加 `_run_custom_enhancement()` 函数
   - 添加 `_build_custom_prompt()` 函数

## 代码修改摘要

### agent_client.py

```python
# 添加到 DEFAULT_MODELS
DEFAULT_MODELS = {
    # ... 现有 providers
    "custom": "minimax-m2.5",
}

# 添加到 API_KEY_MAP
API_KEY_MAP = {
    # ... 现有映射
    "CUSTOM_API_KEY": "custom",
}

# 添加到 PROVIDER_TARGET_MAP
PROVIDER_TARGET_MAP = {
    # ... 现有映射
    "custom": "custom",
}

# 在 _init_api_client() 中
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

# 在 get_model() 中
provider_env_map = {
    # ... 现有映射
    "custom": "CUSTOM_MODEL",
}

# 在 _call_api() 中
if self.provider in ("anthropic", "moonshot", "custom"):
    # 处理不同的内容块类型
    for block in response.content:
        if hasattr(block, "text") and block.text:
            return block.text
        elif hasattr(block, "thinking") and block.thinking:
            return block.thinking
```

### enhance_command.py

```python
def _get_api_keys() -> dict[str, str | None]:
    return {
        # ... 现有 keys
        "custom": os.environ.get("CUSTOM_API_KEY"),
    }

# 在 _pick_mode() 中
if api_keys["custom"]:
    return "api", "custom"
```

### enhance_skill.py

```python
# 处理自定义 provider
if args.target == "custom":
    _run_custom_enhancement(args, skill_dir)
    return
```

## 测试

### 单元测试

```bash
pytest tests/test_agent_client.py -v
```

### 手动测试

```bash
# 创建测试技能目录
mkdir -p /tmp/test-skill/references
echo "# Test Skill" > /tmp/test-skill/SKILL.md
echo "# Reference" > /tmp/test-skill/references/test.md

# 使用自定义 provider 运行增强
CUSTOM_BASE_URL="http://your-endpoint.example.com/api/v1" \
CUSTOM_API_KEY="your-key" \
CUSTOM_MODEL="minimax-m2.5" \
python3 -m skill_seekers.cli.main enhance /tmp/test-skill --target custom --api-key "your-key"
```

### 测试输出

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

## 构建和安装

```bash
# 构建包
cd /Users/duanmeng/github-repo/Skill_Seekers
python3 -m build

# 安装
pip install dist/skill_seekers-3.4.0-py3-none-any.whl --force-reinstall
```

### 打包后测试

测试打包安装后的包是否正确包含自定义 provider 功能：

```bash
# 确保删除旧的可执行文件（避免缓存问题）
rm -f ~/.local/bin/skill-seekers

# 验证 --target custom 选项可用
skill-seekers enhance --help | grep custom

# 设置环境变量并测试
export CUSTOM_BASE_URL="http://your-endpoint.com/api/v1"
export CUSTOM_API_KEY="your-api-key"
export CUSTOM_MODEL="minimax-m2.5"

skill-seekers enhance /path/to/skill --target custom --api-key "your-api-key"
```

**注意**：如果遇到 `invalid choice: 'custom'` 错误，检查是否有多余的可执行文件覆盖了安装的版本：
```bash
which -a skill-seekers
# 删除非预期路径的副本
```

## 注意事项

1. 自定义 provider 内部使用 **Anthropic SDK**，因此适用于任何 Anthropic 兼容 API
2. 认证头（`Authorization: Bearer` 和 `x-api-key`）会自动从 `CUSTOM_API_KEY` 生成
3. 模型名称应与 API 提供商期望的一致
4. 如果 API 返回 thinking blocks，代码会自动处理从中提取文本