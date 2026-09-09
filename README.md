# FinSage · 多智能体金融投研决策系统

> 个人作品集 / 学习演示项目，**非实盘交易系统**，不自动下单、不承诺收益。

FinSage 把「确定性量化因子计算 + LLM 情绪研判 + 规则化风控」用多智能体编排成一条**可解释的投研链路**。核心差异：不是让 LLM 凭空聊股票，而是让因子、情绪、风控各司其职、可独立验证。

方法论融合自：
- 多因子量化研究传统（Fama-French 因子体系、动态因子配置思想）——双通道（因子+情绪）LSTM 融合编码器、时序-特征双维注意力 → 动态因子权重。
- 多智能体 Agent 设计实践——多步推理 + 规则引擎 + 风险提示。

---

## 架构

```
行情/财务数据 ──► FactorAgent ──┐
                                ├─► ResearchAgent(LSTM-Attention 动态权重 + 推理链) ──► RiskAgent ──► ReporterAgent ──► 用户
新闻/舆情数据 ──► SentimentAgent┘
                                   ▲
                              共享状态(LangGraph State) + RAG 向量库(因子说明/新闻/历史报告)
```

| Agent | 职责 | 工具 |
|---|---|---|
| Orchestrator | 依赖编排：Factor∥Sentiment → Research → Risk → Reporter | LangGraph（含无依赖兜底编排器） |
| FactorAgent | 五类风格因子（规模/价值/动量/波动率/质量）标准化 | pandas/numpy + Akshare（合成兜底） |
| SentimentAgent | 新闻/公告情感打分 [-1,1] + 置信度 + 依据 | LLM（Key 可选）+ 词典兜底 |
| ResearchAgent | 动态因子权重 + 个股排序 + 可解释推理链 + 回测 | PyTorch LSTM-Attention（个股打分/情绪融合）；动态权重=regime 条件化因子 IC（与特征注意力同构、可解释） |
| RiskAgent | 规则化风险校验 + 降级建议 | 规则引擎 + LLM 归纳 |
| ReporterAgent | 四段式结构化报告 + 强制免责声明 | 模板 + RAG 检索增强 |

---

## 快速开始

```bash
# 1. 建隔离环境并装依赖
python -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt

# 2. 跑一个端到端演示（默认沪深300样例池，Akshare优先+合成兜底）
python -m finsage --pool HS300 --start 2025-01-01 --end 2025-12-31

# 3. 完全离线（纯合成数据，最快验证链路）
python -m finsage --pool 600519.SH,000858.SZ --source synthetic --top-n 3

# 4. 结构化 JSON 摘要
python -m finsage --pool HS300 --json
```

报告会保存到 `data/reports/finsage_report.md` 与 `.html`（浏览器打开即可看）。

---

## 接入真实 LLM（.env 一键配置，支持中转站）

系统用 OpenAI 兼容接口。推荐在项目根目录建 `.env` 文件（已 git 忽略，模块加载时自动读取），**无需改代码、无需 export**：

```ini
# finsage/.env
OPENAI_BASE_URL=https://你的中转站/v1
OPENAI_API_KEY=sk-xxx
OPENAI_MODEL=glm-5.3-flash
```

实测可用的模型示例（任意 OpenAI 兼容服务均可接入）：
- `glm-5.3-flash`：推理模型，输出质量高，但每次消耗思考 token（较慢/略贵）。✅ 已验证
- `minimax-m2.7`：普通 chat 模型，更快更省。✅ 已验证
- 切换模型只需改 `OPENAI_MODEL` 一行。

要点：
- 推理模型（如 glm-5.3-flash）会把思考放进 `reasoning_tokens`，正式答案仍在 `content`；系统已统一后处理剥离 `<think>` 噪声、去首尾空白。
- 为避免触发中转站限流，SentimentAgent 对真实 LLM 调用设了上限（`LLM_CAP=12`），超限自动转词典兜底，情绪矩阵仍完整。
- ReporterAgent 会用 LLM 生成「智能投研结论（LLM 生成）」段落；未配置 Key 时全部降级模板，链路照常可跑。

未配置 Key 时，SentimentAgent / ReporterAgent 自动降级为确定性兜底（词典/模板），**整条链路依旧可跑、可演示**。

---

## 验收对照（需求文档第 8 节）

1. ✅ 给定股票池 + 日期，稳定输出因子评分、情绪分、综合信号、自然语言报告。
2. ✅ 报告含风险提示与因子可解释性（推理链引用具体因子数值）。
3. ✅ 回测模块：动态因子权重组合 vs 静态等权基准，输出收益/回撤/夏普对比。
4. ✅ Streamlit 交互演示页（见 `app.py`：因子权重与回测 / 因子·情绪·得分 / 投研报告 / 风险提示 四页签，含累计收益曲线、动态权重时序、情绪热力图）。
5. ✅ 全程不依赖付费数据即可跑通（Akshare 免费源 + 合成兜底）。

---

## Streamlit 交互演示页

```bash
pip install streamlit plotly
streamlit run app.py
# 浏览器打开 http://localhost:8501
```

页面内容（左侧参数，右侧四页签）：

- **因子权重与回测**：5 个核心指标卡（动态/静态累计收益、夏普、回撤、风险数）+ 累计净值曲线（动态 vs 静态 vs 全样本）+ 本期动态权重 vs 静态等权条形对比 + 权重偏离表。
- **因子·情绪·得分**：动态权重时序堆叠面积图（展示随市场状态 regime 切换）+ 个股综合得分 Top-N + 舆情情绪热力图（个股 × 再平衡日）。
- **投研报告**：完整 Markdown 报告（含 LLM 生成的「智能投研结论」）+ 一键下载。
- **风险提示**：风险清单 + 降级/对冲建议。

技术说明：

- 复用 `run_pipeline` 同一套多智能体链路，页面不重复实现逻辑。
- 图表用 Plotly 渲染（累计收益曲线、权重对比、regime 时序、得分、情绪热力图）。
- 侧边栏实时显示 LLM 模式，并支持**切换 LLM 模型**（`glm-5.3-flash` / `minimax-m2.7` / 离线 Mock），可现场对比不同大模型生成的投研结论；默认数据源 `synthetic` 保证离线稳定，切 `akshare` 即尝试真实行情。
- 已用 Streamlit AppTest 做端到端回归（`test_app.py`），覆盖点击运行后的图表渲染与报告生成，无运行时异常。

> 文档版本 v1.2，2026-09-02（新增 Streamlit 交互演示页 + AppTest 端到端验证）。
