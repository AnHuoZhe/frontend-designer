---
name: frontend-designer
description: 前端设计自动化。输入后端已就绪的项目，自动读代码提取功能，五层逐层设计，git worktree隔离生成，用户确认后合入。与forge同级，专注前端设计。
version: 1.1.0
metadata:
  hermes:
    tags: [frontend, design, ui, codex-plugin]
    paired_with: frontend-reviewer
---

# Frontend Designer

前端设计专家。从后端代码自动提取功能 → 逐层设计 → API Client生成 → 隔离实现 → 审查 → 合入。

## 触发条件

用户说"设计前端""生成前端""前端设计"且项目后端已就绪。

与 forge 的区别：forge 做全栈（需求到部署），frontend-designer 只做前端设计（后端已就绪→生成前端）。

## 核心流程

```
阶段A: 解析+澄清 → 阶段B: 五层设计 → 阶段C: API Client生成 → 阶段D: 实现 → 阶段E: 审查 → 阶段F: 合入
```

每阶段有结构化I/O。详细Schema见 `docs/data-contracts.md` 和 `templates/`。

---

## 阶段A：解析 + 需求澄清

**目标**：从后端代码提取功能清单，澄清用户设计需求。

**输入**：项目根目录路径

**步骤**：

1. 递归扫描后端代码。支持的框架：
   - FastAPI（Python）
   - Express（Node）
   - Flask（Python）
   - NestJS（TypeScript）
   
   提取：路由定义、请求参数（类型/必填/默认值）、响应类型、认证要求、分页方式、注释

2. 解析失败降级：提示"未检测到支持的后端框架，请手动提供API清单"，用户提供简化JSON

3. 输出 `feature-extraction.json`（写入项目 `.frontend-designer/` 目录）

4. 读 `~/.codex/frontend-preferences.json` 加载历史偏好

5. 追问用户——项目定位、目标用户、风格倾向、参考产品

6. 输出 `design-clarification.json`

完成标准：用户确认两个输出文件。

**失败处理**：解析无结果 → 降级提示；LLM不可用 → 提示稍后重试

---

## 阶段B：五层逐层设计

**目标**：从骨架到皮肤，逐层穿衣服。每层三选一。

**输入**：feature-extraction.json + design-clarification.json + frontend-preferences.json

**层级依赖（DAG）**：

```
第1层(信息优先级) ──→ 第2层(交互流程)
                          │
             ┌────────────┤
             ↓            ↓
        第3层(空间分配)  第4层(组件表达)
             │            │
             └─────┬──────┘
                   ↓
             第5层(视觉皮肤)
```

- 第1层选错 → 推倒全部
- 第2层选错 → 推倒第3/4层
- 第3层和第4层可并行生成
- 第3/4层选错 → 只改对应层和可能的第5层
- 第5层选错 → 只换肤
- 下层选择与上层冲突时 → 提示用户是否回退

**第1层：信息优先级**——用户第一眼最需要什么？哪些不可省、哪些折叠、哪些不出现。

**第2层：交互流程**——用户怎么在功能间移动？单页/多Tab/主从双区。

**第3层：空间分配**——屏幕空间怎么分。侧边栏宽度、卡片密度。

**第4层：组件表达**——同一功能用Table还是Card还是List。按钮位置、搜索框大小。

**第5层：视觉皮肤**——颜色、圆角、间距、字体。

每层输出 `layer-{N}-options.json`，用户选择后写入 `layer-decisions.json`（含每层选择的理由 `rationale`）。

**偏好自动学习**：
- 同一维度同一选择连续3次 → 自动设为默认，提示"已设为默认，本次是否沿用？"
- 用户选"本次不跳过" → 清零计数
- 分维度独立判定
- 超过30天未使用 → 所有默认清零

**失败处理**：
- 用户中断 → 已选层写入 `layer-decisions.json`（含 `incomplete: true`），下次恢复
- 三版差异不足 → 重试一次，仍不足合并为两版

---

## 阶段C：API Client 生成

**目标**：从解析出的API签名生成类型安全的TypeScript API client。

**输入**：feature-extraction.json + layer-decisions.json

