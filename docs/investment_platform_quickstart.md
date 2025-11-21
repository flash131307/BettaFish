# Multi-Agent Investment Platform - 快速开始指南

## 🎯 核心架构总结

基于BettaFish的设计理念，您的投资研究平台架构如下：

```
Router Agent (Flask主应用)
    ├─ 并行执行
    │   ├─ MarketDataAgent (市场数据分析)
    │   │   └─ FinancialStatementRAG (10-K财报作为RAG工具)
    │   └─ NewsAnalysisAgent (新闻情绪分析)
    │
    ├─ 串行审核
    │   └─ DialecticalAgent (辩证审核)
    │       ├─ 逻辑一致性检查
    │       ├─ 反向论证
    │       └─ 风险评估 → [需要重新分析?] YES→反馈 / NO→继续
    │
    └─ 报告生成
        └─ ReportAgent (最终报告)
```

## 📝 关键设计决策

### 1. 10-K财报：RAG ✅ vs 独立Agent ❌

**决定：作为RAG工具集成在MarketDataAgent中**

**原因**：
- ✅ 财报是结构化文档，查询驱动
- ✅ 无需独立的反思迭代
- ✅ 作为MarketDataAgent的数据支撑更合理
- ❌ 不需要独立生成报告
- ❌ 不需要独立的工作流

### 2. DialecticalAgent：BettaFish的ForumEngine进化版

| BettaFish ForumEngine | Investment DialecticalAgent |
|----------------------|----------------------------|
| 监听多个Agent日志 | 接收MarketDataAgent和NewsAnalysisAgent的输出 |
| 生成论坛主持人评论 | 进行辩证审核（逻辑检查+反向论证+风险评估） |
| 被动观察 | 主动审核+反馈迭代 |

## 🚀 核心代码示例

### 示例1: FinancialStatementRAG实现

