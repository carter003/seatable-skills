# SeaTable Python 脚本官方事实清单

本文件只整理官方资料里与 Python 脚本编写直接相关、最容易被 AI 写错的事实。

## 官方来源

- SeaTable 编程手册（中文）：https://seatable.github.io/seatable-scripts-cn/
- SeaTable Python 文档入口：https://seatable.github.io/seatable-scripts-cn/python/
- SeaTable Web API（中文）：https://docs.seatable.cn/published/seatable-api/home.md
- SeaTable Admin Docs 仓库：https://github.com/seatable/seatable-admin-docs

## 运行环境与初始化

- 官方 Python 脚本文档说明，脚本可在你自己的服务器运行，也可以上传到 SeaTable 后运行。
- 云端运行时可从 `seatable_api` 导入 `context`，其中常用属性包括：
  - `context.server_url`
  - `context.api_token`
  - `context.current_table`
  - `context.current_row`
  - `context.current_username`
  - `context.current_id_in_org`
- 云端脚本常见初始化方式：

```python
from seatable_api import Base, context

base = Base(context.api_token, context.server_url)
base.auth()
```

- 官方入门页也给出了外部脚本的初始化示例：从环境变量读取 `dtable_web_url` 与 `api_token` 后调用 `Base(...).auth()`。

## Token 与 API 基础

- SeaTable 文档区分 Account token、API Token 和 access-token。
- 如果只操作单个 base，官方建议优先使用“读写单个 base”的 API，不必使用用户名密码。
- 通过 API Token 或 Account token 都可以换取 base 的 access-token。
- 官方 Web API 文档说明：一个 base 的 access-token 有效期是 3 天。
- 国内云和私有部署时，`dtable_server` 与 `dtable_db` 的实际域名要以 access-token 返回结果为准，不应写死。

## 读行与分页

- `base.list_rows(table_name, view_name=None, start=None, limit=None)` 用于获取行。
- `list_rows` 的 `limit` 最大返回 1000 行，即使传入更大的值也不会超过 1000。
- 读取全表时，应该用 `start + limit` 循环分页。
- 如果要读取特定视图下的行，可以指定 `view_name`。

## SQL 查询

- `base.query(sql)` 用于执行 SQL 查询。
- 官方说明：默认最多返回 100 条结果。
- 如果需要更多结果，必须在 SQL 语句中显式加入 `limit`。
- 官方列出的典型异常包括：
  - `ValueError: sql can not be empty`
  - `ConnectionError: network error`
  - `Exception: no such table`
  - `Exception: no such column`
  - `Exception: columns in group by should match columns in select`
- `query` 适合筛选、排序、分组、去重，不适合替代全表分页读取。

## 行写入与批量操作

- 单行新增：`base.append_row(table_name, row_data, apply_default=False)`
- 批量新增：`base.batch_append_rows(table_name, rows_data, apply_default=False)`
- 单行更新：`base.update_row(table_name, row_id, row_data)`
- 批量更新：`base.batch_update_rows(table_name, rows_data=updates_data)`
- 单行删除：`base.delete_row(table_name, row_id)`
- 批量删除：`base.batch_delete_rows(table_name, row_ids)`
- `append_row` 支持 `apply_default=True`，在未显式提供某列值时采用列默认值。

## 链接操作

- 新增链接：`base.add_link(link_id, table_name, other_table_name, row_id, other_row_id)`
- 更新一行的链接集合：`base.update_link(link_id, table_name, other_table_name, row_id, other_rows_ids)`
- 批量更新链接：`base.batch_update_links(link_id, table_name, other_table_name, row_id_list, other_rows_ids_map)`
- 移除链接：`base.remove_link(link_id, table_name, other_table_name, row_id, other_row_id)`
- 通过列名获取 `link_id`：`base.get_column_link_id(table_name, column_name)`
- 查询被链接记录时使用的是 `link_column_key`，不是 `link_id`：

```python
base.get_linked_records(table_id, link_column_key, rows)
```

这两个标识不要混淆。

## 云端环境支持库

- 官方“云端环境下支持的库”页面写明：云端 Python 脚本运行在 Docker 容器中。
- 官方页面当前写的是 Python 3.7 环境。
- 官方列出的可直接使用库包括：
  - `seatable-api`
  - `dateutils`
  - `requests`
  - `pyOpenSSL`
  - `Pillow`
  - `python-barcode`
  - `pandas`
  - `numpy`
- 如果需要其他库，官方说明要联系 SeaTable，或改在本地运行脚本。

## 对 AI 最重要的落地结论

- 整表处理默认用 `list_rows` 分页，不要直接用 `query` 扫全表。
- 当前行脚本才依赖 `context.current_row`；批处理脚本应独立设计读取入口。
- 云端脚本优先用 `context.api_token + context.server_url` 初始化。
- 外部脚本要显式说明 token 来源与过期策略。
- 链接场景要先确认 `link_id`，查询链接记录时再确认 `link_column_key`。