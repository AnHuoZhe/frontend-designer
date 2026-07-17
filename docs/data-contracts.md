# 数据契约 — 各阶段JSON Schema定义

> 每个阶段的输入输出结构化格式。消除自然语言黑盒传递。

## 阶段A 输出

### feature-extraction.json

```json
{
  "$schema": "feature-extraction.schema.json",
  "project": "string — 项目名（从目录名或package.json读取）",
  "framework": "string — FastAPI | Express | Flask | NestJS | unknown",
  "detected_at": "ISO8601 — 解析时间",
  "features": [
    {
      "name": "string — 功能名称（人话）",
      "endpoint": "string — 如 GET /api/library",
      "method": "string — GET | POST | PUT | DELETE | PATCH",
      "request_schema": {
        "path_params": [
          {"name": "string", "type": "string", "required": true}
        ],
        "query_params": [
          {"name": "string", "type": "string", "required": false, "default": "any"}
        ],
        "body_schema": "object | null — 请求体类型定义"
      },
      "response_schema": {
        "type": "string — single | list | paginated_list",
        "item_type": "string — 响应条目类型名",
        "fields": ["string — 字段名列表"]
      },
      "auth_required": "boolean",
      "pagination": "string — offset-limit | cursor | none",
      "user_action": "string — 用户行为描述（如'浏览、搜索、筛选demo'）",
      "needs_ui": "boolean — 是否需要前端界面",
      "ui_pattern": "string — 推荐UI模式（如'table_with_search_filter'）"
    }
  ],
  "hidden_features": ["string — 不需要UI的端点（如健康检查）"],
  "warnings": ["string — 解析时遇到的问题（如缺少response_schema的端点）"]
}
```

### design-clarification.json

```json
{
  "$schema": "design-clarification.schema.json",
  "project_type": "string — internal-tool | public-product | admin-panel | data-dashboard | other",
  "target_user": {
    "role": "string — 谁在用（如'FPS玩家'）",
    "tech_level": "string — low | medium | high",
    "frequency": "string — daily | weekly | monthly"
  },
  "style_preference": "string — minimal | professional | playful | dark | custom",
  "reference_products": ["string — 参考产品名"],
  "tech_stack": {
    "framework": "react",
    "styling": "tailwind",
    "component_kit": "shadcn-ui"
  },
  "confirmed_at": "ISO8601",
  "notes": "string — 用户额外说明"
}
```

## 阶段B 输出

### layer-{N}-options.json（每层生成三版时）

```json
{
  "$schema": "layer-options.schema.json",
  "layer": "int — 1到5",
  "name": "string — 层名（信息优先级/交互流程/空间分配/组件表达/视觉皮肤）",
  "decision_question": "string — 这层要回答的问题",
  "locked_from_previous": {
    "layer_1": "string | null",
    "layer_2": "string | null",
    "layer_3": "string | null",
    "layer_4": "string | null"
  },
  "options": [
    {
      "id": "string — 唯一标识（如data-first）",
      "label": "string — 人话标签（如'数据优先'）",
      "description": "string — 详细描述，包含布局逻辑",
      "ascii_layout": "string — ASCII布局图",
      "suitable_for": "string — 适合什么场景",
      "preference_match": "boolean — 是否匹配用户历史偏好"
    }
  ]
}
```

### layer-decisions.json（用户选定后，增量更新）

```json
{
  "$schema": "layer-decisions.schema.json",
  "complete": "boolean — 是否全部五层已选定",
  "layers": {
    "1": {
      "id": "string — 选项id",
      "label": "string — 选项标签",
      "rationale": "string — 用户为什么选这个（记录当时的理由）",
      "selected_at": "ISO8601"
    }
  }
}
```

如果用户中断：
```json
{
  "complete": false,
  "incomplete": true,
  "last_completed_layer": 2,
  "layers": { ... }
}
```

## 阶段C 输出

### API Client 目录结构

```
src/api/
├── types.ts       # TypeScript接口（从response_schema和request_schema生成）
├── client.ts      # fetch封装，自动处理：base URL、auth header、错误转换
└── endpoints/
    └── {module}.ts  # 每个功能模块一个文件，导出async函数
```

types.ts 示例：
```typescript
// 自动从 feature-extraction.json 生成
export interface DemoSummary {
  id: string;
  map: string;
  date: string;
  kd_ratio: number;
  platform: string;
}

export interface PaginatedResponse<T> {
  items: T[];
  total: number;
  page: number;
}
```

client.ts 示例：
```typescript
const BASE_URL = import.meta.env.VITE_API_URL || 'http://localhost:8000';

async function request<T>(path: string, options?: RequestInit): Promise<T> {
  const res = await fetch(`${BASE_URL}${path}`, {
    ...options,
    headers: {
      'Content-Type': 'application/json',
      ...(localStorage.getItem('apiKey') ? { 'Authorization': `Bearer ${localStorage.getItem('apiKey')}` } : {}),
      ...options?.headers,
    },
  });
  if (!res.ok) throw new Error(`API error: ${res.status}`);
  return res.json();
}
```

## 项目标记文件

### .frontend-designer（被设计项目的根目录）

```json
{
  "version": "1.0",
  "design_commit": "string — HEAD commit hash at design time",
  "design_date": "ISO8601",
  "layers": {
    "1": {"id": "string", "rationale": "string"},
    "2": {"id": "string", "rationale": "string"},
    "3": {"id": "string", "rationale": "string"},
    "4": {"id": "string", "rationale": "string"},
    "5": {"id": "string", "rationale": "string"}
  }
}
```

## 用户偏好文件

### ~/.codex/frontend-preferences.json

```json
{
  "layout": {
    "preferred": "string | null",
    "history": ["string"],
    "consecutive_count": "int"
  },
  "interaction_flow": {
    "preferred": "string | null",
    "history": ["string"],
    "consecutive_count": "int"
  },
  "space_allocation": {
    "preferred": "string | null",
    "history": ["string"],
    "consecutive_count": "int"
  },
  "component_expression": {
    "preferred": "string | null",
    "history": ["string"],
    "consecutive_count": "int"
  },
  "color_schemes": {
    "preferred": "string | null",
    "history": ["string"],
    "consecutive_count": "int"
  },
  "density": "string — compact | comfortable | spacious",
  "component_kit": "string — shadcn-ui",
  "last_used": "ISO8601"
}
```
