# 后端代码解析指南

本指南用于阶段 A 从后端代码生成 `feature-extraction.json`。输出必须符合 [数据契约](../docs/data-contracts.md) 中 `feature-extraction.json` 的 Schema；不能确认的字段保留为空值或标注解析来源，不得臆测。

## 扫描范围与通用规则

1. 优先递归扫描 `src/`、`app/`、`api/`、`routes/` 与框架惯用目录；未识别框架时，扫描项目内所有源码文件，并搜索 `@app.`、`router.`、`@Controller` 等路由特征。
2. 每个端点以 `HTTP 方法 + 规范化路径` 去重。同一端点出现在路由文件、控制器与测试中时，合并可互补的字段；冲突时以实际路由定义为准，并记录无法消解的冲突。
3. 从 `Authorization`、`Bearer`、`JWT`、`API.?[Kk]ey` 及认证中间件/依赖识别 `auth_required`。
4. 从 `page`、`limit`、`offset`、`cursor` 参数及返回体中的分页元数据识别 `pagination`。
5. 返回 HTML、文件流或重定向的端点不作为 JSON 数据功能；只有用户可见的 JSON 数据端点才进入功能提取结果。

## FastAPI

- 读取 `@app.get`、`@app.post`、`@app.put`、`@app.delete`，以及 `APIRouter` 上的同名装饰器；合并 router 前缀。
- 从函数签名读取 path/query/body 参数的类型注解、默认值和是否必填；`Query`、`Path`、`Body`、`Depends` 提供额外约束与认证线索。
- 从 `response_model` 及其 Pydantic `BaseModel` 子类递归提取响应字段；列表、分页包装模型和可选字段应保留原有层级。

## Express

- 读取 `router.get/post/put/delete`、`app.get/post/put/delete` 及 router 挂载前缀。
- 读取 `req.params`、`req.query`、`req.body` 的访问点与 TypeScript 类型；将它们分别映射到 path、query、body 请求结构。
- 从 `res.json(...)` 的对象、数组及其上游类型/序列化器推断返回结构；若只能发现动态对象，保留已知字段并标记不完整。

## Flask

- 读取 `@app.route` 的路径和 `methods` 参数；未写 `methods` 时按 Flask 默认 `GET` 处理。
- 同时识别 Blueprint 路由和注册前缀。
- 从函数参数、`request.args`、`request.view_args`、`request.get_json()` 与 `jsonify(...)` 提取请求和响应结构。

## NestJS

- 读取 `@Controller` 与 `@Get`、`@Post`、`@Put`、`@Delete` 装饰器，拼接控制器和方法路径。
- 从 `@Param`、`@Query`、`@Body` 及 DTO 类提取请求字段、类型和校验约束。
- 从方法返回类型、`@ApiResponse`、序列化 DTO 和 service 返回对象提取响应结构。

## `needs_ui` 判定

| 条件 | `needs_ui` |
| --- | --- |
| GET 读取操作，且返回用户可见 JSON 数据 | `true` |
| POST / PUT / DELETE，由用户在界面中触发 | `true` |
| 路径含 `/health`、`/admin`、`/internal` | `false` |
| 返回 HTML 而非 JSON | `false` |

无法判断时，保守设为 `false`，并将端点放入 `hidden_features` 或解析备注，等待用户确认。
