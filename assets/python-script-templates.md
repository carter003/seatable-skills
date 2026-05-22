# SeaTable Python 可复制模板

这些模板基于官方文档约束整理，目标是让 AI 在生成脚本时直接套骨架，而不是从零拼装。

## 模板 1：SeaTable 云端脚本基础骨架

```python
from seatable_api import Base, context, dateutils
import traceback

base = Base(context.api_token, context.server_url)
base.auth()

start_time = dateutils.now()
logs = []


def log(message):
    text = str(message)
    print(text)
    logs.append(text)


def main():
    table_name = '表1'
    rows = base.list_rows(table_name, start=0, limit=100)
    log('读取到 %s 行' % len(rows))


try:
    main()
except Exception as error:
    log('执行异常: %s' % error)
    log(traceback.format_exc())
```

适用场景：手工运行、定时运行、流程触发的云端脚本。

## 模板 2：整表分页读取

```python
def list_all_rows(table_name, view_name=None, page_size=1000):
    all_rows = []
    start = 0
    while True:
        rows = base.list_rows(table_name, view_name=view_name, start=start, limit=page_size)
        if not rows:
            break
        all_rows.extend(rows)
        if len(rows) < page_size:
            break
        start += len(rows)
    return all_rows
```

适用场景：采购计划、订单明细、主数据字典、库存快照等整表扫描任务。

## 模板 3：空值与类型清洗

```python
def has_value(value):
    if value is None:
        return False
    if isinstance(value, str):
        return value.strip() != ''
    if isinstance(value, (list, tuple, set)):
        return any(has_value(item) for item in value)
    if isinstance(value, dict):
        return any(has_value(item) for item in value.values())
    return True


def as_text(value):
    if value is None:
        return ''
    if isinstance(value, str):
        return value.strip()
    if isinstance(value, dict):
        for key in ('display_value', 'name', 'text'):
            if has_value(value.get(key)):
                return str(value.get(key)).strip()
        return ''
    if isinstance(value, (list, tuple)):
        for item in value:
            text = as_text(item)
            if text:
                return text
        return ''
    return str(value).strip()


def as_number(value):
    if value in (None, ''):
        return 0
    try:
        return float(value)
    except (TypeError, ValueError):
        return 0
```

适用场景：链接列、协作者列、选择列、附件列等返回结构不固定时。

## 模板 4：分组处理并回写状态

```python
def update_status(row_id, status, result):
    base.update_row('采购计划', row_id, {
        '计划状态': status,
        '执行结果': result,
    })


def group_rows(rows):
    grouped = {}
    for row in rows:
        sale_no = as_text(row.get('销售订单号'))
        factory = as_text(row.get('加工厂'))
        if not sale_no or not factory:
            update_status(row['_id'], '异常', '缺少销售订单号或加工厂')
            continue
        grouped.setdefault((sale_no, factory), []).append(row)
    return grouped
```

适用场景：按销售订单号、供应商、工厂、月份等业务键做批处理。

## 模板 5：主表 + 明细表 + 链接

```python
def create_order(header_data, detail_rows, detail_link_id, source_link_id):
    header = base.append_row('采购订单总表', header_data)
    header_id = header['_id']

    for detail in detail_rows:
        source_row_id = detail.pop('source_row_id')
        detail_row = base.append_row('采购订单明细', detail)
        detail_row_id = detail_row['_id']

        base.add_link(detail_link_id, '采购订单明细', '采购订单总表', detail_row_id, header_id)
        base.add_link(source_link_id, '采购计划', '采购订单总表', source_row_id, header_id)

    return header_id
```

适用场景：采购单、发货单、对账单、报工单等主从结构数据生成。

## 模板 6：外部 Python 脚本初始化

```python
import os
from seatable_api import Base

server_url = os.environ.get('dtable_web_url')
api_token = os.environ.get('api_token')

base = Base(api_token, server_url)
base.auth()
```

适用场景：脚本在你自己的服务器、CI、定时任务平台或本地环境运行。

## 模板 7：使用 SQL 做筛选与聚合

```python
sql = '''
select 供应商, sum(下单数量) as total_qty
from 采购订单明细
where 生成时间 >= '2026-01-01'
group by 供应商
limit 500
'''

rows = base.query(sql)
```

适用场景：聚合报表、去重、排序、条件筛选。

注意：官方文档说明 `base.query()` 默认最多返回 100 条，不写 `limit` 容易漏数据。