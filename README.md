# 🤖 NoneBot2 插件开发技能指南

> 让 AI 助手能够快速、准确地编写 NoneBot2 插件的完整知识库。

## 📖 这是什么？

这是一份专为 AI 编程助手设计的 **NoneBot2 插件开发技能文件（SKILL.md）**，涵盖了从零开始编写 NoneBot2 插件所需的全部知识。

当你让 AI 助手"帮我写一个 NoneBot 插件"时，它可以加载这份技能文件，直接产出正确、可运行的插件代码。

## 🎯 覆盖内容

| 模块 | 说明 |
|------|------|
| 🏗️ 核心架构 | Driver / Adapter / Plugin / Matcher 关系 |
| 📦 插件结构 | 单文件插件、包插件、加载方式 |
| 🎯 事件响应器 | on_command / on_regex / on_keyword 等全部辅助函数 |
| ⚡ 事件处理 | handle / finish / send / pause / reject 流程控制 |
| 💉 依赖注入 | Bot / Event / State / CommandArg / Depends 等 |
| 💬 消息处理 | Message / MessageSegment / 消息模板 |
| 🔄 会话控制 | got / receive 多轮对话 |
| ⏰ 定时任务 | APScheduler 集成 |
| ⚙️ 配置管理 | .env + pydantic Config |
| 🔌 OneBot V11 | 事件类型 / 消息段 / Bot API / 权限控制 |
| 📝 完整示例 | 5 个可直接运行的插件示例 |
| ✅ 最佳实践 | 错误处理 / 日志 / 网络请求 / 数据持久化 |

## 🚀 如何使用

### 作为 OpenClaw / Codex 技能

将本仓库克隆到技能目录：

```bash
# OpenClaw
git clone https://github.com/MeowAndy/nonebot-plugin-skill.git ~/.openclaw/plugin-skills/nonebot-plugin-dev

# Codex
git clone https://github.com/MeowAndy/nonebot-plugin-skill.git ~/.codex/skills/nonebot-plugin-dev
```

之后 AI 助手在收到 NoneBot 插件开发相关请求时会自动加载此技能。

### 作为参考文档

直接阅读 [SKILL.md](./SKILL.md) 即可，内容按模块组织，包含大量代码示例。

## 📋 适用版本

- **NoneBot2**: v2.5.0+
- **Python**: >= 3.9
- **适配器**: 重点覆盖 OneBot V11，架构知识适用于所有适配器

## 🐺 作者

由 **小凌虾** 整理，基于 [NoneBot 官方文档](https://nonebot.dev/docs/) 提炼。

## 📄 License

MIT
