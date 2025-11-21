# Multi-Agent Investment Research Platform 架构设计方案

> 基于BettaFish项目的架构理念设计

## 📋 目录
1. [整体架构设计](#整体架构设计)
2. [核心Agent设计](#核心agent设计)
3. [RAG vs Agent决策](#rag-vs-agent决策)
4. [工作流设计](#工作流设计)
5. [辩证审核机制](#辩证审核机制)
6. [技术实现细节](#技术实现细节)
7. [项目结构](#项目结构)

---

## 1. 整体架构设计

### 1.1 系统架构图

```
用户查询 (股票代码/公司名称)
    ↓
┌─────────────────────────────────────────┐
│   Router Agent (Flask主应用)             │
│   - 任务分发                              │
│   - 并行编排                              │
│   - 进度监控                              │
└─────────────────────────────────────────┘
    ↓
    ├────────────────────────┐
    ↓                        ↓
┌──────────────────┐  ┌──────────────────┐
│ MarketDataAgent  │  │ NewsAnalysisAgent│
│ 市场数据分析      │  │ 新闻情绪分析      │
│                  │  │                  │
│ - 技术指标分析    │  │ - 新闻聚合        │
│ - 财报RAG        │  │ - 情绪识别        │
│ - 市场趋势        │  │ - 舆情分析        │
└──────────────────┘  └──────────────────┘
    │                        │
    └────────────┬───────────┘
                 ↓
    ┌─────────────────────────┐
    │  DialecticalAgent       │
    │  辩证审核与反思          │
    │                         │
    │  - 逻辑一致性检查        │
    │  - 数据矛盾识别          │
    │  - 反向论证              │
    │  - 风险评估              │
    └─────────────────────────┘
                 ↓
          [是否需要重新分析?]
           ↓YES        ↓NO
    [反馈到对应Agent]   │
                       ↓
            ┌──────────────────┐
            │  ReportAgent     │
            │  投资报告生成     │
            │                  │
            │  - 综合分析       │
            │  - 风险评级       │
            │  - 投资建议       │
            └──────────────────┘
```

### 1.2 核心设计理念（借鉴BettaFish）

| BettaFish概念 | Investment Platform映射 | 说明 |
|--------------|------------------------|------|
| InsightEngine | MarketDataAgent | 本地数据库 → 财报数据库 |
| MediaEngine + QueryEngine | NewsAnalysisAgent | 多模态+网络搜索 → 新闻聚合+情绪分析 |
| ForumEngine | DialecticalAgent | 论坛主持人 → 辩证审核员 |
| ReportEngine | ReportAgent | 报告汇总 → 投资报告生成 |
| Node-based工作流 | 统一采用 | 简洁高效，易扩展 |
| State状态管理 | 统一采用 | InvestmentState数据结构 |

---

## 2. 核心Agent设计

### 2.1 Router Agent（任务编排）

**职责**：
- 接收用户查询（股票代码、时间范围、分析需求）
- 并行启动MarketDataAgent和NewsAnalysisAgent
- 监控各Agent进度
- 触发DialecticalAgent审核
- 最终调用ReportAgent

**实现**：
```python
# app.py - Flask主应用
class RouterAgent:
    def __init__(self):
        self.market_agent = MarketDataAgent()
        self.news_agent = NewsAnalysisAgent()
        self.dialectical_agent = DialecticalAgent()
        self.report_agent = ReportAgent()

    def research(self, query: dict):
        # query = {
        #   "ticker": "AAPL",
        #   "start_date": "2024-01-01",
        #   "end_date": "2024-12-31",
        #   "analysis_type": "comprehensive"
        # }

        # 并行启动两个Agent
        with ThreadPoolExecutor(max_workers=2) as executor:
            market_future = executor.submit(
                self.market_agent.research, query
            )
            news_future = executor.submit(
                self.news_agent.research, query
            )

            market_report = market_future.result()
            news_report = news_future.result()

        # 辩证审核（最多3次迭代）
        for iteration in range(3):
            review = self.dialectical_agent.review(
                market_report, news_report
            )

            if review['approved']:
                break

            # 反馈修正
            if 'market_issues' in review:
                market_report = self.market_agent.refine(
                    review['market_issues']
                )
            if 'news_issues' in review:
                news_report = self.news_agent.refine(
                    review['news_issues']
                )

        # 生成最终报告
        final_report = self.report_agent.generate(
            market_report, news_report, review
        )

        return final_report
```

### 2.2 MarketDataAgent（市场数据分析）

**职责**：
- 技术指标计算（MA, RSI, MACD, Bollinger Bands等）
- 财务指标分析（使用10-K RAG）
- 量价分析
- 市场趋势预测

**核心工具集**：
```python
# MarketDataAgent/tools/

1. TechnicalIndicatorTool
   - calculate_moving_averages()
   - calculate_rsi()
   - calculate_macd()
   - calculate_bollinger_bands()
   - calculate_volume_profile()

2. FinancialStatementRAG  ← 回答您的问题：财报作为RAG工具
   - query_10k_filings(ticker, fiscal_year)
   - extract_financial_metrics(query)
   - compare_year_over_year(ticker, metrics)
   - search_md_and_a(query)  # Management Discussion & Analysis

3. MarketDataAPI
   - get_historical_prices(ticker, start, end)
   - get_realtime_quote(ticker)
   - get_earnings_calendar(ticker)
```

**工作流（Node-based）**：
```
query → ReportStructureNode
          ↓
      [Paragraph 1: 技术面分析]
          ├─ FirstSearchNode → TechnicalIndicatorTool
          ├─ FirstSummaryNode
          └─ ReflectionLoop

      [Paragraph 2: 基本面分析]
          ├─ FirstSearchNode → FinancialStatementRAG
          ├─ FirstSummaryNode
          └─ ReflectionLoop

      [Paragraph 3: 量价分析]
          ├─ FirstSearchNode → MarketDataAPI
          ├─ FirstSummaryNode
          └─ ReflectionLoop
          ↓
      ReportFormattingNode → 市场数据分析报告
```

### 2.3 NewsAnalysisAgent（新闻情绪分析）

**职责**：
- 新闻聚合（多源：Twitter, Reddit, News APIs）
- 情绪识别（Positive/Negative/Neutral）
- 舆情趋势分析
- 重大事件提取

**核心工具集**：
```python
# NewsAnalysisAgent/tools/

1. NewsAggregatorTool
   - search_financial_news(ticker, date_range)
   - search_social_media(ticker, platform=['twitter', 'reddit'])
   - get_analyst_reports(ticker)

2. SentimentAnalysisTool
   - analyze_sentiment(texts)  # 使用FinBERT或其他金融情绪模型
   - calculate_sentiment_score(texts)
   - track_sentiment_trend(ticker, time_range)

3. EventExtractionTool
   - extract_material_events(news_list)
   - classify_event_type(event)  # M&A, Earnings, Product Launch等
   - assess_event_impact(event)
```

**工作流（Node-based）**：
```
query → ReportStructureNode
          ↓
      [Paragraph 1: 新闻概览]
          ├─ FirstSearchNode → NewsAggregatorTool
          ├─ FirstSummaryNode
          └─ ReflectionLoop

      [Paragraph 2: 情绪分析]
          ├─ FirstSearchNode → SentimentAnalysisTool
          ├─ FirstSummaryNode
          └─ ReflectionLoop

      [Paragraph 3: 重大事件]
          ├─ FirstSearchNode → EventExtractionTool
          ├─ FirstSummaryNode
          └─ ReflectionLoop
          ↓
      ReportFormattingNode → 新闻情绪分析报告
```

### 2.4 DialecticalAgent（辩证审核）

**职责**（关键创新！）：
- **逻辑一致性检查**：市场数据和新闻情绪是否矛盾？
- **数据矛盾识别**：例如，财报利好但股价下跌，需要深入分析
- **反向论证**：主动寻找反面观点
- **风险评估**：识别被忽视的风险因素

**核心流程**：
```python
# DialecticalAgent/agent.py

class DialecticalAgent:
    def review(self, market_report: str, news_report: str) -> dict:
        """
        辩证审核流程
        """
        # Step 1: 提取关键论点
        market_claims = self._extract_claims(market_report)
        news_claims = self._extract_claims(news_report)

        # Step 2: 识别矛盾
        contradictions = self._identify_contradictions(
            market_claims, news_claims
        )

        # Step 3: 反向论证（Devil's Advocate）
        counter_arguments = self._generate_counter_arguments(
            market_claims + news_claims
        )

        # Step 4: 风险评估
        risks = self._assess_risks(
            market_report, news_report, contradictions
        )

        # Step 5: 决策
        if len(contradictions) > THRESHOLD or len(risks) > RISK_THRESHOLD:
            return {
                'approved': False,
                'market_issues': self._format_feedback(
                    contradictions, 'market'
                ),
                'news_issues': self._format_feedback(
                    contradictions, 'news'
                ),
                'risks': risks,
                'iteration': self.current_iteration
            }
        else:
            return {
                'approved': True,
                'counter_arguments': counter_arguments,
                'risks': risks
            }
```

**Node设计**：
```python
# DialecticalAgent/nodes/

1. ClaimExtractionNode
   - 从报告中提取具体论点
   - 识别论据和结论

2. ContradictionDetectionNode
   - 跨Agent数据对比
   - 逻辑矛盾识别

3. CounterArgumentNode
   - 生成反向论证
   - 寻找被忽视的视角

4. RiskAssessmentNode
   - 识别系统性风险
   - 评估风险等级
```

### 2.5 ReportAgent（投资报告生成）

**职责**：
- 汇总所有Agent的分析结果
- 生成综合投资建议
- 风险评级
- HTML报告输出

**报告模板**：
```
1. Executive Summary (执行摘要)
2. Market Data Analysis (市场数据分析)
   - 技术面
   - 基本面
   - 量价分析
3. News & Sentiment Analysis (新闻情绪分析)
   - 新闻概览
   - 情绪趋势
   - 重大事件
4. Dialectical Review (辩证审核)
   - 逻辑一致性
   - 反向论证
   - 风险评估
5. Investment Recommendation (投资建议)
   - 综合评分
   - 风险等级
   - 建议操作
6. Appendix (附录)
   - 数据来源
   - 模型参数
```

---

## 3. RAG vs Agent决策

### 3.1 为什么10-K财报应该作为RAG而非独立Agent？

| 维度 | RAG工具 | 独立Agent |
|------|---------|----------|
| **数据特性** | 结构化文档，查询驱动 | ✅ | ❌ |
| **交互模式** | 被动检索 | ✅ | ❌ |
| **是否需要反思迭代** | 否，单次查询即可 | ✅ | ❌ |
| **是否需要独立报告** | 否，作为MarketDataAgent的支撑 | ✅ | ❌ |
| **复杂性** | 低，简单检索+提取 | ✅ | ❌ |

**结论**：**10-K财报应该作为RAG工具，集成在MarketDataAgent中**

### 3.2 FinancialStatementRAG设计

```python
# MarketDataAgent/tools/financial_rag.py

from llama_index import VectorStoreIndex, SimpleDirectoryReader
from llama_index.embeddings import OpenAIEmbedding

class FinancialStatementRAG:
    def __init__(self):
        # 加载10-K文档
        self.documents = SimpleDirectoryReader(
            '10k_filings/'  # 目录结构: 10k_filings/AAPL/2023_10K.pdf
        ).load_data()

        # 构建向量索引
        self.index = VectorStoreIndex.from_documents(
            self.documents,
            embed_model=OpenAIEmbedding()
        )

        self.query_engine = self.index.as_query_engine(
            similarity_top_k=5
        )

    def query_10k_filings(self, query: str, ticker: str,
                          fiscal_year: int) -> str:
        """
        查询特定公司特定年份的10-K文档

        Example:
            query = "What was Apple's revenue growth in 2023?"
            result = rag.query_10k_filings(query, "AAPL", 2023)
        """
        # 构建带元数据过滤的查询
        filtered_query = f"[Company: {ticker}, Year: {fiscal_year}] {query}"

        response = self.query_engine.query(filtered_query)

        return {
            'answer': response.response,
            'sources': [node.metadata for node in response.source_nodes],
            'confidence': response.metadata.get('confidence', 0.0)
        }

    def extract_financial_metrics(self, ticker: str,
                                   fiscal_year: int) -> dict:
        """
        自动提取关键财务指标
        """
        metrics = {}

        queries = [
            "Total Revenue",
            "Net Income",
            "Operating Cash Flow",
            "Total Assets",
            "Total Liabilities",
            "Earnings Per Share (EPS)"
        ]

        for metric in queries:
            result = self.query_10k_filings(
                f"What is the {metric}?", ticker, fiscal_year
            )
            metrics[metric] = result['answer']

        return metrics

    def compare_year_over_year(self, ticker: str,
                                current_year: int,
                                metric: str) -> dict:
        """
        同比分析
        """
        current = self.query_10k_filings(
            f"What is the {metric}?", ticker, current_year
        )
        previous = self.query_10k_filings(
            f"What is the {metric}?", ticker, current_year - 1
        )

        return {
            'current_year': current,
            'previous_year': previous,
            'growth_rate': self._calculate_growth(current, previous)
        }
```

**RAG数据准备**：
```bash
# 目录结构
10k_filings/
├── AAPL/
│   ├── 2021_10K.pdf
│   ├── 2022_10K.pdf
│   └── 2023_10K.pdf
├── MSFT/
│   ├── 2021_10K.pdf
│   ├── 2022_10K.pdf
│   └── 2023_10K.pdf
└── metadata.json  # 存储元数据（公司名、年份、文件路径等）
```

---

## 4. 工作流设计

### 4.1 统一的State数据结构

```python
# state/investment_state.py

from dataclasses import dataclass, field
from typing import List, Dict
from datetime import datetime

@dataclass
class InvestmentState:
    """投资研究的全局状态"""

    # 基本信息
    ticker: str                                    # 股票代码
    company_name: str                             # 公司名称
    query_time: datetime = field(default_factory=datetime.now)

    # 市场数据分析结果
    market_analysis: MarketAnalysis = None

    # 新闻情绪分析结果
    news_analysis: NewsAnalysis = None

    # 辩证审核结果
    dialectical_review: DialecticalReview = None

    # 最终报告
    final_report: str = ""

    # 状态标志
    is_market_completed: bool = False
    is_news_completed: bool = False
    is_review_completed: bool = False
    is_all_completed: bool = False

    # 迭代次数
    review_iterations: int = 0

    def save_to_file(self, filepath: str):
        """持久化状态"""
        pass

    @classmethod
    def load_from_file(cls, filepath: str):
        """从文件加载状态"""
        pass

@dataclass
class MarketAnalysis:
    """市场数据分析结果"""
    technical_indicators: Dict[str, float]
    financial_metrics: Dict[str, float]
    price_trend: str  # "bullish", "bearish", "neutral"
    volume_analysis: str
    support_resistance: Dict[str, float]
    recommendation: str  # "buy", "hold", "sell"

@dataclass
class NewsAnalysis:
    """新闻情绪分析结果"""
    sentiment_score: float  # -1 to 1
    sentiment_trend: str    # "improving", "declining", "stable"
    major_events: List[Dict]
    news_count: int
    social_media_sentiment: Dict[str, float]

@dataclass
class DialecticalReview:
    """辩证审核结果"""
    approved: bool
    contradictions: List[str]
    counter_arguments: List[str]
    risks: List[Dict]
    confidence_score: float
```

### 4.2 完整工作流时序图

```
T0: 用户提交查询
    ↓
T1: RouterAgent初始化State
    ↓
T2: 并行启动
    ├─ MarketDataAgent.research()
    │   ├─ T2.1: ReportStructureNode
    │   ├─ T2.2: 技术面分析 (FirstSearch → Summary → Reflection×3)
    │   ├─ T2.3: 基本面分析 (FinancialRAG查询)
    │   ├─ T2.4: 量价分析
    │   └─ T2.5: ReportFormattingNode → market_report.txt
    │
    └─ NewsAnalysisAgent.research()
        ├─ T2.1: ReportStructureNode
        ├─ T2.2: 新闻聚合 (FirstSearch → Summary → Reflection×3)
        ├─ T2.3: 情绪分析
        ├─ T2.4: 重大事件提取
        └─ T2.5: ReportFormattingNode → news_report.txt
    ↓
T3: 等待两个Agent完成
    ↓
T4: DialecticalAgent.review()
    ├─ T4.1: ClaimExtractionNode
    ├─ T4.2: ContradictionDetectionNode
    ├─ T4.3: CounterArgumentNode
    └─ T4.4: RiskAssessmentNode
    ↓
T5: 判断是否需要重新分析
    ├─ YES → 反馈到对应Agent → 回到T2
    └─ NO → 继续T6
    ↓
T6: ReportAgent.generate()
    ├─ T6.1: TemplateSelectionNode
    ├─ T6.2: SummaryNode（综合所有分析）
    └─ T6.3: HTMLGenerationNode → final_report.html
    ↓
T7: 输出最终报告
```

---

## 5. 辩证审核机制（关键创新）

### 5.1 为什么需要辩证审核？

**传统多Agent系统的问题**：
- 各Agent独立工作，缺乏交叉验证
- 可能产生逻辑矛盾（例如：财报利好但新闻负面）
- 缺少反向思考，容易陷入确认偏误

**DialecticalAgent的价值**：
- **质量保证**：确保分析结果的逻辑一致性
- **风险控制**：主动寻找被忽视的风险
- **深度思考**：通过反向论证提升分析深度

### 5.2 辩证审核的三大机制

#### 机制1：逻辑一致性检查

```python
# DialecticalAgent/nodes/contradiction_detection_node.py

class ContradictionDetectionNode(BaseNode):
    def run(self, input_data: dict) -> dict:
        market_report = input_data['market_report']
        news_report = input_data['news_report']

        # 提取关键论点
        market_claims = self._extract_claims(market_report)
        news_claims = self._extract_claims(news_report)

        # 检测矛盾
        contradictions = []

        # Example 1: 价格趋势 vs 情绪趋势
        if market_claims['price_trend'] == 'bullish' and \
           news_claims['sentiment_trend'] == 'negative':
            contradictions.append({
                'type': 'trend_mismatch',
                'market': 'Price trend is bullish',
                'news': 'Sentiment trend is negative',
                'severity': 'high',
                'suggestion': 'Investigate if positive fundamentals are being overshadowed by short-term negative news'
            })

        # Example 2: 基本面 vs 技术面
        if market_claims['financial_health'] == 'strong' and \
           market_claims['technical_signal'] == 'sell':
            contradictions.append({
                'type': 'fundamental_technical_divergence',
                'description': 'Strong fundamentals but weak technicals',
                'severity': 'medium',
                'suggestion': 'May present a buying opportunity if technical weakness is temporary'
            })

        return {'contradictions': contradictions}
```

#### 机制2：反向论证（Devil's Advocate）

```python
# DialecticalAgent/nodes/counter_argument_node.py

class CounterArgumentNode(BaseNode):
    def run(self, input_data: dict) -> dict:
        """
        为每个主要论点生成反向论证
        """
        claims = input_data['claims']

        counter_arguments = []

        for claim in claims:
            # 使用LLM生成反向论证
            prompt = f"""
            Given the following investment claim:
            "{claim['statement']}"

            Your role: Devil's Advocate

            Task: Generate a well-reasoned counter-argument that challenges this claim.
            Consider:
            1. Alternative interpretations of the data
            2. Potential risks or downsides
            3. Market conditions that could invalidate this claim
            4. Historical precedents where similar claims proved wrong

            Provide a structured counter-argument.
            """

            response = self.llm_client.invoke(prompt)

            counter_arguments.append({
                'original_claim': claim['statement'],
                'counter_argument': response,
                'strength': self._assess_strength(response)
            })

        return {'counter_arguments': counter_arguments}
```

#### 机制3：风险评估

```python
# DialecticalAgent/nodes/risk_assessment_node.py

class RiskAssessmentNode(BaseNode):
    def run(self, input_data: dict) -> dict:
        """
        识别系统性风险和特定风险
        """
        market_report = input_data['market_report']
        news_report = input_data['news_report']

        risks = []

        # 1. 市场风险
        market_risks = self._assess_market_risks(market_report)

        # 2. 行业风险
        industry_risks = self._assess_industry_risks(news_report)

        # 3. 公司特定风险
        company_risks = self._assess_company_risks(
            market_report, news_report
        )

        # 4. 宏观经济风险
        macro_risks = self._assess_macro_risks(news_report)

        all_risks = market_risks + industry_risks + \
                    company_risks + macro_risks

        # 按严重性排序
        all_risks.sort(key=lambda x: x['severity'], reverse=True)

        return {'risks': all_risks}
```

### 5.3 反馈与迭代机制

```python
# RouterAgent中的迭代逻辑

MAX_ITERATIONS = 3

for iteration in range(MAX_ITERATIONS):
    # 辩证审核
    review = dialectical_agent.review(market_report, news_report)

    if review['approved']:
        logger.info(f"✅ Approved after {iteration + 1} iteration(s)")
        break

    logger.warning(f"⚠️ Issues found in iteration {iteration + 1}")

    # 生成反馈
    if review.get('market_issues'):
        logger.info("📝 Refining Market Analysis...")
        market_report = market_agent.refine(
            issues=review['market_issues'],
            previous_report=market_report
        )

    if review.get('news_issues'):
        logger.info("📝 Refining News Analysis...")
        news_report = news_agent.refine(
            issues=review['news_issues'],
            previous_report=news_report
        )
else:
    logger.warning(f"⚠️ Max iterations ({MAX_ITERATIONS}) reached")
```

---

## 6. 技术实现细节

### 6.1 关键技术栈

| 组件 | 技术选型 | 说明 |
|------|---------|------|
| **Web框架** | Flask | 轻量，易于集成 |
| **LLM客户端** | OpenAI API | 支持多种模型 |
| **RAG框架** | LlamaIndex | 向量索引+查询引擎 |
| **向量数据库** | ChromaDB / Pinecone | 本地开发用Chroma，生产用Pinecone |
| **市场数据API** | yfinance, Alpha Vantage | 免费+付费结合 |
| **新闻API** | NewsAPI, Tavily | 多源聚合 |
| **情绪分析** | FinBERT, VADER | 金融领域专用模型 |
| **数据库** | PostgreSQL | 存储历史分析结果 |
| **任务队列** | Celery (可选) | 异步任务处理 |

### 6.2 LLM模型选择建议

```python
# config.py

# MarketDataAgent: 需要精确的数值计算和逻辑推理
MARKET_DATA_AGENT_CONFIG = {
    'model': 'gpt-4o',  # 或 'claude-sonnet-3.5'
    'temperature': 0.2,
    'max_tokens': 4000
}

# NewsAnalysisAgent: 需要理解复杂文本和情绪
NEWS_ANALYSIS_AGENT_CONFIG = {
    'model': 'gpt-4o',
    'temperature': 0.3,
    'max_tokens': 4000
}

# DialecticalAgent: 需要强大的推理能力
DIALECTICAL_AGENT_CONFIG = {
    'model': 'claude-opus-4',  # 或 'gpt-4o'
    'temperature': 0.4,
    'max_tokens': 6000
}

# ReportAgent: 需要良好的文本生成能力
REPORT_AGENT_CONFIG = {
    'model': 'gpt-4o',
    'temperature': 0.5,
    'max_tokens': 8000
}
```

### 6.3 数据源集成

#### 市场数据
```python
# MarketDataAgent/tools/market_data_api.py

import yfinance as yf
from alpha_vantage.timeseries import TimeSeries

class MarketDataAPI:
    def get_historical_prices(self, ticker: str,
                              start: str, end: str) -> pd.DataFrame:
        """使用yfinance获取历史价格"""
        stock = yf.Ticker(ticker)
        df = stock.history(start=start, end=end)
        return df

    def get_technical_indicators(self, ticker: str) -> dict:
        """计算技术指标"""
        df = self.get_historical_prices(ticker, '1y')

        # MA
        df['MA_50'] = df['Close'].rolling(window=50).mean()
        df['MA_200'] = df['Close'].rolling(window=200).mean()

        # RSI
        df['RSI'] = self._calculate_rsi(df['Close'])

        # MACD
        df['MACD'], df['Signal'] = self._calculate_macd(df['Close'])

        return {
            'current_price': df['Close'].iloc[-1],
            'ma_50': df['MA_50'].iloc[-1],
            'ma_200': df['MA_200'].iloc[-1],
            'rsi': df['RSI'].iloc[-1],
            'macd': df['MACD'].iloc[-1],
            'signal': df['Signal'].iloc[-1]
        }
```

#### 新闻数据
```python
# NewsAnalysisAgent/tools/news_aggregator.py

from newsapi import NewsApiClient
from tavily import TavilyClient

class NewsAggregatorTool:
    def __init__(self):
        self.newsapi = NewsApiClient(api_key=NEWS_API_KEY)
        self.tavily = TavilyClient(api_key=TAVILY_API_KEY)

    def search_financial_news(self, ticker: str,
                              company_name: str,
                              days: int = 30) -> List[dict]:
        """聚合多源新闻"""

        # NewsAPI
        newsapi_results = self.newsapi.get_everything(
            q=f'{company_name} OR {ticker}',
            language='en',
            sort_by='relevancy',
            from_param=(datetime.now() - timedelta(days=days)).isoformat()
        )

        # Tavily (AI搜索)
        tavily_results = self.tavily.search(
            query=f"{company_name} {ticker} financial news",
            search_depth="advanced"
        )

        # 合并和去重
        all_news = self._merge_and_deduplicate(
            newsapi_results['articles'],
            tavily_results['results']
        )

        return all_news
```

---

## 7. 项目结构

### 7.1 推荐目录结构

```
InvestmentResearchPlatform/
├── app.py                          # Flask主应用（RouterAgent）
├── config.py                       # 全局配置
├── requirements.txt
├── .env                            # 环境变量（API密钥等）
│
├── MarketDataAgent/
│   ├── agent.py                   # 市场数据分析Agent
│   ├── llms/
│   │   └── base.py                # LLM客户端
│   ├── nodes/
│   │   ├── base_node.py
│   │   ├── report_structure_node.py
│   │   ├── search_node.py
│   │   ├── summary_node.py
│   │   └── formatting_node.py
│   ├── state/
│   │   └── state.py               # MarketAnalysisState
│   ├── tools/
│   │   ├── financial_rag.py       # 10-K财报RAG ← 核心
│   │   ├── technical_indicators.py
│   │   └── market_data_api.py
│   ├── prompts/
│   │   └── prompts.py
│   └── utils/
│       └── config.py
│
├── NewsAnalysisAgent/
│   ├── agent.py                   # 新闻情绪分析Agent
│   ├── llms/
│   ├── nodes/
│   ├── state/
│   ├── tools/
│   │   ├── news_aggregator.py
│   │   ├── sentiment_analysis.py
│   │   └── event_extraction.py
│   ├── prompts/
│   └── utils/
│
├── DialecticalAgent/               # 辩证审核Agent ← 创新点
│   ├── agent.py
│   ├── llms/
│   ├── nodes/
│   │   ├── claim_extraction_node.py
│   │   ├── contradiction_detection_node.py
│   │   ├── counter_argument_node.py
│   │   └── risk_assessment_node.py
│   ├── state/
│   ├── prompts/
│   └── utils/
│
├── ReportAgent/
│   ├── agent.py                   # 报告生成Agent
│   ├── llms/
│   ├── nodes/
│   │   ├── template_selection_node.py
│   │   └── html_generation_node.py
│   ├── templates/
│   │   ├── comprehensive_report.html
│   │   ├── quick_analysis.html
│   │   └── risk_focused.html
│   └── utils/
│
├── data/
│   ├── 10k_filings/               # 财报文档存储
│   │   ├── AAPL/
│   │   ├── MSFT/
│   │   └── ...
│   └── vector_stores/             # 向量数据库
│       └── chroma_db/
│
├── outputs/
│   ├── market_reports/            # 市场数据分析报告
│   ├── news_reports/              # 新闻情绪分析报告
│   ├── dialectical_reviews/      # 辩证审核报告
│   └── final_reports/             # 最终投资报告
│
├── logs/
│   ├── market_agent.log
│   ├── news_agent.log
│   ├── dialectical_agent.log
│   └── report_agent.log
│
├── tests/
│   ├── test_market_agent.py
│   ├── test_news_agent.py
│   ├── test_dialectical_agent.py
│   └── test_integration.py
│
└── docs/
    ├── architecture.md
    ├── api_documentation.md
    └── user_guide.md
```

### 7.2 配置文件示例

```python
# config.py

import os
from dotenv import load_dotenv

load_dotenv()

# === Flask配置 ===
HOST = os.getenv('HOST', '0.0.0.0')
PORT = int(os.getenv('PORT', 5000))

# === 数据库配置 ===
DATABASE_URL = os.getenv('DATABASE_URL',
    'postgresql://user:password@localhost:5432/investment_research')

# === LLM配置 ===
MARKET_DATA_AGENT_CONFIG = {
    'api_key': os.getenv('OPENAI_API_KEY'),
    'model': 'gpt-4o',
    'temperature': 0.2,
    'max_tokens': 4000,
    'base_url': os.getenv('OPENAI_BASE_URL', 'https://api.openai.com/v1')
}

NEWS_ANALYSIS_AGENT_CONFIG = {
    'api_key': os.getenv('OPENAI_API_KEY'),
    'model': 'gpt-4o',
    'temperature': 0.3,
    'max_tokens': 4000
}

DIALECTICAL_AGENT_CONFIG = {
    'api_key': os.getenv('ANTHROPIC_API_KEY'),
    'model': 'claude-opus-4',
    'temperature': 0.4,
    'max_tokens': 6000,
    'base_url': 'https://api.anthropic.com'
}

REPORT_AGENT_CONFIG = {
    'api_key': os.getenv('OPENAI_API_KEY'),
    'model': 'gpt-4o',
    'temperature': 0.5,
    'max_tokens': 8000
}

# === RAG配置 ===
VECTOR_STORE_TYPE = 'chroma'  # 'chroma' or 'pinecone'
CHROMA_PERSIST_DIR = './data/vector_stores/chroma_db'
EMBEDDING_MODEL = 'text-embedding-3-large'

# === 市场数据API ===
ALPHA_VANTAGE_API_KEY = os.getenv('ALPHA_VANTAGE_API_KEY')
FINNHUB_API_KEY = os.getenv('FINNHUB_API_KEY')

# === 新闻API ===
NEWS_API_KEY = os.getenv('NEWS_API_KEY')
TAVILY_API_KEY = os.getenv('TAVILY_API_KEY')

# === 工作流参数 ===
MAX_REFLECTIONS = 3              # 每个Agent的最大反思次数
MAX_DIALECTICAL_ITERATIONS = 3   # 辩证审核的最大迭代次数
CONTRADICTION_THRESHOLD = 2      # 触发重新分析的矛盾阈值

# === 报告配置 ===
REPORT_OUTPUT_DIR = './outputs/final_reports'
REPORT_FORMAT = 'html'  # 'html' or 'pdf' or 'markdown'
```

---

## 8. 实施路线图

### Phase 1: 基础架构搭建（2-3周）
- [ ] 搭建项目结构
- [ ] 实现RouterAgent（Flask主应用）
- [ ] 实现BaseNode和StateMutationNode基类
- [ ] 实现InvestmentState数据结构
- [ ] 配置LLM客户端和API集成

### Phase 2: MarketDataAgent（3-4周）
- [ ] 实现技术指标计算工具
- [ ] 搭建FinancialStatementRAG系统
  - [ ] 收集10-K文档
  - [ ] 构建向量索引
  - [ ] 实现查询接口
- [ ] 实现MarketDataAgent的完整工作流
- [ ] 测试和优化

### Phase 3: NewsAnalysisAgent（2-3周）
- [ ] 集成新闻聚合API
- [ ] 实现情绪分析工具（FinBERT）
- [ ] 实现事件提取工具
- [ ] 实现NewsAnalysisAgent的完整工作流
- [ ] 测试和优化

### Phase 4: DialecticalAgent（3-4周）← 核心创新
- [ ] 实现ClaimExtractionNode
- [ ] 实现ContradictionDetectionNode
- [ ] 实现CounterArgumentNode
- [ ] 实现RiskAssessmentNode
- [ ] 实现反馈迭代机制
- [ ] 测试和优化

### Phase 5: ReportAgent（1-2周）
- [ ] 设计HTML报告模板
- [ ] 实现报告生成逻辑
- [ ] 集成所有Agent的输出
- [ ] 测试和优化

### Phase 6: 集成与优化（2-3周）
- [ ] 端到端集成测试
- [ ] 性能优化
- [ ] 日志和监控系统
- [ ] 用户界面开发（可选）
- [ ] 文档编写

---

## 9. 与BettaFish的对比

| 维度 | BettaFish | Investment Platform |
|------|-----------|-------------------|
| **领域** | 舆情分析 | 投资研究 |
| **核心Agent数量** | 4个（Insight, Media, Query, Report） | 4个（Market, News, Dialectical, Report） |
| **数据源** | 本地数据库 + 网络搜索 + 多模态 | 财报RAG + 市场数据API + 新闻API |
| **协作机制** | ForumEngine（论坛主持人） | DialecticalAgent（辩证审核） |
| **RAG应用** | 本地数据库查询 | 10-K财报文档RAG |
| **工作流** | Node-based（统一） | Node-based（统一） |
| **状态管理** | State数据结构 | InvestmentState数据结构 |
| **创新点** | 论坛式Agent协作 | 辩证审核+反向论证 |

---

## 10. 总结

### 核心设计决策

1. **10-K财报作为RAG而非独立Agent** ✅
   - 原因：查询驱动，无需独立反思迭代
   - 实现：集成在MarketDataAgent的工具集中

2. **DialecticalAgent作为审核层** ✅
   - 原因：确保逻辑一致性，主动寻找风险
   - 实现：在并行Agent完成后，进行辩证审核

3. **Node-based工作流** ✅
   - 原因：简洁、易扩展、易理解
   - 实现：借鉴BettaFish，不依赖LangGraph

4. **并行+迭代的混合模式** ✅
   - 原因：提高效率同时保证质量
   - 实现：MarketDataAgent和NewsAnalysisAgent并行，DialecticalAgent串行审核

### 关键优势

1. **质量保证**：辩证审核机制确保分析结果的可靠性
2. **深度分析**：反向论证避免确认偏误
3. **模块化设计**：易于扩展新的Agent和工具
4. **数据驱动**：多源数据融合（财报、市场数据、新闻）
5. **可解释性**：完整的State追踪和日志系统

---

**参考资料**：
- BettaFish项目：https://github.com/666ghj/Weibo_PublicOpinion_AnalysisSystem
- LlamaIndex文档：https://docs.llamaindex.ai/
- FinBERT：https://github.com/ProsusAI/finBERT

**最后更新**：2025-11-21
**版本**：v1.0
**作者**：基于BettaFish架构设计
