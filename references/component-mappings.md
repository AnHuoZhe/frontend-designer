# 功能类型 → 组件类型映射表

> 后端功能按用户行为分类 → 每类对应一组标准UI组件

## 映射规则

| 用户行为 | 功能类型 | shadcn/ui 组件 | 其他组件库对应 |
|---------|---------|---------------|-------------|
| 浏览一堆同类型条目 | 列表/表格 | Table, DataTable | Ant Table, MUI DataGrid |
| 搜索某一条目 | 搜索 | Command, Input | Ant AutoComplete, MUI Autocomplete |
| 按条件筛选 | 筛选器 | Select, Combobox, Tabs | Ant Select, MUI Filter |
| 查看单条详情 | 详情 | Card, Sheet(抽屉), Dialog | Ant Drawer, MUI Dialog |
| 填表单提交 | 创建/配置 | Form, Input, Textarea, Select | Ant Form, MUI Form |
| 开关/切换 | 切换 | Switch, Toggle, Checkbox | Ant Switch, MUI Switch |
| 触发操作 | 操作 | Button + Progress + Toast | Ant Button, MUI LoadingButton |
| 分步操作 | 向导 | Stepper, Steps | Ant Steps, MUI Stepper |
| 看统计数据 | 概览 | Card + 数值突出显示 | Ant Statistic, MUI Card |
| 看趋势变化 | 图表 | 第三方（Recharts/Chart.js） | Ant Charts, MUI Charts |
| 展开折叠内容 | 折叠 | Accordion, Collapsible | Ant Collapse, MUI Accordion |
| 导航 | 导航 | Sidebar, Breadcrumb, Tabs | Ant Menu, MUI Drawer |
| 多视图切换 | 切换视图 | Tabs, TabsList | Ant Tabs, MUI Tabs |
| 空状态 | 空提示 | 自定义Card + 图标 + 引导文字 | Ant Empty, MUI EmptyState |
| 加载中 | 加载 | Skeleton, Spinner | Ant Skeleton, MUI Skeleton |
| 错误提示 | 错误 | Alert, Toast | Ant Alert, MUI Alert |
| 确认操作 | 确认 | AlertDialog | Ant Modal.confirm, MUI Dialog |

## 组合模式

常见功能的组件组合：

| 功能 | 组件组合 |
|------|---------|
| 数据管理面板（CRUD） | Table + Dialog(Form) + Button + AlertDialog(删除确认) |
| 搜索+列表 | Command(搜索) + Tabs(分类) + Table(结果) |
| 仪表盘 | 多个 Card + 数值 + 图表 |
| 配置页 | Form 多段 + Accordion 折叠分组 |
| 详情页 | Card(主体) + Tabs(分段) + Accordion(折叠区域) |
| 分步向导 | Stepper + Form(每步) + Button(上一步/下一步) |

## 映射优先级

对每个 needs_ui=true 的后端功能：

1. 先判断用户行为（上面表第一列）
2. 查表获取组件类型
3. 如果行为模糊（一个功能可以做多件事），输出 2-3 种可选映射供用户选（在阶段B第4层处理）

## 反模式

- 一个页面混用两个组件库（如 shadcn Table + Ant Button）
- 功能不需要但硬塞组件（数据少于5条用Table→改用List）
- 忽略空状态/加载态/错误态设计
