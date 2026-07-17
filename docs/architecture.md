# Frontend Designer — 架构文档

> 版本 1.1 | 2026-07-17 | 审查修订

## 一、定位

Codex 插件。输入一个后端已就绪的项目 → 自动读代码提取功能 → 逐层设计前端 → 在 git worktree 隔离区生成代码 → 审查 → 用户确认后合入 main。

与 forge 同级独立项目。协议、审查体系、中期记忆机制与 forge 一致。

## 二、核心流程与数据契约

```
阶段A: 解析+澄清 → 阶段B: 五层设计 → 阶段C: API Client生成 → 阶段D: 实现 → 阶段E: 审查 → 阶段F: 合入
```

每阶段定义结构化 I/O，不依赖自然语言黑盒传递。

### 数据契约总览

| 阶段 | 输入 | 输出 |
|------|------|------|
| A 解析+澄清 | 项目路径 | `feature-extraction.json` + `design-clarification.json` |
| B 五层设计 | A的输出 + frontend-preferences.json | `layer-decisions.json`（每层含理由） |
| C API Client | feature-extraction.json + layer-decisions.json | `api-client/` 目录（TypeScript类型+调用函数） |
| D 实现 | B+C的输出 | worktree中的完整前端代码 |
| E 审查 | D的代码 + B+C的输出 | `frontend-review.md` |
| F 合入 | E通过 | merged main + 更新 `.frontend-designer` |

---

### 阶段A：解析 + 需求澄清

**输入**：项目根目录路径

**步骤**：

1. 递归扫描后端代码，提取：
   - 路由定义（FastAPI/Express/Flask/NestJS）
   - 请求参数（类型、必填、默认值）
   - 响应类型（Pydantic模型、TypeScript接口、dataclass）
   - 认证要求（JWT/OAuth/API Key）
   - 分页方式（offset-limit/cursor）
   - 注释和docstring

2. 解析失败降级：
   - 未检测到支持的后端框架 → 提示"未检测到支持的后端框架，请手动提供API清单"
   - 手动降级路径：用户提供简化JSON `[{name, endpoint, method, params, response}]`

3. 输出 `feature-extraction.json`：
```json
{
  "project": "death-replay-agent",
  "framework": "FastAPI",
  "features": [
    {
      "name": "Demo库列表",
      "endpoint": "GET /api/library",
      "method": "GET",
      "request_schema": {
        "query_params": [
          {"name": "search", "type": "string", "required": false},
          {"name": "platform", "type": "string", "required": false},
          {"name": "page", "type": "int", "required": false, "default": 1}
        ]
      },
      "response_schema": {
        "type": "paginated_list",
        "item_type": "DemoSummary",
        "fields": ["id", "map", "date", "kd_ratio", "platform"]
      },
      "auth_required": true,
      "pagination": "offset-limit",
      "user_action": "浏览、搜索、筛选demo",
      "needs_ui": true,
      "ui_pattern": "table_with_search_filter"
    }
  ],
  "hidden_features": ["GET /health", "POST /admin/migrate"]
}
```

4. 读 `~/.codex/frontend-preferences.json` 加载历史偏好

5. 追问用户：
   - 项目定位（内部工具/对外产品/管理面板/数据看板）
   - 目标用户画像
   - 风格倾向（极简/专业/活泼/暗色系）
   - 参考产品

6. 输出 `design-clarification.json`：
```json
{
  "project_type": "data-dashboard",
  "target_user": {"role": "玩家", "tech_level": "low", "frequency": "weekly"},
  "style_preference": "dark-professional",
  "reference_products": ["leetify"],
  "tech_stack": {"framework": "react", "styling": "tailwind", "component_kit": "shadcn-ui"},
  "confirmed_at": "2026-07-17T19:00:00Z"
}
```

完成标准：用户确认 feature-extraction.json（功能清单无误）和 design-clarification.json（设计需求确认）。

**失败模式**：
- 解析无结果 → 触发降级提示，用户手动提供API清单
- LLM不可用 → 提示用户稍后重试，不写入任何文件

---

### 阶段B：五层逐层设计

**输入**：feature-extraction.json + design-clarification.json + frontend-preferences.json

每层生成三版选项，用户选一。上层选定后下层在锁定约束下生成。

**层级依赖模型（有向无环图）**：

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

- 第1层选错 → 推倒全部，从第2层重新生成
- 第2层选错 → 推倒第3和/或第4层（取决于冲突程度）
- 第3层和第4层可并行生成（互不依赖）
- 第3或第4层选错 → 只改对应层和可能的第5层
- 第5层选错 → 只换肤
- 当下层选择与上层冲突时 → 提示用户，建议回退调整上层

每层输出格式（以 `layer-1-options.json` 为例）：
```json
{
  "layer": 1,
  "name": "信息优先级",
  "options": [
    {
      "id": "data-first",
      "label": "数据优先",
      "description": "首屏展示统计概览卡片，最近demo列表折叠在下方",
      "ascii_layout": "...",
      "suitable_for": "用户主要目的是看数据趋势，偶尔看单场详情"
    }
    // ...另外两版
  ]
}
```

