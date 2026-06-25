# Skill Registry

个人 AI Agent 技能注册表，由 [skillpod](https://github.com/anthropics/skillpod) 管理。本仓库作为技能发布源，供 Claude Code、Codex 等 AI Agent 工具安装使用。

## 已注册技能

| 技能 | 分类 | 说明 |
|------|------|------|
| `douyin-mini-game` | 游戏开发 | 抖音小游戏 JS API 完整参考，覆盖 `tt.*` 前端接口 |
| `douyin-mini-game-server-api` | 后端开发 | 抖音小游戏服务端 OpenAPI，覆盖 `mgplatform/api` 接口 |
| `douyin-mini-game-unity` | 游戏开发 | 抖音小游戏 Unity C# API 参考，覆盖 `TT.*` SDK |
| `element-ui-vue2` | 前端 UI | Element UI Vue 2.x 桌面端组件库 |
| `element-plus-ui-vue3` | 前端 UI | Element Plus Vue 3.x 桌面端组件库（Composition API） |
| `java-design-patterns` | 软件架构 | Java 设计模式权威参考，182 个模式 + 26 个设计原则 |
| `uview-ui` | 前端 UI | uView UI uni-app 跨平台组件库（Vue 2） |

## 安装

```bash
# 通过 skillpod CLI 安装
skillpod install <skill-name>

# 或直接从仓库克隆到本地 skills 目录
git clone https://github.com/<your-account>/skills.git ~/.claude/skills/
```

## 目录结构

```
registry/
├── README.md              # 本文件
└── skills/
    ├── douyin-mini-game/
    ├── douyin-mini-game-server-api/
    ├── douyin-mini-game-unity/
    ├── element-plus-ui-vue3/
    ├── element-ui-vue2/
    ├── java-design-patterns/
    └── uview-ui/
```

## 贡献

此仓库为个人维护的技能注册表。如需新增技能或提交改进，请通过 PR 方式提交。

## 安全声明

部分技能（如 `douyin-mini-game-unity`、`douyin-mini-game`）包含支付、登录等敏感 API 示例代码，已标注安全警告。使用前请务必审查代码，确保服务端验证、日志脱敏、调试命令禁用等安全措施到位。
