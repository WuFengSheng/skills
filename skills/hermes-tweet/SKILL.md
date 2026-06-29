---
name: hermes-tweet
description: "Hermes Agent native X/Twitter plugin. Invoke when a Hermes workflow needs X search, account reads, social listening, trends, monitors, giveaway audits, or explicitly approved X actions through Xquik."
---

# Hermes Tweet 技能

Hermes Tweet 是面向 Hermes Agent 的原生 X/Twitter 插件。它通过 Xquik 提供目录化
API 路由，让 Agent 先发现端点，再执行受控读取或经过审批的动作。

## 触发条件

**在以下场景中自动调用此技能：**

1. Hermes Agent 工作流需要搜索 X/Twitter 内容、读取账号资料、查看推文详情或趋势时
2. 需要做社交监听、品牌研究、创作者研究、发布监控、社区审计或抽奖审计时
3. 需要把 X/Twitter 自动化限制在已登记的 Xquik API 路由内时
4. 需要区分只读操作和会改变账号或工作流状态的动作时
5. 用户明确要求通过 Hermes Tweet、Xquik、`tweet_explore`、`tweet_read` 或 `tweet_action` 完成任务时

## 安装

在 Hermes Agent 环境中安装并启用插件：

```bash
hermes plugins install Xquik-dev/hermes-tweet --enable
```

如果从 PyPI 安装：

```bash
uv pip install --python ~/.hermes/hermes-agent/venv/bin/python hermes-tweet
hermes plugins enable hermes-tweet
```

配置运行时环境变量：

```bash
export XQUIK_API_KEY="your_xquik_api_key"
export HERMES_TWEET_ENABLE_ACTIONS="false"
```

不要把 API key 写入提示词、Issue、PR、日志或工具参数。

## 工具

| 工具 | 用途 | 默认安全性 |
|------|------|------------|
| `tweet_explore` | 搜索 Hermes Tweet 内置 Xquik 端点目录，不发起外部 API 调用 | 始终可用 |
| `tweet_read` | 调用目录中登记的只读端点 | 需要 `XQUIK_API_KEY` |
| `tweet_action` | 调用写入、私有读取、监控、webhook、抽取任务、媒体或抽奖动作端点 | 需要 `XQUIK_API_KEY` 与 `HERMES_TWEET_ENABLE_ACTIONS=true` |

## 工作流

1. 先用 `tweet_explore` 搜索能力或端点。
2. 如果端点是只读 `GET`，再用 `tweet_read`。
3. 如果端点会改变账号或工作流状态，先说明端点和 payload，再确认用户已经批准。
4. 只有在 `HERMES_TWEET_ENABLE_ACTIONS=true` 时才使用 `tweet_action`。
5. 如果缺少 `XQUIK_API_KEY`，只要求用户在 Hermes 运行时环境中配置，不要求用户在对话中提供密钥值。

## 安全规则

- 不要直接创建 HTTP fallback。只使用 Hermes Tweet 暴露的目录化工具。
- 不要猜测未登记的 `/api/v1/...` 路径。
- 不要请求、输出、记录或缓存 API key、cookie、密码、TOTP 或签名密钥。
- 默认保持 `tweet_action` 关闭，除非工作流有明确审批步骤。
- 对无人值守、定时任务或 gateway 场景优先使用 `tweet_read`。

## 参考

- GitHub: https://github.com/Xquik-dev/hermes-tweet
- PyPI: https://pypi.org/project/hermes-tweet/