用户选择后写入 `layer-decisions.json`：
```json
{
  "layers": {
    "1": {
      "id": "data-first",
      "rationale": "用户说'主要是看自己最近打得怎么样'"
    },
    "2": {"id": "master-detail", "rationale": "浏览demo库→选中看详情，最自然的交互顺序"},
    "3": {"id": "sidebar-left-narrow", "rationale": "导航项少（3个），不需要宽侧边栏"},
    "4": {"id": "table-cards-mix", "rationale": "列表用Table方便排序比较，详情用Card"},
    "5": {"id": "dark-slate-pink", "rationale": "参考Leetify配色，暗色专业感"}
  }
}
```

偏好自动学习：
- 用户选中某选项 → 写入 frontend-preferences.json 的对应维度 history
- 同一维度同一选择连续3次 → 该维度自动设为默认，提示用户"检测到你连续3次选择X，已设为默认。本次是否沿用？"
- 用户选"本次不跳过" → 清零计数，重新累积
- 分维度独立判定：layout默认了不代表color_scheme也被默认
- 超过30天未使用 → 所有默认清零，重新累积

**失败模式**：
- 用户中断会话 → 已选层写入 `layer-decisions.json`（加上 `incomplete: true`），下次继续时从中断层恢复
- LLM生成的三版差异不足 → 重试一次，仍不足则合并为两版，标注"差异较小"

---

### 阶段C：API Client 生成

**输入**：feature-extraction.json + layer-decisions.json

从解析出的API签名直接生成类型安全的TypeScript API client，而非让LLM在实现阶段根据自然语言重新推断API调用方式。

**输出**：`src/api/` 目录
```
src/api/
├── types.ts       # 请求/响应 TypeScript 接口定义
├── client.ts      # fetch/axios 封装（含认证拦截器）
└── endpoints/     # 每个端点的调用函数
    ├── library.ts
    ├── analysis.ts
    └── platform.ts
```

**生成规则**：
- 从 `feature-extraction.json` 的 `request_schema` 生成 TypeScript 接口
- 从 `response_schema` 生成返回类型
- `auth_required=true` 的端点 → 自动添加 Authorization header
- `pagination` 字段 → 生成分页参数类型

**失败模式**：
- 解析输出不完整（缺少 response_schema）→ 标记为 `// TODO: 响应类型未解析`，生成部分类型

---

### 阶段D：实现

**输入**：layer-decisions.json + api-client/ 目录

**步骤**：

1. 环境检查：
   - 检查 git working tree 是否 dirty → 提示用户先 commit 或 stash
   - 检查磁盘空间 → 不足时拒绝创建 worktree
   - 检查 worktree 路径是否已存在 → 提示清理或换名

2. 创建 git worktree：
   ```
   git worktree add ../<project>-frontend feat/frontend-design
   ```

3. 初始化前端项目：
   - `npm create vite@latest . -- --template react-ts`
   - `npm install tailwindcss @tailwindcss/vite`
   - `npx shadcn-ui@latest init`（根据第5层选择自动配置：暗色主题、slate基础色、css变量模式）
   - `npx shadcn-ui@latest add` 按第4层组件选择安装对应组件

4. 生成代码：
   - 复制 api-client/ 到 worktree 的 src/api/
   - 生成页面组件、布局组件、路由配置
   - 第5层配色写入 tailwind.config

5. 验证：
   - `npm install && npm run build`（编译验证）
   - 编译失败 → 最多重试3次。3次仍失败 → 标记失败组件，输出错误日志到 `docs/build-errors.md`，暂停并告知用户

6. 写入 `.frontend-designer` 标记文件：
```json
{
  "version": "1.0",
  "design_commit": "<HEAD commit hash>",
  "design_date": "2026-07-17T19:00:00Z",
  "layers": {
    "1": {"id": "data-first", "rationale": "..."},
    "2": {"id": "master-detail", "rationale": "..."},
    "3": {"id": "sidebar-left-narrow", "rationale": "..."},
    "4": {"id": "table-cards-mix", "rationale": "..."},
    "5": {"id": "dark-slate-pink", "rationale": "..."}
  }
}
```

**失败模式**：
- worktree创建失败 → 检查dirty state/磁盘/路径冲突，输出具体原因
- npm install失败 → 检查Node版本和网络，重试一次
- npm run build失败 → 最多3次修复+重试。3次后暂停，输出错误日志
- LLM/Codex API不可用 → 保存中间状态，提示稍后重试

---

### 阶段E：审查

**输入**：worktree中的完整前端代码 + layer-decisions.json + feature-extraction.json

**审查项**：

1. **冒烟测试**：每个页面至少能渲染，不报错（`npm run build` 通过即视为通过）
2. **API对照**：feature-extraction.json 中每个 needs_ui=true 的端点都有对应的页面/组件
3. **层级落实**：对照 layer-decisions.json 逐层检查——
   - 第1层：信息优先级是否在页面上体现
   - 第2层：交互流程是否正确（单页/多Tab/主从双区）
   - 第3层：空间分配是否正确（侧边栏宽度/卡片密度）
   - 第4层：组件选择是否正确（Table/Card/List）
   - 第5层：配色是否匹配
