# Frontend Designer

Codex 插件：后端就绪后，自动解析 API 与功能，并通过五层设计流程生成前端设计与实现方案。

## 安装

将本目录克隆或复制到 Codex 的 skills 目录，使 `SKILL.md` 可被 Codex 发现。

## 使用

在 Codex 中说“设计前端”，并提供后端项目路径。插件会先提取功能和 API 契约，再依次完成信息优先级、交互流程、空间分配、组件表达和视觉皮肤的三选一设计。

## 五层框架

```text
信息优先级 → 交互流程
                 ├→ 空间分配 ─┐
                 └→ 组件表达 ─┴→ 视觉皮肤
```

第3层与第4层可并行；视觉皮肤只依赖这两层的锁定结果。详见 [五层框架](references/layer-framework.md)。

## 与 forge 的关系

Frontend Designer 与 forge 是同级、独立的 Codex 插件：forge 编排全栈项目交付，Frontend Designer 只负责“后端已就绪 → 前端设计、实现与审查”的流程。

## 文档结构

- `SKILL.md`：主入口与完整流程
- `docs/`：架构、数据契约与需求说明
- `references/`：代码解析、五层框架、组件映射与完整示例
- `templates/`：阶段输出 JSON 模板

## 许可证

PolyForm Noncommercial。

本项目是纯文档型插件，不包含 `npm install`、构建或运行命令。
