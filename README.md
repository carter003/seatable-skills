# SeaTable Python Scripting Skill

一个面向 GitHub Copilot / VS Code Chat 的 SeaTable Python 脚本技能包。

这个 skill 的目标很简单：让 AI 在生成 SeaTable Python 脚本时，优先遵循官方约束，输出更适合真实业务批处理的实现，而不是只给零散片段。

## 适用场景

- SeaTable 云端 Python 脚本
- 外部 Python Runner / 本地脚本
- 批量读取与分页处理
- SQL 条件筛选与聚合
- 主表、明细表、链接列自动化
- 采购单、加工单、业务单据生成脚本

## 包含内容

- `SKILL.md`：skill 定义、使用规则、输出要求
- `assets/python-script-templates.md`：可直接复用的 Python 模板
- `references/official-facts.md`：从官方资料提炼的关键事实清单

## 核心设计原则

- 云端脚本优先使用 `Base(context.api_token, context.server_url)` 初始化
- 整表处理优先使用 `list_rows()` 分页
- 条件批处理优先使用 `query(sql)` 并显式写 `limit`
- 链接场景明确区分 `link_id` 与 `link_column_key`
- 默认考虑空值清洗、日志、幂等、防重复生成

## 使用方式

把这个目录放到你的聊天技能目录中：

```text
.github/skills/seatable-python-scripting/
```

然后在提问时明确这些信息：

- 脚本运行环境
- 表名与字段名
- 关键业务规则
- 是否需要分页、状态回写、幂等检查

## 示例提问

```text
帮我写一个 SeaTable 云端 Python 脚本：
从“采购计划”表读取“计划状态=待生成”的记录，
按“销售订单号 + 加工厂”分组生成采购订单总表和明细表，
生成后回写状态和执行结果，要求支持重复执行保护。
```

## 说明

这个仓库只提供 skill 说明、模板和参考规则，不包含业务私有 token，也不绑定某个固定 Base 结构。