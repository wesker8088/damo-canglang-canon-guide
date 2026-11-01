# 远端 Agent 接入说明

这本书的检索服务只做一件事：**返回带卷章、行号、严格坐标的原文证据**，不替你写答案。  
`brief` 用于快速建立剧情框架；`ask/search` 用于找候选；`cite` 用于逐字核对。

---

## 1. 连接

| 项 | 值 |
|---|---|
| HTTP base | `http://100.64.65.68:8770` |
| MCP（Streamable HTTP） | `http://100.64.65.68:8770/mcp` |
| OpenAPI | `http://100.64.65.68:8770/openapi.json`（需 token） |
| 鉴权 | 请求头 `Authorization: Bearer <token>` |
| 免鉴权 | `GET /health` |
| 网络 | 仅 Tailscale 内网，不对公网开放 |
| 限流 | 所有入口合计 30 次/分钟；超限 429 + `Retry-After: 60` |
| 请求体上限 | 16 KiB；`query` ≤ 500 字 |

token 由管理员配置到运行环境的 `CANON_API_TOKEN`，不要写进对话、日志或代码仓库。

MCP 配置示例：

```json
{
  "mcpServers": {
    "canon": {
      "type": "http",
      "url": "http://100.64.65.68:8770/mcp",
      "headers": { "Authorization": "Bearer ${CANON_API_TOKEN}" }
    }
  }
}
```

> 如果运行环境有 HTTP 代理，访问 `100.64.65.68` 时要绕过代理（curl 加 `--noproxy '*'`）。

调用前先 `GET /health`：返回 `{"status":"ok"}` 才说明在线。服务随家里 PC 开机自启，但 PC 关机或 Tailscale 掉线时会不可用。**不在线时报告“检索服务未启动”，不要凭记忆作答。**

---

## 2. 工具

| 工具 | 用途 | 必填 | 常用可选 |
|---|---|---|---|
| `brief` | **首选**。读人工整理稿（梗概/人物/设定/时间线） | `book` | `section` |
| `ask` | 问具体问题，返回候选证据 + 上下文 | `query` | `book`、`edition`、`volume`、`offset` |
| `search` | 只要检索候选（不做事件/前提核对） | `query` | `mode`、`volume`、`edition` |
| `cite` | 逐字核对一句原文是否存在于本书 | `quote` | `book`、`edition`、`volume` |
| `entity` | 按人物/概念名查出现位置 | `name` | `book`、`edition`、`offset` |
| `health` | 服务状态 | — | — |

**导入书硬规则**：`book=dmcl` 时，`ask/search/cite/entity` 必须同时给 `edition=upload-20261101`；`brief` 除外。

### 2.1 brief

```bash
curl -sS --noproxy '*' -X POST http://100.64.65.68:8770/brief \
  -H "Authorization: Bearer $CANON_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"book":"dmcl"}'
```

可只取一节：

```json
{"book":"dmcl","section":"characters"}
```

可选 section：`logline`、`one_line`、`synopsis`、`acts`、`characters`、`concepts`、`timeline`、`structure`、`open_questions`、`citation_rules`、`query_examples`。

### 2.2 ask

```bash
curl -sS --noproxy '*' -X POST http://100.64.65.68:8770/ask \
  -H "Authorization: Bearer $CANON_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"query":"老猫为什么知道洞里的事","book":"dmcl","edition":"upload-20261101","volume":"上部 绝地勘探"}'
```

`volume` 可写 `上部 绝地勘探` 或 `下部 绝密飞行`。问句尽量包含“人物 + 动作/身份 + 想知道的关系”，不要只问“袁喜乐是谁”。

### 2.3 cite

```bash
curl -sS --noproxy '*' -X POST http://100.64.65.68:8770/cite \
  -H "Authorization: Bearer $CANON_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"quote":"必然导致必然","book":"dmcl","edition":"upload-20261101"}'
```

返回 `verbatim_exists`、`match_count` 和全部命中坐标。一页 200 条不够时按 `continuation.offset` 续页。

### 2.4 entity

```bash
curl -sS --noproxy '*' -X POST http://100.64.65.68:8770/entity \
  -H "Authorization: Bearer $CANON_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"name":"深山","book":"dmcl","edition":"upload-20261101"}'
```

别名命中只用于定位，不代表共指已确认。

---

## 3. 怎么读 ask 的返回