```python
# MarketDataAgent/tools/financial_rag.py

from llama_index.core import VectorStoreIndex, SimpleDirectoryReader
from llama_index.embeddings.openai import OpenAIEmbedding
from llama_index.llms.openai import OpenAI
import os

class FinancialStatementRAG:
    """
    10-K财报RAG系统
    用途：为MarketDataAgent提供财报查询能力
    """

    def __init__(self, filings_dir: str = './data/10k_filings'):
        self.filings_dir = filings_dir

        # 加载所有10-K文档
        print("📄 Loading 10-K filings...")
        self.documents = SimpleDirectoryReader(
            filings_dir,
            recursive=True  # 递归加载子目录
        ).load_data()

        # 构建向量索引
        print("🔍 Building vector index...")
        self.index = VectorStoreIndex.from_documents(
            self.documents,
            embed_model=OpenAIEmbedding(model="text-embedding-3-large")
        )

        # 创建查询引擎
        self.query_engine = self.index.as_query_engine(
            similarity_top_k=5,
            llm=OpenAI(model="gpt-4o", temperature=0.1)
        )

        print("✅ FinancialStatementRAG initialized")

    def query(self, question: str, ticker: str = None,
              fiscal_year: int = None) -> dict:
        """
        查询财报信息

        Args:
            question: 查询问题，例如 "What was Apple's revenue in 2023?"
            ticker: 股票代码（可选），用于过滤
            fiscal_year: 财年（可选），用于过滤

        Returns:
            {
                'answer': str,
                'sources': List[dict],
                'confidence': float
            }
        """
        # 构建带过滤的查询
        if ticker and fiscal_year:
            filtered_query = f"[{ticker} {fiscal_year}] {question}"
        elif ticker:
            filtered_query = f"[{ticker}] {question}"
        else:
            filtered_query = question

        # 执行查询
        response = self.query_engine.query(filtered_query)

        # 提取源文档信息
        sources = []
        if hasattr(response, 'source_nodes'):
            for node in response.source_nodes:
                sources.append({
                    'file': node.metadata.get('file_name', 'Unknown'),
                    'score': node.score,
                    'text_snippet': node.text[:200] + '...'
                })

        return {
            'answer': str(response),
            'sources': sources,
            'confidence': response.metadata.get('confidence', 0.0)
        }

    def extract_financial_metrics(self, ticker: str,
                                   fiscal_year: int) -> dict:
        """
        自动提取关键财务指标

        Returns:
            {
                'revenue': str,
                'net_income': str,
                'eps': str,
                'operating_cash_flow': str,
                ...
            }
        """
        metrics = {}

        # 定义要提取的指标
        queries = {
            'revenue': f"What was {ticker}'s total revenue in fiscal year {fiscal_year}?",
            'net_income': f"What was {ticker}'s net income in fiscal year {fiscal_year}?",
            'eps': f"What was {ticker}'s earnings per share (EPS) in fiscal year {fiscal_year}?",
            'operating_cash_flow': f"What was {ticker}'s operating cash flow in fiscal year {fiscal_year}?",
            'total_assets': f"What were {ticker}'s total assets at the end of fiscal year {fiscal_year}?",
            'total_debt': f"What was {ticker}'s total debt in fiscal year {fiscal_year}?"
        }

        for metric_name, query in queries.items():
            result = self.query(query, ticker, fiscal_year)
            metrics[metric_name] = result['answer']

        return metrics

    def compare_year_over_year(self, ticker: str,
                                current_year: int,
                                metric: str = 'revenue') -> dict:
        """
        同比分析

        Returns:
            {
                'current_year_value': str,
                'previous_year_value': str,
                'growth_analysis': str
            }
        """
        # 查询当前年份
        current_query = f"What was {ticker}'s {metric} in fiscal year {current_year}?"
        current_result = self.query(current_query, ticker, current_year)

        # 查询前一年
        previous_year = current_year - 1
        previous_query = f"What was {ticker}'s {metric} in fiscal year {previous_year}?"
        previous_result = self.query(previous_query, ticker, previous_year)

        # 分析增长
        analysis_query = f"""
        Compare {ticker}'s {metric} between {previous_year} and {current_year}.
        Calculate the growth rate and provide analysis.
        """
        growth_result = self.query(analysis_query, ticker)

        return {
            'current_year_value': current_result['answer'],
            'previous_year_value': previous_result['answer'],
            'growth_analysis': growth_result['answer']
        }


# 使用示例
if __name__ == '__main__':
    # 初始化RAG
    rag = FinancialStatementRAG('./data/10k_filings')

    # 查询示例1: 简单查询
    result = rag.query(
        "What was Apple's revenue in 2023?",
        ticker="AAPL",
        fiscal_year=2023
    )
    print("Answer:", result['answer'])
    print("Sources:", result['sources'])

    # 查询示例2: 提取所有关键指标
    metrics = rag.extract_financial_metrics("AAPL", 2023)
    print("\nKey Metrics:")
    for metric, value in metrics.items():
        print(f"  {metric}: {value}")

    # 查询示例3: 同比分析
    comparison = rag.compare_year_over_year("AAPL", 2023, "revenue")
    print("\nYear-over-Year Comparison:")
    print(f"  2023: {comparison['current_year_value']}")
    print(f"  2022: {comparison['previous_year_value']}")
    print(f"  Analysis: {comparison['growth_analysis']}")
```

### 示例2: DialecticalAgent核心逻辑