4. **组件一致性**：不混用多个组件库
5. **空状态**：列表为空时是否有引导文字
6. **加载态**：是否有加载指示
7. **错误态**：API调用失败是否有提示

审查报告写入 worktree 的 `docs/frontend-review.md`，分必须改/建议改/可忽略。

**失败模式**：
- 编译失败 → 审查直接标记为必须改，不进行后续检查
- API端点缺失 → 列出缺失端点清单

---

### 阶段F：合入

用户确认审查通过 → `git merge feat/frontend-design` → `git worktree remove ../project-frontend`

合入后 commit message 格式：`前端设计合入：{项目名} v{版本号}`

**失败模式**：
- merge冲突 → 暂停，列出冲突文件，让用户手动解决或指定策略（保留前端/保留原有）

---

## 三、增量更新

当后端新增功能后触发。流程：

1. 读 `.frontend-designer` 获取 `design_commit` 和 `layers`

2. `git diff design_commit..HEAD` 识别变更：
   - **新增 API**：新的路由/端点
   - **修改 API**：已有端点签名变更（参数增删、类型变化、响应结构变）
   - **删除 API**：端点已移除

3. 变更分析：
   - 新增 → 按已有设计模式（读 layer-decisions.json）生成对应页面/组件
   - 修改 → 标记受影响的前端代码位置（API client函数、页面组件），生成修改建议
   - 删除 → 列出对应的前端死代码位置，在审查报告中标注为待清理。用户选择保留（降级为静态页）或删除

4. **最小化修改已有部分**——不是不改已有部分，而是：
   - 导航菜单/侧边栏：必须更新（新增入口）
   - 路由注册：必须更新
   - API client类型定义：必须更新（响应签名变更时）
   - 已有页面组件：尽量不改，仅当API签名变更导致编译错误时修改

5. worktree 隔离生成 → 审查（与首次设计相同） → 合入

6. 更新 `.frontend-designer` 中的 `design_commit`

---

## 四、错误处理与恢复总表

| 阶段 | 失败场景 | 处理 |
|------|---------|------|
| A | 解析不到框架 | 提示手动提供API清单JSON |
| A | LLM不可用 | 保存需求描述，提示稍后重试 |
| B | 用户中断会话 | 已选层写入layer-decisions.json(标记incomplete)，恢复时从中断层继续 |
| B | 三版差异不足 | 重试一次，仍不足则合并为两版 |
| C | response_schema缺失 | 生成部分类型，标记TODO |
| D | worktree创建失败 | 检查dirty state/磁盘/路径冲突，输出原因 |
| D | npm install失败 | 检查Node版本+网络，重试一次 |
| D | npm run build失败 | 最多3次修复重试。3次后暂停，输出build-errors.md |
| D | LLM/Codex不可用 | 保存中间状态，提示稍后 |
| E | 编译失败 | 标记为必须改，不进行后续检查 |
| E | API端点缺失 | 列出缺失端点清单 |
| F | merge冲突 | 列出冲突文件，让用户选择策略 |

---

## 五、文件结构

### Codex 插件（frontend-designer/）

```
frontend-designer/
├── SKILL.md                      # 主技能入口
├── docs/
│   ├── architecture.md           # 本架构文档
│   ├── layer-framework.md        # 五层设计框架详解
│   └── data-contracts.md         # 各阶段JSON Schema定义
├── references/
│   ├── component-mappings.md     # 功能类型→组件映射表
│   ├── code-parsing-guide.md     # 后端代码解析指南
│   └── design-examples.md        # 完整设计示例
├── templates/
│   ├── feature-extraction.json   # 阶段A输出模板
│   ├── design-clarification.json # 阶段A输出模板
│   ├── layer-options.json        # 阶段B输出模板
│   └── layer-decisions.json      # 阶段B输出模板
└── LICENSE                       # PolyForm Noncommercial
```

### 项目标记文件（.frontend-designer）

写入被设计项目的根目录。含完整的层级选择和设计理由。

### 用户偏好文件（frontend-preferences.json）

路径：`~/.codex/frontend-preferences.json`（与 forge 独立，不绑定项目目录结构）

---

## 六、默认技术栈

| 层 | 默认选择 | 1.0阶段支持 |
|----|---------|------------|
| 框架 | React + Vite | 仅此 |
| 样式 | Tailwind CSS | 仅此 |
| 组件库 | shadcn/ui | 仅此 |
| 图标 | Lucide React | 仅此 |

1.0 聚焦 shadcn/ui，不展开多组件库支持。扩展放在后续版本。

---

## 七、与 forge 的边界

- forge：全栈项目从零到一（需求→架构→后端→前端→部署）
- frontend-designer：只做前端设计（后端已就绪→生成前端）

合并时机：独立演进。当两者稳定且出现"锻造新项目→直接调前端设计器"的稳定需求模式时再考虑。