```jsonc
{
  "status": "有字面候选；语义待核",
  "answer": null,
  "book": "dmcl",
  "edition": "upload-20261101",
  "evidence": [
    {
      "chapter_id": "dmcl@upload-20261101:0045",
      "volume": "上部 绝地勘探",
      "title": "第四十四章 老猫",
      "line_start": 1991,
      "quote": "……",
      "kind": "检索候选；未核语义支持",
      "certificate": { "...": "严格坐标" }
    }
  ],
  "coverage": { "candidate_count": 37, "semantic": "新书未生成人工语义断言" },
  "continuation": { "offset": 8, "...": "..." }
}
```

| status | 意思 | 你该怎么做 |
|---|---|---|
| `有字面候选；语义待核` | 关键词在原文中出现了，但没人核实它是否回答你的问题 | 读 `quote` 和上下文，自己判断 |
| `仅有语义候选；未核语义支持` | 向量邻居，可能只是气氛相近 | 不要当证据；换关键词或加 `volume` |
| `未找到证据` | 这份文本里没检索到 | 说“未找到”，不要编；换说法或拆小问题 |
| `前提未获证据支持` | 问题里的说法和原文冲突 | 看 `premise_check.checks` |

导入书的 `kind` 默认是“检索候选；未核语义支持”。**不要把它当“叙述说明”**。要区分叙述/角色说法，必须读 `quote` 和 `context_lines`。

---

## 4. 推荐工作流

### 4.1 做作品前：建立框架

1. `brief(book=dmcl)` 读 `logline`、`synopsis`、`acts`。
2. 读 `characters`，记下“谁是谁、谁做了什么、谁死了”。
3. 读 `concepts`，记下“七二三工程/深山/深渊/时空漩涡/敌特”五个核心。
4. 读 `open_questions`，避免把留白写成确定答案。
5. 需要具体场景时，再用 `ask` + `cite` 定位。

### 4.2 核对一个说法

1. `ask` 用“人物 + 动作”具体问。
2. 看 `status`；有 `evidence` 就读 `quote`。
3. 要引用原文：把 `quote` 拿去 `cite` 逐字核对，并记下卷名、章名、行号。
4. 要引用整段：用 `cite` 返回的坐标回传严格校验，不要自己改字。

### 4.3 查找人物/概念

`entity` 适合“这个人/东西出现在哪些章”；不适合“这个人为什么这样做”。后者用 `ask`。

---

## 5. 本作高频误判

| 误判 | 纠正 |
|---|---|
| 把下部文件当上部重复本 | 下部是承接上部结尾的**下部正文**；上部结尾在“启动放映机”，下部从放映胶片开始。 |
| 把“老猫是先遣队唯一幸存者”当成全部真相 | 这是老猫自述；他仍有隐瞒，正文后段才逐步揭开。 |
| 把“袁喜乐是敌特”当成她全部动机 | 作者后记明确她是敌特；但她的感情是否全为伪装，正文保留模糊。 |
| 把“深渊灯光”写成“日军基地仍在运转” | 正文只写看到灯光；作者后记用“全是石头”打趣，未确认。 |
| 把“循环”说成无限次 | 作者后记明确说“不说那么明白”；正文只呈现至少一次闭合。 |
| 把“年轻毛五月”说成老猫本人 | 正文只写同名同姓、28 岁、不认识主角；身份未确认。 |
| 把“石刻巨人”说成外星人/机械 | 裴青推测是远古文明石雕；作者后记确认“远古文明”，未说外星/机械。 |

---

## 6. 引用格式

> “必然导致必然。”（《大漠苍狼·下部 绝密飞行》第五十八章）

- 每条短引 ≤30 字。
- 标卷名 + 章名；需要精确坐标时附 `chapter_id` / 行号。
- 角色说的话写“某某说”，不要直接写成叙述事实。
- 作者后记的解释写“作者在后记中说明”，不要写成正文情节。

---

## 7. 常见问题

| 现象 | 原因/处理 |
|---|---|
| 401 | token 缺失或错误 |
| 400 | 字段不符合 schema；`dmcl` 忘记 `edition` 是最常见原因 |
| 429 | 超过 30 次/分钟，等 60 秒 |
| 一直“仅有语义候选” | 问句太泛；加人物、动作、卷名，或改用 `literal` |
| `cite` 返回 `verbatim_exists: false` | 这句话不在这份文本中；换短引或确认不是影视/同人台词 |
| `brief` 返回“尚未登记” | 只支持已登记书目；本作是 `dmcl` |