```python
# DialecticalAgent/agent.py

from typing import Dict, List
from llms.base import LLMClient
import json

class DialecticalAgent:
    """
    辩证审核Agent
    职责：
    1. 检查MarketDataAgent和NewsAnalysisAgent的逻辑一致性
    2. 生成反向论证（Devil's Advocate）
    3. 评估潜在风险
    4. 决定是否需要重新分析
    """

    def __init__(self, config: dict):
        self.llm_client = LLMClient(
            api_key=config['api_key'],
            model=config['model'],
            temperature=config['temperature']
        )

        self.max_iterations = config.get('max_iterations', 3)
        self.contradiction_threshold = config.get('contradiction_threshold', 2)

    def review(self, market_report: str, news_report: str) -> dict:
        """
        辩证审核主流程

        Returns:
            {
                'approved': bool,
                'contradictions': List[dict],
                'counter_arguments': List[dict],
                'risks': List[dict],
                'feedback': dict  # 如果approved=False，包含反馈
            }
        """
        print("\n🔍 Starting Dialectical Review...")

        # Step 1: 提取关键论点
        market_claims = self._extract_claims(market_report, "market")
        news_claims = self._extract_claims(news_report, "news")
        print(f"  ✓ Extracted {len(market_claims)} market claims")
        print(f"  ✓ Extracted {len(news_claims)} news claims")

        # Step 2: 检测矛盾
        contradictions = self._detect_contradictions(
            market_claims, news_claims
        )
        print(f"  ✓ Found {len(contradictions)} contradictions")

        # Step 3: 生成反向论证
        counter_arguments = self._generate_counter_arguments(
            market_claims + news_claims
        )
        print(f"  ✓ Generated {len(counter_arguments)} counter-arguments")

        # Step 4: 风险评估
        risks = self._assess_risks(market_report, news_report, contradictions)
        print(f"  ✓ Identified {len(risks)} potential risks")

        # Step 5: 决策
        high_severity_contradictions = [
            c for c in contradictions if c['severity'] == 'high'
        ]
        high_severity_risks = [
            r for r in risks if r['severity'] == 'high'
        ]

        if len(high_severity_contradictions) > 0 or \
           len(high_severity_risks) >= self.contradiction_threshold:
            print("  ❌ Review FAILED - Issues found")
            return {
                'approved': False,
                'contradictions': contradictions,
                'counter_arguments': counter_arguments,
                'risks': risks,
                'feedback': self._generate_feedback(
                    contradictions, risks
                )
            }
        else:
            print("  ✅ Review PASSED")
            return {
                'approved': True,
                'contradictions': contradictions,
                'counter_arguments': counter_arguments,
                'risks': risks
            }

    def _extract_claims(self, report: str, source: str) -> List[dict]:
        """提取报告中的关键论点"""
        prompt = f"""
        Analyze the following {source} analysis report and extract all key claims and assertions.

        Report:
        {report}

        For each claim, provide:
        1. The claim statement
        2. Supporting evidence mentioned
        3. Confidence level (high/medium/low)

        Return as JSON array:
        [
            {{
                "claim": "...",
                "evidence": "...",
                "confidence": "high"
            }},
            ...
        ]
        """

        response = self.llm_client.invoke(prompt)

        # 解析JSON
        try:
            claims = json.loads(response)
            for claim in claims:
                claim['source'] = source
            return claims
        except:
            return []

    def _detect_contradictions(self, market_claims: List[dict],
                                news_claims: List[dict]) -> List[dict]:
        """检测跨Agent的矛盾"""
        prompt = f"""
        You are a critical analyst reviewing two sets of claims about the same investment.

        Market Data Claims:
        {json.dumps(market_claims, indent=2)}

        News Analysis Claims:
        {json.dumps(news_claims, indent=2)}

        Task: Identify any contradictions or inconsistencies between these two sets of claims.

        For each contradiction, provide:
        1. Description of the contradiction
        2. Which claims are contradicting
        3. Severity (high/medium/low)
        4. Possible explanation
        5. Recommendation for resolution

        Return as JSON array:
        [
            {{
                "contradiction": "...",
                "market_claim": "...",
                "news_claim": "...",
                "severity": "high",
                "explanation": "...",
                "recommendation": "..."
            }},
            ...
        ]
        """

        response = self.llm_client.invoke(prompt)

        try:
            return json.loads(response)
        except:
            return []

    def _generate_counter_arguments(self, claims: List[dict]) -> List[dict]:
        """生成反向论证（Devil's Advocate）"""
        prompt = f"""
        You are playing the role of Devil's Advocate in an investment analysis.

        Given Claims:
        {json.dumps(claims, indent=2)}

        Task: For each significant claim, generate a well-reasoned counter-argument that challenges it.

        Consider:
        1. Alternative interpretations of the data
        2. Potential risks or downsides being overlooked
        3. Market conditions that could invalidate the claim
        4. Historical precedents where similar claims proved wrong

        Return as JSON array:
        [
            {{
                "original_claim": "...",
                "counter_argument": "...",
                "supporting_points": ["...", "..."],
                "strength": "high"  // high/medium/low
            }},
            ...
        ]
        """

        response = self.llm_client.invoke(prompt)

        try:
            return json.loads(response)
        except:
            return []

    def _assess_risks(self, market_report: str, news_report: str,
                      contradictions: List[dict]) -> List[dict]:
        """评估风险"""
        prompt = f"""
        You are a risk assessment expert reviewing an investment analysis.

        Market Report:
        {market_report}

        News Report:
        {news_report}

        Identified Contradictions:
        {json.dumps(contradictions, indent=2)}

        Task: Identify all potential risks that may have been overlooked or understated.

        Categories to consider:
        1. Market risks (volatility, liquidity, etc.)
        2. Company-specific risks (management, competition, etc.)
        3. Industry risks (regulation, disruption, etc.)
        4. Macroeconomic risks (recession, inflation, etc.)

        For each risk, provide:
        1. Risk description
        2. Category
        3. Severity (high/medium/low)
        4. Likelihood (high/medium/low)
        5. Potential impact
        6. Mitigation suggestions

        Return as JSON array:
        [
            {{
                "risk": "...",
                "category": "market",
                "severity": "high",
                "likelihood": "medium",
                "impact": "...",
                "mitigation": "..."
            }},
            ...
        ]
        """

        response = self.llm_client.invoke(prompt)

        try:
            return json.loads(response)
        except:
            return []

    def _generate_feedback(self, contradictions: List[dict],
                           risks: List[dict]) -> dict:
        """生成反馈，指导Agent重新分析"""
        market_feedback = []
        news_feedback = []

        for contradiction in contradictions:
            if contradiction.get('market_claim'):
                market_feedback.append({
                    'issue': contradiction['contradiction'],
                    'recommendation': contradiction['recommendation']
                })
            if contradiction.get('news_claim'):
                news_feedback.append({
                    'issue': contradiction['contradiction'],
                    'recommendation': contradiction['recommendation']
                })

        # 添加高风险项
        for risk in risks:
            if risk['severity'] == 'high':
                if risk['category'] in ['market', 'company-specific']:
                    market_feedback.append({
                        'issue': f"Risk not addressed: {risk['risk']}",
                        'recommendation': risk['mitigation']
                    })
                else:
                    news_feedback.append({
                        'issue': f"Risk not addressed: {risk['risk']}",
                        'recommendation': risk['mitigation']
                    })

        return {
            'market_issues': market_feedback,
            'news_issues': news_feedback
        }


# 使用示例
if __name__ == '__main__':
    config = {
        'api_key': 'sk-...',
        'model': 'gpt-4o',
        'temperature': 0.4,
        'max_iterations': 3,
        'contradiction_threshold': 2
    }

    dialectical_agent = DialecticalAgent(config)

    # 模拟报告
    market_report = """
    Technical Analysis: Strong bullish trend with RSI at 75 (overbought).
    Fundamental Analysis: Revenue growth of 15% YoY, strong cash flow.
    Recommendation: BUY with target price of $200.
    """

    news_report = """
    Sentiment Analysis: Overall negative sentiment (-0.6) in the past 30 days.
    Major Events: CEO resignation announced, regulatory investigation ongoing.
    Social Media: Growing concerns about product quality issues.
    """

    # 审核
    result = dialectical_agent.review(market_report, news_report)

    if not result['approved']:
        print("\n❌ Review Failed:")
        print("\nContradictions:")
        for c in result['contradictions']:
            print(f"  - {c['contradiction']} (Severity: {c['severity']})")

        print("\nFeedback for MarketDataAgent:")
        for f in result['feedback']['market_issues']:
            print(f"  - {f['issue']}")
            print(f"    → {f['recommendation']}")

        print("\nFeedback for NewsAnalysisAgent:")
        for f in result['feedback']['news_issues']:
            print(f"  - {f['issue']}")
            print(f"    → {f['recommendation']}")
```

