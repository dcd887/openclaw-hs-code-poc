# OpenClaw × 海关归类 PoC

一个双代理（planner → checker）的海关商品归类概念验证，跑在 OpenClaw 2026.10.7 上，
后端接的是自研 MCP server（8972 条 2026 版税则）。

## 它解决什么

报关场景里，客户给的是一句口语：「进口一批聚乙烯颗粒，比重 0.92」。
要变成可申报的编码，要经过：拆特征 → 查税则 → 排干扰项 → 标出待确认信息。
这个过程现在靠人，慢且不可复现。

这个 PoC 把它拆成两道工序，每道工序有独立职责和固定输出格式。

## 架构

```
用户一句话
   ↓
planner（coordinator 角色）
   拆成结构化任务单：商品 / 关键特征 / 可能章 / 需查证 / 风险点
   硬规矩：不许自己编编码
   ↓
checker（researcher 角色）
   调 MCP 工具查证 → 给出编码 + 引用原文 + 税费 + 候选对比 + 置信度
   硬规矩：每个编码必须来自工具返回
   ↓
结构化结论（含待确认清单）
```

## 技术栈

| 层 | 用了什么 |
|---|---|
| 代理框架 | OpenClaw 2026.10.7（多代理、隔离 workspace、AGENTS.md 约定） |
| 模型 | DeepSeek `deepseek-flash` |
| 工具层 | MCP（Model Context Protocol）自研 server |
| 工具数 | 3 个：`search_hs_code` / `get_hs_detail` / `calculate_tax` |
| 数据 | 8972 条 2026 版进出口税则 + 1228 条章注 |
| 运行时 | 便携版 Node 24.22.1（隔离安装，不动系统 Node） |

## 实测结果（全部来自真实工具调用）

| 商品 | 结论编码 | 置信度 | 亮点 |
|---|---|---|---|
| 聚乙烯颗粒（比重 0.92） | 3901.1000 | 中 | 指出 LLDPE（3901.4020）无法仅凭密度排除；发现名称含「6.5」与工具 `import_tariff=0.0%` 矛盾并拒绝拍板 |
| 烘焙面包 | 1905.9000 | 中 | 指出缺形态/配方，无法在 1905.90 与 1905.40 之间自定 |
| 香水 | 3303.0000 | 编码高 / 税费低 | 主动指出消费税 0% 与「高档化妆品」规则冲突 |
| 去壳腰果 | 0801.3200 | 高 | 限定前提（生/干、仅机械去壳），熟制调味则不成立 |
| 葡萄糖粉 | 1702.3000 | 中 | 指南缺果糖含量，无法确认 1702.30 / 1702.40 分界 |
| 聚乙烯（早前单代理） | 3901.1000 / .2000 / .4020 | — | 正确剔除 3905.3000 聚乙烯醇、3920.9100 PVB —— 同名词不同物 |

## 这个 PoC 证明了什么

1. 代理能不能守住边界：planner 被明确禁止编编码，它确实只做拆解；checker 的每个编码都能回溯到工具返回原文。
2. 会不会主动暴露不确定性：5 个商品里只有 1 个给了「高」置信度，其余都列了具体缺什么。工具返回值有矛盾时它会停下来报警（税率 6.5 vs 0.0%）。
3. 排干扰项的能力：聚乙烯那条里，它把「聚乙烯醇」（PVA）和「PVB 薄膜」正确识别为同名词不同物，而不是照着关键词照抄。

## 还没做的（诚实清单）

- 多代理协同目前是「人工串联」（我手动把 planner 的输出交给 checker），
  没有配 `sessions_spawn` 自动委派——OpenClaw 支持，但这一版没接
- 只有 3 个 MCP 工具；没有做批量并行（10 条一起跑）
- 没有接监管条件/原产地数据，税费只算到关税+消费税+增值税
- 没有前端界面，交互靠 CLI

## 怎么跑

```bash
# 环境
<Node 目录>\node.exe  # 便携 Node
<OpenClaw 目录>\node_modules\openclaw\dist\index.js  # OpenClaw CLI

# MCP server 已注册为 hs-code-poc
openclaw mcp probe hs-code-poc

# 单轮
openclaw agent --local --agent planner --message "客户进口聚乙烯颗粒，比重 0.92"
openclaw agent --local --agent checker --message "<把上面输出粘进来>"
```

## 文件

| 文件 | 内容 |
|---|---|
| `mcp/hs_code_poc_server.py` | MCP server（3 工具，8972 条全量） |
| `agents/planner/AGENTS.md` | planner 的角色定义与输出格式 |
| `agents/checker/AGENTS.md` | checker 的角色定义与硬规矩 |
| `evidence/live/*.txt` | 4 次真实运行的原始输出（planner / checker / 腰果 / 香水） |
| `evidence/shots/demo_full.png` | 演示全图（1480×1300，一次看完全流程） |
| `evidence/PoC演示.mp4` | 演示视频（19 秒，8 倍速节奏回放真实输出） |
| `evidence/demo.html` | 演示页面源码（可重跑 `python render_demo.py` 重建） |

## 演示怎么来的

agent 单次调用 30–60 秒，且多个进程会争抢 `~/.openclaw/state/openclaw.sqlite` 所有权
（实测报 `state database is busy`），桌面录屏会把"卡住不动"录进去。

所以：先真实跑完把输出存盘（`evidence/live/`），再用无头浏览器渲染成终端风格页面，
逐帧截图合成视频。视频与截图里每一行都是真实运行结果，未改写；页面上也明确标注了
「本段为真实运行的实录回放」。
