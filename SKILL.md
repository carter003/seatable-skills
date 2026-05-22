---
name: seatable-python-scripting
description: '编写 SeaTable Python 脚本与自动化程序。用于 SeaTable 云端脚本、外部 Python Runner、Base/Context 初始化、分页读取、SQL 查询、行增删改、链接操作、空值清洗、批量处理、采购单/业务单据自动生成等场景。'
argument-hint: '描述要生成的 SeaTable Python 脚本目标、涉及的表、关键字段和触发方式'
user-invocable: true
---

# SeaTable Python Scripting

这个技能用于让 AI 按 SeaTable 官方资料编写 Python 脚本，优先生成可直接运行、适合批量数据处理的实现。

优先参考：

- [官方事实清单](./references/official-facts.md)
- [可复制模板](./assets/python-script-templates.md)

## 何时使用

- 需要为 SeaTable 编写 Python 脚本
- 需要在 SeaTable 云端脚本环境中操作 Base 数据
- 需要在本地或服务器上通过 Python 调用 SeaTable API
- 需要分页读取整表、分组处理、批量写回、建立链接关系
- 需要把业务规则转成稳定的自动化脚本

## 默认工作方式

1. 先判断脚本运行环境。
2. 如果是 SeaTable 云端脚本，优先使用 `context.server_url` 和 `context.api_token` 初始化 `Base`。
3. 如果是外部脚本，使用 API Token 换取 access-token，或使用官方示例中的环境变量初始化。
4. 先确认读取路径。
5. 整表扫描优先用 `base.list_rows()` 分页读取，不要假设 `base.query()` 能返回全部数据。
6. 条件筛选、排序、聚合优先用 `base.query(sql)`，但必须显式写 `limit`。
7. 写入前先统一做空值、文本、数字转换，避免字段类型不稳定导致脏数据。
8. 涉及链接列时，区分 `link_id` 和 `link_column_key`，不要混用。
9. 面向批处理任务时，默认不要依赖 `context.current_row`，除非需求明确是“用户在当前行手工运行”。
10. 输出代码时，同时给出关键假设、字段映射和错误处理点。

## 编写规则

### 1. 初始化规则

- 云端脚本优先使用以下导入模式：

```python
from seatable_api import Base, context

base = Base(context.api_token, context.server_url)
base.auth()
```

- 如果脚本需要日期处理，可直接导入 `dateutils`。
- 如果脚本在外部运行，优先给出可配置的初始化方式，不要把真实 token 写死进代码。

### 2. 读取规则

- 全表读取时使用分页助手，单次 `limit` 不超过 1000。
- `base.query(sql)` 默认最多返回 100 条，只有在 SQL 中写 `limit` 才能取更多。
- 如果是“按状态/条件筛选少量待处理记录”的批处理，优先使用 `base.query(sql)` 配合 `limit + offset` 分页，让服务端先过滤，再逐页处理。
- 不要为了避免 `base.query()` 截断，就把条件批处理改成 `base.list_rows()` 整表扫描后在本地过滤；这种改法在云端脚本里会明显拖慢首轮执行时间。
- 需要视图过滤时可在 `list_rows(table_name, view_name=...)` 中指定视图。
- 需要精确读取单行时使用 `base.get_row(table_name, row_id)`。

### 3. 写入规则

- 单行新增用 `base.append_row()`。
- 大批量新增优先考虑 `base.batch_append_rows()`。
- 单行更新用 `base.update_row()`。
- 多行更新优先考虑 `base.batch_update_rows()`。
- 删除操作默认保守，除非用户明确要求，否则不要自动生成删除逻辑。

### 4. 链接规则

- 新增链接使用 `base.add_link(link_id, table_name, other_table_name, row_id, other_row_id)`。
- 覆盖某一行的链接集合使用 `base.update_link(...)`。
- 查询一批行的被链接记录使用 `base.get_linked_records(table_id, link_column_key, rows)`。
- 需要通过列名查链接 id 时，使用 `base.get_column_link_id(table_name, column_name)`。

### 5. 稳定性规则

- 对文本、数字、列表、字典字段都做容错转换。
- 为批处理增加日志累积和最终摘要。
- 对“找不到字段值”“供应商不存在”“数量为 0”“重复生成”等业务条件写清楚处理分支。
- 对外部依赖或官方限制写成注释或说明，例如查询上限、运行环境、支持库限制。

## 面向 AI 的输出要求

- 默认输出完整可运行脚本，不只给片段。
- 如果需求里没有明确字段名，先用占位字段并把待确认项列清楚。
- 如果需求是整表处理，自动补上分页读取函数。
- 如果需求包含状态流转，自动补上“执行结果”或“日志字段”的更新方案。
- 如果需求涉及主表和明细表，自动考虑先生成主记录，再生成明细，再建立链接。
- 如果需求可能重复执行，自动设计幂等检查，例如按业务主键查重。

## 建议流程

1. 明确表名、字段名、链接关系、触发方式。
2. 选择运行环境：云端脚本或外部脚本。
3. 设计读取方式：分页读取或 SQL 查询。
4. 设计数据清洗函数。
5. 设计主业务流程：校验、分组、写入、回写状态。
6. 设计异常处理和执行日志。
7. 对批量任务补充幂等与重复生成保护。

## 不要这样写

- 不要默认依赖 `context.current_row` 来处理整表任务。
- 不要用 `base.query()` 代替整表分页读取。
- 不要把“条件过滤批处理”和“整表扫描批处理”混为一谈；前者优先 SQL 分页，后者才用 `list_rows()` 分页。
- 不要把 `link_id` 当作 `link_column_key` 传给链接查询接口。
- 不要把真实 API Token、Account Token、access-token 硬编码到脚本里。
- 不要假设所有单元格都是纯字符串；链接列、协作者列、文件列常常返回复杂结构。

## 生成代码前的最小检查清单

- 这是云端脚本还是外部脚本？
- 是否需要处理整表？如果需要，是否已经分页？
- 是否会重复运行？如果会，是否已做幂等检查？
- 是否要写回状态字段或执行日志？
- 是否涉及链接列？如果涉及，是否已经确认 `link_id`？
- 是否使用了官方文档支持的方法和参数？

## 资源

- 读取官方约束与注意事项：[官方事实清单](./references/official-facts.md)
- 复制基础骨架和常用模式：[可复制模板](./assets/python-script-templates.md)