### 示例3: Router Agent主流程

```python
# app.py - Flask主应用

from flask import Flask, request, jsonify
from concurrent.futures import ThreadPoolExecutor
from MarketDataAgent.agent import MarketDataAgent
from NewsAnalysisAgent.agent import NewsAnalysisAgent
from DialecticalAgent.agent import DialecticalAgent
from ReportAgent.agent import ReportAgent
import config

app = Flask(__name__)

# 初始化所有Agent
market_agent = MarketDataAgent(config.MARKET_DATA_AGENT_CONFIG)
news_agent = NewsAnalysisAgent(config.NEWS_ANALYSIS_AGENT_CONFIG)
dialectical_agent = DialecticalAgent(config.DIALECTICAL_AGENT_CONFIG)
report_agent = ReportAgent(config.REPORT_AGENT_CONFIG)

@app.route('/api/research', methods=['POST'])
def research():
    """
    主要API端点

    Request Body:
    {
        "ticker": "AAPL",
        "start_date": "2024-01-01",
        "end_date": "2024-12-31",
        "analysis_type": "comprehensive"
    }
    """
    data = request.json
    ticker = data.get('ticker')

    print(f"\n{'='*60}")
    print(f"🚀 Starting Investment Research for {ticker}")
    print(f"{'='*60}\n")

    # Step 1: 并行执行MarketDataAgent和NewsAnalysisAgent
    print("📊 Step 1: Parallel Analysis")
    with ThreadPoolExecutor(max_workers=2) as executor:
        market_future = executor.submit(market_agent.research, data)
        news_future = executor.submit(news_agent.research, data)

        market_report = market_future.result()
        news_report = news_future.result()

    print("  ✅ MarketDataAgent completed")
    print("  ✅ NewsAnalysisAgent completed")

    # Step 2: 辩证审核（最多3次迭代）
    print("\n🔍 Step 2: Dialectical Review")
    for iteration in range(config.MAX_DIALECTICAL_ITERATIONS):
        print(f"\n  Iteration {iteration + 1}:")

        review = dialectical_agent.review(market_report, news_report)

        if review['approved']:
            print(f"  ✅ Approved after {iteration + 1} iteration(s)")
            break

        print(f"  ⚠️ Issues found, requesting refinement...")

        # 并行重新分析
        with ThreadPoolExecutor(max_workers=2) as executor:
            futures = []

            if review['feedback'].get('market_issues'):
                print("    → Refining Market Analysis")
                futures.append(executor.submit(
                    market_agent.refine,
                    review['feedback']['market_issues'],
                    market_report
                ))
            else:
                futures.append(None)

            if review['feedback'].get('news_issues'):
                print("    → Refining News Analysis")
                futures.append(executor.submit(
                    news_agent.refine,
                    review['feedback']['news_issues'],
                    news_report
                ))
            else:
                futures.append(None)

            # 收集结果
            if futures[0]:
                market_report = futures[0].result()
            if futures[1]:
                news_report = futures[1].result()
    else:
        print(f"  ⚠️ Max iterations ({config.MAX_DIALECTICAL_ITERATIONS}) reached")

    # Step 3: 生成最终报告
    print("\n📝 Step 3: Generating Final Report")
    final_report = report_agent.generate({
        'ticker': ticker,
        'market_report': market_report,
        'news_report': news_report,
        'dialectical_review': review
    })

    print("  ✅ Final report generated")
    print(f"\n{'='*60}")
    print("✨ Research Complete!")
    print(f"{'='*60}\n")

    return jsonify({
        'status': 'success',
        'ticker': ticker,
        'market_analysis': market_report,
        'news_analysis': news_report,
        'dialectical_review': {
            'contradictions': review['contradictions'],
            'counter_arguments': review['counter_arguments'],
            'risks': review['risks']
        },
        'final_report': final_report,
        'report_url': f"/reports/{ticker}_{data['end_date']}.html"
    })

if __name__ == '__main__':
    app.run(host=config.HOST, port=config.PORT, debug=True)
```

