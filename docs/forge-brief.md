# Frontend Designer — Codex 实现需求

## 项目定位

Codex插件。与forge同级独立项目。输入后端已就绪的项目 → 自动读代码提取功能 → 五层逐层设计 → git worktree隔离生成 → 审查 → 合入main。

**本项目是纯文档型Codex插件**——只包含SKILL.md和参考markdown文件，无需npm/node/任何运行时。不要创建package.json或前端脚手架。

## 前置步骤

在动手之前，先读以下已有文件，确保新文件与已有设计一致：
- `SKILL.md`
- `docs/architecture.md`
- `docs/data-contracts.md`

## 项目目录

E:\work\word\study\frontend-designer\

## 已完成的文件（不要改）

- `SKILL.md` — 主技能入口，完整流程
- `docs/architecture.md` — 架构文档V1.1（含数据契约、错误处理、DAG依赖）
- `docs/data-contracts.md` — 各阶段JSON Schema定义
- `references/component-mappings.md` — 16种功能类型→组件映射表
- `references/layer-framework.md` — 五层框架详解（需要更新依赖模型，见任务1）
- `LICENSE` — PolyForm Noncommercial

## 需要做的事

### 1. 更新 references/layer-framework.md

当前文档只在"框架原理"段落提了一句"逐层收敛"，没有独立的依赖关系章节。

**改动位置**：在"框架原理"段落之后、第1层详解之前，新增一个"## 层级依赖关系"章节。

**内容**：把线性模型改为DAG模型——

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

增加说明：
- 第3层和第4层可并行生成（互不依赖）
- 第2层选错可能只推倒第3层或第4层，取决于冲突程度
- 下层选择与上层冲突时→提示用户回退调整上层
- 第5层仅依赖第3和第4层的锁定结果，不直接依赖第1和第2层

用death-replay-agent的实例标注每层的输入来自哪层（如第3层的三版选项是在第2层选定"主从双区"后才生成的）。

### 2. 创建 references/code-parsing-guide.md

后端代码解析指南。内容：

- FastAPI解析规则：读@app.get/post/put/delete装饰器，读函数签名的参数类型注解，读Pydantic BaseModel子类作为response_model
- Express解析规则：读router.get/post/put/delete，读req.params/req.query/req.body，读res.json的返回结构
- Flask解析规则：读@app.route装饰器的methods参数
- NestJS解析规则：读@Controller/@Get/@Post装饰器，读DTO类
- 通用降级：扫描src/或app/目录下所有文件，搜'@app.'或'router.'字符串
- 识别认证：搜'Authorization'/'Bearer'/'JWT'/'API.?[Kk]ey'字样
- 识别分页：搜'page'/'limit'/'offset'/'cursor'参数
- 字段去重：同一个endpoint在不同文件出现时合并
- needs_ui判断规则：GET读操作且返回用户可见数据→true；POST/PUT/DELETE且用户触发→true；/health /admin /internal路径→false；返回HTML而非JSON→false
- 输出格式必须匹配 `docs/data-contracts.md` 中 feature-extraction.json 的Schema

### 3. 创建 references/design-examples.md

完整设计示例——以death-replay-agent为例走一遍五层。

**功能清单**（从后端提取）：
- Demo库列表：GET /api/library → table_with_search_filter
- 平台目录：GET /api/platforms → checkbox_list
- 分析配置：POST /api/config → form
- 触发分析：POST /api/library/{id}/analyze → button_with_progress
- 分析结果：GET /api/analysis/{id} → card_with_accordion

**每层三版**：
- 第1层——数据优先/操作优先/概览优先（各附ASCII布局简图）
- 第2层——单页滚动/多Tab/主从双区（各附页面跳转关系图）
- 第3层——窄侧边栏(250px)/宽侧边栏(380px)/无侧边栏顶部导航（各附空间切分图）
- 第4层——以Demo列表为例：Table表格/Card卡片/List列表（各附组件排列图）
- 第5层——三版配色方案，每版列出所有色值：

```
A版 暗色冷调：
  背景: slate-950/900, 卡片: slate-800, 强调: cyan-400
  文字: slate-100/400, 边框: slate-700, 圆角: rounded-lg (8px)

B版 暗色暖调：
  背景: neutral-950/900, 卡片: neutral-800, 强调: amber-400
  文字: neutral-100/400, 边框: neutral-700, 圆角: rounded-lg (8px)

C版 暗色品牌调：
  背景: stone-950, 卡片: stone-800, 强调: pink-500 + cyan-400(辅助)
  文字: stone-100/400, 边框: stone-700, 圆角: rounded-xl (12px)
```

**用户选择路径**：数据优先→主从双区→窄侧边栏→Table+Cards混合→暗色品牌调

**最终完整ASCII布局图**——把所有层级选择综合成一个完整的页面布局。

### 4. 创建 templates/ 目录和四个JSON模板文件

参考 `docs/data-contracts.md` 中的Schema定义。

用**描述性字符串值**标注字段含义，不要用JSON注释（JSON不支持注释）。格式如：
```json
{
  "project_type": "internal-tool | public-product | admin-panel | data-dashboard",
  "target_user": {"role": "谁在用", "tech_level": "low | medium | high"}
}
```

四个模板：
- `templates/feature-extraction.json`
- `templates/design-clarification.json`
- `templates/layer-options.json`
- `templates/layer-decisions.json`

### 5. 创建 README.md

- 项目名称：Frontend Designer
- 一句话描述：Codex插件，后端就绪→自动前端设计
- 安装方式：克隆到Codex技能目录
- 使用方式：在Codex中说"设计前端"并指定项目路径
- 五层框架简图（ASCII）
- 与forge的关系说明（同级独立项目，forge做全栈，本工具只做前端设计）
- 许可证：PolyForm Noncommercial
- 不包含npm install/build/run指令（纯文档项目）

---

## 实现要求

1. 直接改文件，一步到位，不要先出设计说明再确认
2. 本项目是纯文档——只写markdown和JSON模板，不创建任何代码/配置/脚手架
3. git commit格式：中文，简要描述做了什么
4. 所有新文件写完后，用 `ls -R` 确认目录结构完整
5. 完成后在项目目录下 `git init && git add -A && git commit -m "第一版，前端设计自动化Codex插件"`