**输出**：`src/api/` 目录
```
src/api/
├── types.ts       # 请求/响应 TypeScript 接口
├── client.ts      # fetch封装（含认证拦截器）
└── endpoints/     # 每个端点的调用函数
    ├── library.ts
    ├── analysis.ts
    └── platform.ts
```

**失败处理**：response_schema缺失 → 生成部分类型，标记TODO

---

## 阶段D：实现

**目标**：在git worktree中生成完整前端代码。

**步骤**：

1. **环境检查**：dirty working tree / 磁盘空间 / worktree路径冲突
2. **创建 worktree**：`git worktree add ../<project>-frontend feat/frontend-design`
3. **初始化项目**：Vite + React + Tailwind + shadcn/ui（根据第5层自动配置）
4. **安装组件**：`npx shadcn-ui@latest add` 按第4层选择安装
5. **生成代码**：复制api-client/ + 生成页面/布局/路由
6. **验证**：`npm install && npm run build`
7. **写入标记**：`.frontend-designer`（含完整 layer-decisions 和 design_commit）

**失败处理**：
- worktree创建失败 → 输出具体原因（dirty state/磁盘/路径冲突）
- npm install失败 → 检查Node版本+网络，重试一次
- npm run build失败 → 最多3次修复重试。3次后暂停，输出 `docs/build-errors.md`
- LLM/Codex不可用 → 保存中间状态，提示稍后

---

## 阶段E：审查

**目标**：验证生成代码的质量和完整性。

**审查项**：
1. 冒烟测试（`npm run build` 通过）
2. API对照（每个 needs_ui=true 的端点都有对应页面/组件）
3. 层级落实（对照 layer-decisions.json 逐层检查）
4. 组件一致性（不混用组件库）
5. 三态完整（空状态/加载态/错误态）

输出 `docs/frontend-review.md`，分必须改/建议改/可忽略。

**失败处理**：编译失败 → 标记必须改，不进行后续检查。API端点缺失 → 列出清单。

---

## 阶段F：合入

用户确认审查通过 → `git merge feat/frontend-design` → `git worktree remove ../project-frontend`

Commit message：`前端设计合入：{项目名} v{版本号}`

**失败处理**：merge冲突 → 列出冲突文件，用户选择策略

---

## 增量更新

触发：后端新增功能后，用户要求更新前端。

1. 读 `.frontend-designer` 获取 `design_commit` 和 `layers`
2. `git diff design_commit..HEAD` 识别：新增/修改/删除的API
3. 处理策略：
   - 新增 → 按已有设计模式生成新页面/组件
   - 修改 → 标记受影响的前端代码位置，最小化修改
   - 删除 → 列出前端死代码，用户选择保留或删除
4. worktree隔离 → 审查 → 合入
5. 更新 `.frontend-designer` 中的 `design_commit`

---

## 文件产出

- `.frontend-designer/feature-extraction.json` — 功能清单
- `.frontend-designer/design-clarification.json` — 设计需求
- `.frontend-designer/layer-decisions.json` — 五层决策（含理由）
- `.frontend-designer` — 项目标记文件（根目录）
- `docs/frontend-review.md` — 审查报告（worktree中）
- `~/.codex/frontend-preferences.json` — 用户偏好（跨项目）

---

## 前端偏好自动学习

`~/.codex/frontend-preferences.json` 跨项目累积。结构：

```json
{
  "layout": {"preferred": "sidebar-left", "history": ["sidebar-left","sidebar-left","sidebar-left"]},
  "color_schemes": {"preferred": null, "history": ["dark-slate", "dark-blue"]},
  "density": "compact",
  "component_kit": "shadcn-ui"
}
```

规则：连续3次同一选择→自动默认（分维度独立）。用户选"跳过"→清零。30天未用→全部清零。

## 参考文件

- `docs/architecture.md` — 完整架构文档（含错误处理总表）
- `docs/data-contracts.md` — 各阶段JSON Schema定义
- `references/component-mappings.md` — 功能类型→组件映射表
- `references/layer-framework.md` — 五层框架详解（含每层三版示例）
- `templates/` — 各阶段输出模板