## 📦 数据准备

### 10-K文档组织结构

```bash
data/10k_filings/
├── AAPL/
│   ├── metadata.json
│   ├── 2021_10K.pdf
│   ├── 2022_10K.pdf
│   └── 2023_10K.pdf
├── MSFT/
│   ├── metadata.json
│   ├── 2021_10K.pdf
│   ├── 2022_10K.pdf
│   └── 2023_10K.pdf
└── TSLA/
    ├── metadata.json
    └── 2023_10K.pdf
```

**metadata.json示例**：
```json
{
  "ticker": "AAPL",
  "company_name": "Apple Inc.",
  "filings": [
    {
      "fiscal_year": 2023,
      "file_path": "2023_10K.pdf",
      "filing_date": "2023-11-03",
      "period_end": "2023-09-30"
    },
    {
      "fiscal_year": 2022,
      "file_path": "2022_10K.pdf",
      "filing_date": "2022-10-28",
      "period_end": "2022-09-24"
    }
  ]
}
```

### 获取10-K文档的方法

```python
# scripts/download_10k.py

from sec_edgar_downloader import Downloader

def download_10k_filings(ticker: str, num_filings: int = 3):
    """
    从SEC EDGAR下载10-K文档

    Args:
        ticker: 股票代码
        num_filings: 下载最近几份10-K
    """
    dl = Downloader("YourCompany", "your_email@example.com")

    dl.get("10-K", ticker, amount=num_filings, download_details=True)

    print(f"✅ Downloaded {num_filings} 10-K filings for {ticker}")

# 使用
download_10k_filings("AAPL", 3)
download_10k_filings("MSFT", 3)
download_10k_filings("TSLA", 3)
```

## 🎯 下一步行动

1. **克隆BettaFish项目**，学习其Node-based工作流实现
   ```bash
   cd /home/user/BettaFish
   # 重点阅读：
   # - InsightEngine/agent.py (Agent主流程)
   # - InsightEngine/nodes/ (Node实现)
   # - InsightEngine/state/state.py (State数据结构)
   ```

2. **创建项目结构**
   ```bash
   mkdir InvestmentResearchPlatform
   cd InvestmentResearchPlatform
   # 按照docs/investment_platform_architecture.md中的目录结构创建
   ```

3. **实现FinancialStatementRAG**（最核心的部分）
   - 收集10-K文档
   - 构建向量索引
   - 测试查询功能

4. **实现DialecticalAgent**（最具创新性的部分）
   - 实现矛盾检测
   - 实现反向论证
   - 实现风险评估

5. **集成测试**
   - 端到端测试完整流程
   - 优化Prompt
   - 调整审核阈值

## 📚 参考资料

- **BettaFish源码**：`/home/user/BettaFish/`
- **详细架构文档**：`/home/user/BettaFish/docs/investment_platform_architecture.md`
- **LlamaIndex文档**：https://docs.llamaindex.ai/
- **SEC EDGAR API**：https://www.sec.gov/edgar
