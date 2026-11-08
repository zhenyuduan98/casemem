# CaseMem

跨场景 Agent 的可审计长期记忆。客服、行程、报销共用同一套管线：按业务隔离，只把有证据的要求升格成规则，下次处理前先召回。

**项目说明：** 实现了一个可跨业务复用的 Agent 长期记忆层。对话和工具结果先按会话压缩，再只把有证据的要求升格成规则：用户明确的长期交代、跨会话重复出现、工具失败后的修正。记忆按 `project_id` 隔离，客服偏好、行程约束和报销预审走同一套整理与召回流程，不会串到别的业务。新任务开始前召回相关规则，并保留无记忆对照和逐条遵守检查。技术栈为 Python、Agently、通义千问（qwen-plus）。

## 它解决什么问题

把整段历史对话塞进下一次请求，模型拿到的是流水，不是可以遵守的规则。客服会重复问已经交代过的处理方式，行程会重犯上一次被工具否掉的排法，报销会每次都追问同一条制度。

CaseMem 在两次任务之间做整理。模型负责摘要、合并和生成检索词；代码负责落盘、按业务过滤、按证据决定能否升格，以及把选中的规则拼进下一次请求。

## 同一套管线，多个业务

仓库里的仿真数据已经是两个业务，而不是单一行程场景。

| 业务 | project_id | 会留下的规则 |
|---|---|---|
| 客服 / 工单 | 接入时新建，例如 `support-agent` | 客户明确的联系偏好、重复出现的处理要求、工具失败后的修正 |
| 行程规划 | `travel-agent` | 午睡窗口、远距离景点独占一整天、输出必须附路线检查 |
| 报销预审 | `expense-agent` | 小额交通票以后按新制度直接预审，不必每次追问 |

客服事件和现有材料使用同一结构：`project_id`、`session_id`、`event_id`、角色、正文，工具失败时带上 `tool` 和 `status`。换业务时改的是事件内容，整理、升格和召回代码不变。

当前材料是 `materials/simulated_long_conversation.jsonl`：36 条事件，3 个会话。北京亲子游和西安亲子游属于 `travel-agent`，出租车发票预审属于 `expense-agent`。召回旅行任务时，报销记忆会被项目过滤掉。

## 一次新任务里发生什么

```text
原始事件
  -> 按会话压缩成摘要和候选
  -> 只升格有证据的规则
  -> 按当前业务和当前任务召回
  -> 生成回复，并逐条检查是否遵守
```

升格只走三条通道：

- 用户明确说了以后一直有效
- 同一要求在多个会话里重复出现
- 工具失败之后形成了修正做法

对不上的候选留在待观察列表，不进入长期规则。

## 记忆分层

| 层 | 存什么 | 跨场景里对应什么 |
|---|---|---|
| Raw | 原始对话和工具结果 | 客服原话、路线检查原文、发票预审对话 |
| Consolidated | 单次会话摘要和候选 | 这一单怎么失败、怎么改 |
| Semantic | 跨会话仍然有效的规则 | 处理偏好、项目约束、预审制度 |
| Working | 当前任务带进上下文的记忆包 | 这一次回复实际要用的规则和摘要 |

后台心跳比较已处理的事件号。有新事件才重新整理；没有新事件就跳过模型调用。

## 演示对照

旅行业务上的一句新任务：「带 5 岁的女儿去上海玩两天，孩子一直想去迪士尼乐园。」

无记忆起草时，两天都排进迪士尼，没有午睡窗口。带上本业务的记忆后，系统召回午睡、上午安排重点景点、远距离景点独占一天、必须附路线检查等规则，并逐条给出已满足、部分满足或未满足。检查会指出草案仍然没把午睡写进安排，也会指出「住在王府井附近」这条北京规则不能直接用到上海。规则留着证据，遵守情况可以复核。报销业务的记忆不参与这次召回。

## 运行

需要 Python 3.12 和阿里云百炼的 `DASHSCOPE_API_KEY`。密钥放在系统环境变量或项目根目录 `.env`，不要提交到仓库。

```bash
pip install -i https://mirrors.aliyun.com/pypi/simple/ agently python-dotenv
```

`.env` 可参考 `.env.example`：

```text
DASHSCOPE_BASE_URL=https://dashscope.aliyuncs.com/compatible-mode/v1
DASHSCOPE_MODEL=qwen-plus
DASHSCOPE_API_KEY=你的百炼 API Key
```

最小请求：

```bash
python demo.py
```

记忆整理与召回：

```bash
python scripts/01_compress_session_memory.py
python scripts/02_promote_long_term_memory.py
python scripts/03_recall_new_session.py
python scripts/04_recall_process_memory.py
python scripts/05_extract_capability_memory.py
```

文件记忆、心跳和综合对照：

```bash
python scripts/01_raw_only_context.py
python scripts/06_heartbeat_trigger.py
python scripts/07_file_recall.py
python scripts/08_integrated_workspace_case/run.py
```

`06` 会在有新原始事件时依次调用收集、抽取、晋升和工作记忆脚本。`08` 清空本次演示目录，重建四层记忆，导入 RecordStore，再输出有记忆和无记忆两份结果。结果写在 `.demo_runs/`。

## 目录

```text
demo.py                     一次请求里的记忆槽位
materials/                  仿真对话和当前过程事件
scripts/01–05               压缩、升格、召回、方法卡
scripts/02_collect_raw.py   起，文件版四层记忆
scripts/06_heartbeat_trigger.py
scripts/08_integrated_workspace_case/
```

## 技术栈

Python 3.12、Agently 4.1、通义千问 qwen-plus（DashScope OpenAI 兼容接口）、本地 JSON/JSONL 与 RecordStore。
