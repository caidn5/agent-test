# 适配器：Claude Code 宿主 + 飞书通道（lark-cli）

实现三个能力接口的 Claude Code 版本。**本文件是 skill 中唯一允许出现具体 CLI 命令的地方。**

> 命令以 lark-cli 实测为准：执行前先加载 lark-im / lark-base 技能获取其 references 中的准确参数；命令成功判据统一为输出信封 `ok == true`。

## 适配器规范（其他宿主照此实现）

任何新适配器只需实现以下函数语义，SKILL.md 主流程即可无改动运行：

1. `send(question) -> sent_at`：把一条文本消息发给被测对象；失败抛错
2. `await_reply(sent_at, timeout) -> reply_text`：等待并返回被测对象对本次提问的完整回复；超时抛 Timeout
3. `table_create(name, fields) -> {table_ref}`：创建用例表
4. `rows_append(table_ref, rows[]) -> record_ids[]`：批量写入用例
5. `row_update(table_ref, record_id, patch)`：单行回写
6. `rows_read(table_ref) -> rows[]`：读回用例行（断点续跑用）
7. `progress_save/load(payload)`：进度落盘/读盘（本适配体直接用工作区 JSON 文件）

## 提问通道（飞书 IM 单聊）

### 自检（Phase 0）
```
lark-cli auth status --json --verify     # 认证有效，具备 im 消息 scope
lark-cli im +chat-list --as user --types p2p   # 列单聊
```
从返回中按主题配置的被测对象名称/ID 匹配目标单聊 `oc_xxx`，**向用户确认后**采用。往返实测：发一条无害问候（如「你好，测试通道」），确认能读到被测对象回复。

### 发送
```
lark-cli im +messages-send --chat-id <oc_xxx> --as user --text "<问题>"
```

### 轮询与完成判定（await_reply）
发送前先记录当前最新一条消息的 `message_id` 作为水位线。然后循环（间隔 10-15s）：
```
lark-cli im +chat-messages-list --chat-id <oc_xxx> --as user --order asc --page-size 50
```
取水位线之后的消息，判定完成须同时满足：
1. 最后一条消息的发送者是**被测对象**（不是自己）
2. 该消息内容长度**连续 3 次轮询稳定**（流式/分段输出会持续变长）
3. 累计等待 ≥30s（防秒回的中间态）

被测对象可能一条回复拆成多条消息（文本+卡片+追问）：水位线后所有来自被测对象的连续消息按时间序拼接为完整回复。**追问/反问处理**：若回复是要求补充信息的确认类反问（如「请确认模型」），按主题配置的「会话预处理」答复后继续等待本轮真正的回答；无法自动处理则把反问记为该题回答，状态照常「已执行」，留给人工。

**出题委托的长回复（Phase 3）**：生成结果很长且分批输出（每批 ≤10 条）。完成判定照旧（水位线+3 次稳定）；批次收完且未到声明条数/无明确收尾（如「以上共 N 条」）时，发「继续」取下一批；生成阶段单轮等待上限放宽为 600s。

### 超时与失败
- 单题上限默认 300s（主题配置可覆盖）→ 状态「超时」，回答列记录最后已见内容
- 发送即报错 → 状态「发送失败」，继续下一条

### 限速
题间停 5-10s；连续 3 题超时则暂停整批，报告用户。

## 探查回复中的多维表格（Phase 1）

被测对象可能用一张多维表格（如指标/维度汇总表）回复元数据探查，消息里是分享链接或卡片。处理：

- 从回复文本/卡片提取表格标识：链接形如 `.../base/<app_token>?table=<table_id>&view=<view_id>`
- 先读字段列表再读记录，**分页读全**（默认每页 20 条，表大时翻页直至无 has_more）；命令以 lark-base 技能为 SSOT
- 提取的指标/维度/口径整理进探查结果文件，注明「来源：被测对象自报多维表格 <链接>」
- 无权限/链接失效：请用户开通访问，或回复被测对象「请直接以文本列出全部指标与维度」兜底

## 用例表（飞书多维表格）

以 lark-base 技能为唯一命令来源（先读其 SKILL.md 与 `references/lark-base-field-json.md`、`lark-base-record-batch-create.md`）。要点：
- 建表：`lark-cli base +base-create --name <名> --table-name <表名> --fields '<字段JSON>'`（默认 `--as user`）
- 字段 JSON 结构以 lark-base 的字段规范文件为 SSOT，禁用旧结构
- 写用例：`+record-batch-create`，≤200 条/批、串行
- 回写：`+record-batch-update`（单行也走它），实测配方（2026-10-22，WinPS 5.1）：
  - 拿 record_id / 回读验证：`+record-list --base-token <t> --table-id <tbl> --filter-json @tmp-filter.json --format json`，取 `data.record_id_list`；筛选体同走临时文件——`--filter-json` 内联在 PS 5.1 下**时灵时不灵**（2026-10-22 同一写法一次成功一次被拆碎），一律按下行 @file 规矩走；回读核对打印整行（筛选模式与全表读的列序不同，且列序会随表结构变动漂移——2026-10-23 实测同一表 [3] 从 DAX 漂成使用说明；按列号定位/扫描前先打印全行确认，跨列搜索时遍历所有列）
  - 更新体含中文/嵌套引号时**禁内联传参**（PS 会把带空格 JSON 拆碎、内嵌双引号传丢）：ConvertTo-Json 写入**当前目录**相对路径临时文件（UTF-8 无 BOM），`--json "@tmp-file.json"`，用完即删
  - 结构：`{"update_records":{"recX":{"字段名":"文本值"}}}`；响应 `ok:true` + `rev` 递增即成功，回读验证
- 读回：`+record-list`（带 record_id）或 `+record-search`（keyword 匹配，但不返回 record_id）

## 进度持久化

工作区 `progress/<主题>-progress.json`：
```json
{
  "table": {"base_token": "...", "table_id": "...", "url": "..."},
  "chat_id": "oc_xxx",
  "waterline_msg_id": "om_xxx",
  "cases": [{"record_id": "recX", "编号": "A-01", "状态": "已执行"}],
  "updated_at": "..."
}
```
每题回写后立即重写此文件（整文件覆盖，原子性靠先写临时文件再替换）；同时把进度摘要同步进工作区记忆文件（若有）。断点续跑 = 读此文件 + 表内状态列双重校验。
