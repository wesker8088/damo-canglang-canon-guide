# 大漠苍狼 Canon Guide

给远端 agent 的接入说明：如何在**不读整本原文**的情况下，快速了解南派三叔《大漠苍狼》上下部，并为拆解、解说、人物分析、场景选取等任务取证。

本仓库只放**操作说明**，不放小说原文、不放完整剧情包、不放访问 token。

## 书目

| 项 | 值 |
|---|---|
| book | `dmcl` |
| edition | `upload-20261101` |
| 内容 | 上部《绝地勘探》+ 下部《绝密飞行》 |
| 章节段 | 120（上部 52、下部 68） |
| 检索块 | 1,440 |

## 服务

| 项 | 值 |
|---|---|
| HTTP base | `http://100.64.65.68:8770` |
| MCP | `http://100.64.65.68:8770/mcp` |
| 鉴权 | `Authorization: Bearer <token>` |
| 免鉴权 | `GET /health` |
| 网络 | 仅 Tailscale 内网，不对公网开放 |
| 限流 | 所有入口合计 30 次/分钟 |

token 由管理员私下配置到运行环境的 `CANON_API_TOKEN`，**不要写进本仓库、对话、日志或代码**。

## 最短流程

```bash
# 0. 在线检查
curl -sS --noproxy '*' http://100.64.65.68:8770/health

# 1. 先拿剧情框架（人工整理稿）
curl -sS --noproxy '*' -X POST http://100.64.65.68:8770/brief \
  -H "Authorization: Bearer $CANON_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"book":"dmcl","section":"synopsis"}'

# 2. 再问具体问题（返回候选证据，不自动作答）
curl -sS --noproxy '*' -X POST http://100.64.65.68:8770/ask \
  -H "Authorization: Bearer $CANON_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"query":"老猫为什么知道洞里的事","book":"dmcl","edition":"upload-20261101","volume":"上部 绝地勘探"}'

# 3. 引用前逐字核对
curl -sS --noproxy '*' -X POST http://100.64.65.68:8770/cite \
  -H "Authorization: Bearer $CANON_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"quote":"必然导致必然","book":"dmcl","edition":"upload-20261101"}'
```

## 文档

- [AGENT_GUIDE.md](AGENT_GUIDE.md)：连接、工具、返回结构、推荐流程、常见误判
- [REMOTE_TASK_PROMPT.md](REMOTE_TASK_PROMPT.md)：可直接粘贴给远端 agent 的任务提示模板
- [SECURITY.md](SECURITY.md)：token、网络、引用和全文提取边界

## 三条底线

1. `brief` 是人工整理稿，不是原文；需要逐字引用时用 `cite`。
2. `ask/search` 返回候选证据，`answer` 永远为 `null`；排序靠前不等于语义支持。
3. 每条短引 ≤30 字，注明《大漠苍狼·上部/下部》与章名；不要复制整段原文。

## 常见误判

- 下部《绝密飞行》不是上部的重复版本，而是承接上部结尾的下部正文。
- 老猫是“先遣队唯一活着出来的人”，这是老猫自述；他仍有隐瞒。
- 袁喜乐是敌特（作者后记明确），但她的感情是否全为伪装，正文保留模糊。
- “深渊灯光”“循环次数”“墙上被刮掉的字”“年轻毛五月是谁”是作者故意留白，不要写成确定设定。
- 不要把“深渊里的灯光”直接写成“日军基地仍在运转”。

---

本仓库只用于制作流程和检索说明；小说人物、设定与引文版权归原作者及权利人所有。
