# 投资研究平台扩展架构：会话管理 + 投资组合追踪 + 双重记忆体系

> 基于BettaFish架构扩展支持持久化、会话管理和记忆机制

## 📋 目录
1. [扩展需求分析](#扩展需求分析)
2. [MongoDB双重记忆体系设计](#mongodb双重记忆体系设计)
3. [会话管理机制](#会话管理机制)
4. [投资组合追踪](#投资组合追踪)
5. [基础风险分析](#基础风险分析)
6. [完整架构设计](#完整架构设计)
7. [实现roadmap](#实现roadmap)

---

## 1. 扩展需求分析

### 1.1 当前BettaFish的持久化机制

**现状**：
```python
# BettaFish目前的持久化
├── MySQL数据库 → 舆情数据（微博、新闻等）
├── 文件系统 → State临时保存（JSON）
└── 日志文件 → agent发言、advisor建议
```

**局限性**：
- ❌ 无会话管理（每次查询独立）
- ❌ 无历史记录追踪
- ❌ 无用户偏好记忆
- ❌ 无实体关系图谱
- ❌ 无投资组合管理

---

### 1.2 扩展功能需求

#### 1️⃣ 会话管理机制

**需求**：
- 支持多轮对话
- 保存历史分析记录
- 追踪用户查询历史
- 支持会话恢复

**场景示例**：
```
用户会话1:
T1: "分析AAPL的投资价值"
T2: "对比AAPL和MSFT"  ← 需要记住之前分析过AAPL
T3: "AAPL的风险有哪些" ← 需要引用T1的分析结果

用户会话2:
T1: "上次分析的AAPL现在怎么样了？" ← 需要找到历史会话
```

---

#### 2️⃣ 投资组合追踪

**需求**：
- 用户自定义投资组合
- 追踪每个标的的历史分析
- 组合级别的风险评估
- 定期自动更新分析

**场景示例**：
```python
portfolio = {
    "name": "科技股组合",
    "holdings": [
        {"ticker": "AAPL", "weight": 0.4},
        {"ticker": "MSFT", "weight": 0.3},
        {"ticker": "GOOGL", "weight": 0.3}
    ]
}

# 功能：
- 追踪每个标的的分析历史
- 自动触发定期分析（每周/每月）
- 组合级别的风险评估
- 相关性分析
```

---

#### 3️⃣ 基础风险分析

**需求**：
- 单标的风险评估
- 组合风险评估
- 风险指标追踪
- 风险预警

**风险指标**：
- 波动率（Volatility）
- Beta系数
- VaR（Value at Risk）
- 最大回撤
- 夏普比率

---

#### 4️⃣ MongoDB双重记忆体系

**两层记忆**：
```
Layer 1: 对话上下文记忆
├── 用户查询历史
├── Agent分析结果
├── 用户反馈
└── 会话元数据

Layer 2: 实体关系记忆
├── 公司实体（ticker, name, sector...）
├── 分析实体（报告、指标、建议...）
├── 关系图谱（公司关联、行业关系...）
└── 知识图谱（概念、事件、人物...）
```

---

## 2. MongoDB双重记忆体系设计

### 2.1 整体架构

```
MongoDB数据库
├── conversations (会话集合) ← Layer 1: 对话上下文
├── messages (消息集合)
├── portfolios (组合集合)
├── analyses (分析集合)
├── entities (实体集合) ← Layer 2: 实体关系
├── relationships (关系集合)
└── risk_metrics (风险指标集合)
```

---

### 2.2 Layer 1: 对话上下文记忆

#### Collection: conversations

```python
{
    "_id": ObjectId("..."),
    "conversation_id": "conv_20250121_001",
    "user_id": "user_123",
    "title": "AAPL投资分析",  # 自动生成或用户命名
    "created_at": ISODate("2025-01-21T10:00:00Z"),
    "updated_at": ISODate("2025-01-21T11:30:00Z"),
    "status": "active",  # active, archived, deleted
    "metadata": {
        "session_count": 3,  # 会话轮次
        "total_messages": 15,
        "primary_tickers": ["AAPL", "MSFT"],  # 涉及的主要标的
        "tags": ["tech", "comparison", "risk-analysis"]
    },
    "summary": "用户分析了AAPL和MSFT，重点关注风险和估值"  # LLM生成的摘要
}
```

#### Collection: messages

```python
{
    "_id": ObjectId("..."),
    "message_id": "msg_20250121_001",
    "conversation_id": "conv_20250121_001",  # 外键
    "role": "user",  # user, assistant, system
    "content": "分析AAPL的投资价值",
    "timestamp": ISODate("2025-01-21T10:00:00Z"),

    # 如果是assistant消息，包含完整的分析结果
    "analysis_result": {
        "ticker": "AAPL",
        "analysis_id": "analysis_20250121_001",  # 外键到analyses集合
        "market_analysis": {
            "technical": "...",
            "fundamental": "..."
        },
        "news_analysis": {
            "sentiment_score": 0.65,
            "major_events": [...]
        },
        "advisor_insights": "...",
        "recommendation": "BUY",
        "risk_level": "MEDIUM"
    },

    # 消息元数据
    "metadata": {
        "tokens_used": 5000,
        "execution_time_ms": 45000,
        "agents_involved": ["MarketDataAgent", "NewsAnalysisAgent"]
    }
}
```

---

### 2.3 Layer 2: 实体关系记忆

#### Collection: entities

**公司实体**：
```python
{
    "_id": ObjectId("..."),
    "entity_id": "entity_AAPL",
    "entity_type": "company",
    "ticker": "AAPL",
    "name": "Apple Inc.",
    "sector": "Technology",
    "industry": "Consumer Electronics",

    # 基础信息
    "info": {
        "market_cap": 3000000000000,
        "employees": 164000,
        "founded": "1976-04-01",
        "hq": "Cupertino, CA"
    },

    # 分析历史（引用）
    "analysis_history": [
        {
            "analysis_id": "analysis_20250121_001",
            "date": ISODate("2025-01-21"),
            "recommendation": "BUY",
            "risk_level": "MEDIUM"
        },
        # ... 历史分析记录
    ],

    # 关系图谱（引用）
    "relationships": [
        {
            "related_entity_id": "entity_MSFT",
            "relationship_type": "competitor",
            "strength": 0.9
        },
        {
            "related_entity_id": "entity_TIM_COOK",
            "relationship_type": "ceo",
            "strength": 1.0
        }
    ],

    # 记忆元数据
    "first_mentioned": ISODate("2025-01-15"),
    "last_analyzed": ISODate("2025-01-21"),
    "mention_count": 15,
    "created_at": ISODate("2025-01-15"),
    "updated_at": ISODate("2025-01-21")
}
```

**概念实体**：
```python
{
    "_id": ObjectId("..."),
    "entity_id": "entity_AI_CHIP",
    "entity_type": "concept",
    "name": "AI芯片",
    "description": "用于AI计算的专用芯片",

    # 关联公司
    "related_companies": [
        {"entity_id": "entity_NVDA", "relevance": 1.0},
        {"entity_id": "entity_AAPL", "relevance": 0.6}
    ],

    # 提及历史
    "mentions": [
        {
            "conversation_id": "conv_20250121_001",
            "context": "AAPL在AI芯片领域的布局",
            "timestamp": ISODate("2025-01-21")
        }
    ]
}
```

---

#### Collection: relationships

```python
{
    "_id": ObjectId("..."),
    "relationship_id": "rel_001",
    "source_entity_id": "entity_AAPL",
    "target_entity_id": "entity_MSFT",
    "relationship_type": "competitor",

    # 关系属性
    "properties": {
        "strength": 0.9,  # 关系强度
        "sentiment": "neutral",
        "context": "同属科技巨头，产品线有竞争"
    },

    # 关系历史
    "history": [
        {
            "date": ISODate("2025-01-21"),
            "event": "用户对比分析AAPL和MSFT",
            "context": "..."
        }
    ],

    "created_at": ISODate("2025-01-21"),
    "updated_at": ISODate("2025-01-21")
}
```

**支持的关系类型**：
- `competitor`: 竞争关系
- `supplier`: 供应商关系
- `customer`: 客户关系
- `partner`: 合作伙伴
- `same_sector`: 同行业
- `correlation`: 相关性（股价相关）
- `executive`: 高管关系
- `ownership`: 持股关系

---

### 2.4 Collection: analyses（分析记录）

```python
{
    "_id": ObjectId("..."),
    "analysis_id": "analysis_20250121_001",
    "ticker": "AAPL",
    "user_id": "user_123",
    "conversation_id": "conv_20250121_001",

    # 分析时间
    "analysis_date": ISODate("2025-01-21T10:00:00Z"),
    "data_period": {
        "start": ISODate("2024-01-21"),
        "end": ISODate("2025-01-21")
    },

    # 分析结果（完整）
    "market_analysis": {
        "technical_indicators": {
            "rsi": 65,
            "macd": {"macd": 2.5, "signal": 1.8},
            "moving_averages": {
                "ma_50": 175.2,
                "ma_200": 168.5
            }
        },
        "fundamental_metrics": {
            "pe_ratio": 28.5,
            "revenue_growth": 0.12,
            "profit_margin": 0.25
        },
        "price_trend": "bullish",
        "support_levels": [170, 165],
        "resistance_levels": [180, 185]
    },

    "news_analysis": {
        "sentiment_score": 0.65,
        "sentiment_trend": "improving",
        "major_events": [
            {
                "event": "新产品发布",
                "date": ISODate("2025-01-15"),
                "impact": "positive"
            }
        ],
        "news_count": 150,
        "social_sentiment": {
            "twitter": 0.7,
            "reddit": 0.6
        }
    },

    "advisor_insights": {
        "contradictions": [],
        "risks": [
            {
                "risk": "市场超买风险",
                "severity": "medium",
                "description": "RSI达到65，接近超买区域"
            }
        ],
        "counter_arguments": [
            "尽管技术面超买，但基本面支撑强劲"
        ]
    },

    # 最终建议
    "recommendation": {
        "action": "BUY",  # BUY, HOLD, SELL
        "confidence": 0.75,
        "target_price": 195.0,
        "stop_loss": 165.0,
        "reasoning": "技术面和基本面均支撑上涨，新闻情绪积极"
    },

    # 风险评估
    "risk_assessment": {
        "overall_risk": "MEDIUM",  # LOW, MEDIUM, HIGH
        "risk_factors": [
            {"factor": "市场波动", "score": 0.6},
            {"factor": "估值过高", "score": 0.5}
        ],
        "risk_score": 0.55
    },

    # 元数据
    "metadata": {
        "agents_used": ["MarketDataAgent", "NewsAnalysisAgent"],
        "execution_time_ms": 45000,
        "data_sources": ["yfinance", "NewsAPI", "10-K RAG"],
        "llm_tokens": 5000
    },

    "created_at": ISODate("2025-01-21T10:00:00Z")
}
```

---

## 3. 会话管理机制

### 3.1 会话生命周期

```python
# session_manager.py

from pymongo import MongoClient
from datetime import datetime
from typing import List, Dict, Optional

class SessionManager:
    """会话管理器"""

    def __init__(self, mongo_uri: str, db_name: str = "investment_platform"):
        self.client = MongoClient(mongo_uri)
        self.db = self.client[db_name]
        self.conversations = self.db.conversations
        self.messages = self.db.messages

    def create_conversation(self, user_id: str, title: str = None) -> str:
        """创建新会话"""
        conversation_id = f"conv_{datetime.now().strftime('%Y%m%d_%H%M%S')}"

        conversation = {
            "conversation_id": conversation_id,
            "user_id": user_id,
            "title": title or "新建投资分析",
            "created_at": datetime.utcnow(),
            "updated_at": datetime.utcnow(),
            "status": "active",
            "metadata": {
                "session_count": 0,
                "total_messages": 0,
                "primary_tickers": [],
                "tags": []
            }
        }

        self.conversations.insert_one(conversation)
        return conversation_id

    def add_message(self, conversation_id: str, role: str,
                    content: str, analysis_result: Dict = None):
        """添加消息到会话"""
        message_id = f"msg_{datetime.now().strftime('%Y%m%d_%H%M%S%f')}"

        message = {
            "message_id": message_id,
            "conversation_id": conversation_id,
            "role": role,
            "content": content,
            "timestamp": datetime.utcnow(),
            "analysis_result": analysis_result,
            "metadata": {}
        }

        self.messages.insert_one(message)

        # 更新conversation统计
        self.conversations.update_one(
            {"conversation_id": conversation_id},
            {
                "$inc": {"metadata.total_messages": 1},
                "$set": {"updated_at": datetime.utcnow()}
            }
        )

        return message_id

    def get_conversation_history(self, conversation_id: str,
                                  limit: int = 50) -> List[Dict]:
        """获取会话历史"""
        messages = self.messages.find(
            {"conversation_id": conversation_id}
        ).sort("timestamp", 1).limit(limit)

        return list(messages)

    def get_user_conversations(self, user_id: str,
                               limit: int = 20) -> List[Dict]:
        """获取用户的所有会话"""
        conversations = self.conversations.find(
            {"user_id": user_id, "status": "active"}
        ).sort("updated_at", -1).limit(limit)

        return list(conversations)

    def find_relevant_context(self, conversation_id: str,
                              ticker: str) -> List[Dict]:
        """查找相关的历史上下文"""
        # 在当前会话中查找关于该ticker的历史分析
        messages = self.messages.find({
            "conversation_id": conversation_id,
            "analysis_result.ticker": ticker
        }).sort("timestamp", -1).limit(3)

        return list(messages)

    def update_conversation_metadata(self, conversation_id: str,
                                     ticker: str = None, tags: List[str] = None):
        """更新会话元数据"""
        update_doc = {"$set": {"updated_at": datetime.utcnow()}}

        if ticker:
            update_doc["$addToSet"] = {"metadata.primary_tickers": ticker}

        if tags:
            update_doc["$addToSet"] = {"metadata.tags": {"$each": tags}}

        self.conversations.update_one(
            {"conversation_id": conversation_id},
            update_doc
        )
```

---

### 3.2 集成到Agent工作流

```python
# MarketDataAgent/agent.py

class MarketDataAgent:
    def __init__(self, config, session_manager: SessionManager = None):
        # ... 原有初始化
        self.session_manager = session_manager

    def research(self, query: dict, conversation_id: str = None):
        """
        研究流程（增强版，支持会话上下文）

        Args:
            query: {
                'ticker': 'AAPL',
                'user_query': '分析AAPL的投资价值',
                'conversation_id': 'conv_xxx'  # 可选
            }
        """
        ticker = query['ticker']

        # 1. 查找历史上下文（如果有会话ID）
        historical_context = None
        if conversation_id and self.session_manager:
            historical_context = self.session_manager.find_relevant_context(
                conversation_id, ticker
            )

        # 2. 执行分析（原有流程）
        # 如果有历史上下文，添加到第一个SummaryNode的prompt中
        if historical_context:
            context_summary = self._summarize_historical_context(historical_context)
            # 将context_summary添加到prompt

        market_report = self._execute_research(query)

        # 3. 保存分析结果到MongoDB
        if self.session_manager:
            self.session_manager.add_message(
                conversation_id,
                role="assistant",
                content=f"Market analysis for {ticker}",
                analysis_result={
                    "ticker": ticker,
                    "analysis_id": f"analysis_{datetime.now().strftime('%Y%m%d_%H%M%S')}",
                    "market_analysis": market_report,
                    # ... 其他结果
                }
            )

            # 更新conversation元数据
            self.session_manager.update_conversation_metadata(
                conversation_id,
                ticker=ticker,
                tags=["market-analysis"]
            )

        return market_report
```

---

## 4. 投资组合追踪

### 4.1 Collection: portfolios

```python
{
    "_id": ObjectId("..."),
    "portfolio_id": "portfolio_001",
    "user_id": "user_123",
    "name": "科技股组合",
    "description": "关注AI和云计算领域的科技公司",

    # 持仓配置
    "holdings": [
        {
            "ticker": "AAPL",
            "weight": 0.4,
            "target_weight": 0.4,
            "shares": 100,
            "avg_cost": 150.0,
            "current_price": 175.0,  # 自动更新
            "unrealized_pnl": 2500.0,
            "last_updated": ISODate("2025-01-21T10:00:00Z")
        },
        {
            "ticker": "MSFT",
            "weight": 0.3,
            "shares": 50,
            "avg_cost": 280.0
        },
        {
            "ticker": "GOOGL",
            "weight": 0.3,
            "shares": 40,
            "avg_cost": 130.0
        }
    ],

    # 组合统计
    "statistics": {
        "total_value": 50000.0,
        "total_cost": 45000.0,
        "total_pnl": 5000.0,
        "total_return": 0.111,  # 11.1%
        "diversification_score": 0.75
    },

    # 风险指标（定期计算）
    "risk_metrics": {
        "volatility": 0.25,
        "beta": 1.15,
        "sharpe_ratio": 1.2,
        "max_drawdown": -0.15,
        "var_95": -0.05,  # 95% VaR
        "last_calculated": ISODate("2025-01-21T00:00:00Z")
    },

    # 分析历史
    "analysis_history": [
        {
            "analysis_id": "portfolio_analysis_20250121",
            "date": ISODate("2025-01-21"),
            "overall_recommendation": "HOLD",
            "rebalancing_needed": true,
            "rebalancing_suggestions": [
                {"ticker": "AAPL", "action": "reduce", "from_weight": 0.45, "to_weight": 0.4}
            ]
        }
    ],

    # 自动分析设置
    "auto_analysis": {
        "enabled": true,
        "frequency": "weekly",  # daily, weekly, monthly
        "next_run": ISODate("2025-01-28T09:00:00Z"),
        "alerts": {
            "risk_threshold": 0.3,  # 触发预警的风险阈值
            "drawdown_threshold": -0.2  # 回撤超过20%预警
        }
    },

    "created_at": ISODate("2025-01-15"),
    "updated_at": ISODate("2025-01-21"),
    "status": "active"
}
```

---

### 4.2 PortfolioManager实现

```python
# portfolio_manager.py

import numpy as np
from typing import List, Dict
from datetime import datetime, timedelta

class PortfolioManager:
    """投资组合管理器"""

    def __init__(self, mongo_uri: str, db_name: str = "investment_platform"):
        self.client = MongoClient(mongo_uri)
        self.db = self.client[db_name]
        self.portfolios = self.db.portfolios
        self.analyses = self.db.analyses

    def create_portfolio(self, user_id: str, name: str,
                        holdings: List[Dict]) -> str:
        """创建投资组合"""
        portfolio_id = f"portfolio_{datetime.now().strftime('%Y%m%d_%H%M%S')}"

        portfolio = {
            "portfolio_id": portfolio_id,
            "user_id": user_id,
            "name": name,
            "holdings": holdings,
            "statistics": {},
            "risk_metrics": {},
            "created_at": datetime.utcnow(),
            "updated_at": datetime.utcnow(),
            "status": "active"
        }

        self.portfolios.insert_one(portfolio)

        # 立即计算组合统计
        self.update_portfolio_statistics(portfolio_id)

        return portfolio_id

    def update_portfolio_statistics(self, portfolio_id: str):
        """更新组合统计数据"""
        portfolio = self.portfolios.find_one({"portfolio_id": portfolio_id})

        if not portfolio:
            return

        holdings = portfolio['holdings']

        # 计算总价值、收益等
        total_value = sum(h['shares'] * h.get('current_price', h['avg_cost'])
                         for h in holdings)
        total_cost = sum(h['shares'] * h['avg_cost'] for h in holdings)
        total_pnl = total_value - total_cost
        total_return = total_pnl / total_cost if total_cost > 0 else 0

        # 计算分散化得分（基于权重分布）
        weights = [h.get('weight', 0) for h in holdings]
        diversification_score = 1 - sum(w**2 for w in weights)

        statistics = {
            "total_value": total_value,
            "total_cost": total_cost,
            "total_pnl": total_pnl,
            "total_return": total_return,
            "diversification_score": diversification_score
        }

        self.portfolios.update_one(
            {"portfolio_id": portfolio_id},
            {
                "$set": {
                    "statistics": statistics,
                    "updated_at": datetime.utcnow()
                }
            }
        )

    def calculate_portfolio_risk(self, portfolio_id: str,
                                 historical_data: Dict[str, np.ndarray]):
        """
        计算组合风险指标

        Args:
            portfolio_id: 组合ID
            historical_data: {ticker: returns_array} 历史收益率数据
        """
        portfolio = self.portfolios.find_one({"portfolio_id": portfolio_id})
        holdings = portfolio['holdings']

        # 提取权重和收益率
        tickers = [h['ticker'] for h in holdings]
        weights = np.array([h.get('weight', 0) for h in holdings])

        # 构建收益率矩阵
        returns_matrix = np.column_stack([
            historical_data[ticker] for ticker in tickers
        ])

        # 计算组合收益率
        portfolio_returns = returns_matrix @ weights

        # 计算风险指标
        volatility = np.std(portfolio_returns) * np.sqrt(252)  # 年化波动率

        # Beta（相对于市场，这里简化为SPY）
        if 'SPY' in historical_data:
            market_returns = historical_data['SPY']
            covariance = np.cov(portfolio_returns, market_returns)[0, 1]
            market_variance = np.var(market_returns)
            beta = covariance / market_variance if market_variance > 0 else 1.0
        else:
            beta = 1.0

        # 夏普比率（假设无风险利率3%）
        risk_free_rate = 0.03
        mean_return = np.mean(portfolio_returns) * 252
        sharpe_ratio = (mean_return - risk_free_rate) / volatility if volatility > 0 else 0

        # 最大回撤
        cumulative = (1 + portfolio_returns).cumprod()
        running_max = np.maximum.accumulate(cumulative)
        drawdown = (cumulative - running_max) / running_max
        max_drawdown = np.min(drawdown)

        # VaR (95%)
        var_95 = np.percentile(portfolio_returns, 5)

        risk_metrics = {
            "volatility": float(volatility),
            "beta": float(beta),
            "sharpe_ratio": float(sharpe_ratio),
            "max_drawdown": float(max_drawdown),
            "var_95": float(var_95),
            "last_calculated": datetime.utcnow()
        }

        # 更新到数据库
        self.portfolios.update_one(
            {"portfolio_id": portfolio_id},
            {"$set": {"risk_metrics": risk_metrics}}
        )

        return risk_metrics

    def get_portfolio_analysis_history(self, portfolio_id: str,
                                       limit: int = 10) -> List[Dict]:
        """获取组合的历史分析记录"""
        portfolio = self.portfolios.find_one({"portfolio_id": portfolio_id})

        if not portfolio:
            return []

        # 获取所有持仓的历史分析
        tickers = [h['ticker'] for h in portfolio['holdings']]

        analyses = self.analyses.find({
            "ticker": {"$in": tickers}
        }).sort("analysis_date", -1).limit(limit * len(tickers))

        return list(analyses)

    def check_rebalancing_needed(self, portfolio_id: str,
                                 threshold: float = 0.05) -> Dict:
        """
        检查是否需要再平衡

        Args:
            threshold: 权重偏差阈值（默认5%）
        """
        portfolio = self.portfolios.find_one({"portfolio_id": portfolio_id})
        holdings = portfolio['holdings']

        rebalancing_needed = False
        suggestions = []

        for holding in holdings:
            current_weight = holding.get('weight', 0)
            target_weight = holding.get('target_weight', current_weight)

            deviation = abs(current_weight - target_weight)

            if deviation > threshold:
                rebalancing_needed = True
                action = "reduce" if current_weight > target_weight else "increase"
                suggestions.append({
                    "ticker": holding['ticker'],
                    "action": action,
                    "from_weight": current_weight,
                    "to_weight": target_weight,
                    "deviation": deviation
                })

        return {
            "rebalancing_needed": rebalancing_needed,
            "suggestions": suggestions
        }
```

---

## 5. 基础风险分析

### 5.1 Collection: risk_metrics

```python
{
    "_id": ObjectId("..."),
    "metric_id": "risk_metric_20250121_001",
    "ticker": "AAPL",  # 或 portfolio_id
    "metric_type": "single_stock",  # single_stock, portfolio
    "date": ISODate("2025-01-21"),

    # 风险指标
    "metrics": {
        # 市场风险
        "volatility": 0.28,  # 年化波动率
        "beta": 1.2,
        "var_95": -0.045,  # 95% VaR
        "cvar_95": -0.06,  # 95% CVaR (条件风险价值)

        # 回撤风险
        "max_drawdown": -0.25,
        "current_drawdown": -0.05,
        "drawdown_duration_days": 15,

        # 收益风险比
        "sharpe_ratio": 1.5,
        "sortino_ratio": 2.1,
        "calmar_ratio": 0.8,

        # 下行风险
        "downside_deviation": 0.15,
        "skewness": -0.3,  # 负偏度表示左尾风险
        "kurtosis": 4.2,   # 峰度>3表示厚尾

        # 流动性风险
        "avg_daily_volume": 50000000,
        "bid_ask_spread": 0.01,
        "liquidity_score": 0.9
    },

    # 风险评级
    "risk_rating": {
        "overall": "MEDIUM",  # LOW, MEDIUM, HIGH
        "market_risk": "MEDIUM",
        "liquidity_risk": "LOW",
        "volatility_risk": "MEDIUM",
        "concentration_risk": "LOW"
    },

    # 风险因素
    "risk_factors": [
        {
            "factor": "市场波动",
            "description": "年化波动率28%，高于市场平均",
            "severity": "medium",
            "impact_score": 0.6
        },
        {
            "factor": "估值风险",
            "description": "P/E比率28.5，接近历史高位",
            "severity": "medium",
            "impact_score": 0.5
        }
    ],

    "created_at": ISODate("2025-01-21T10:00:00Z")
}
```

---

### 5.2 RiskAnalyzer实现

```python
# risk_analyzer.py

import numpy as np
import pandas as pd
from scipy import stats
from typing import Dict, List

class RiskAnalyzer:
    """风险分析器"""

    def __init__(self, mongo_uri: str, db_name: str = "investment_platform"):
        self.client = MongoClient(mongo_uri)
        self.db = self.client[db_name]
        self.risk_metrics = self.db.risk_metrics

    def analyze_single_stock_risk(self, ticker: str,
                                   returns: np.ndarray,
                                   market_returns: np.ndarray = None) -> Dict:
        """
        分析单个股票的风险

        Args:
            ticker: 股票代码
            returns: 历史收益率数组
            market_returns: 市场收益率数组（计算Beta用）
        """
        # 1. 波动率
        volatility = np.std(returns) * np.sqrt(252)  # 年化

        # 2. Beta
        beta = 1.0
        if market_returns is not None:
            covariance = np.cov(returns, market_returns)[0, 1]
            market_variance = np.var(market_returns)
            beta = covariance / market_variance if market_variance > 0 else 1.0

        # 3. VaR和CVaR
        var_95 = np.percentile(returns, 5)
        cvar_95 = returns[returns <= var_95].mean()

        # 4. 最大回撤
        cumulative = (1 + returns).cumprod()
        running_max = np.maximum.accumulate(cumulative)
        drawdown = (cumulative - running_max) / running_max
        max_drawdown = np.min(drawdown)
        current_drawdown = drawdown[-1]

        # 5. Sharpe比率
        risk_free_rate = 0.03 / 252  # 日化
        excess_returns = returns - risk_free_rate
        sharpe_ratio = (np.mean(excess_returns) / np.std(returns) * np.sqrt(252)) if np.std(returns) > 0 else 0

        # 6. Sortino比率（只考虑下行波动）
        downside_returns = returns[returns < 0]
        downside_deviation = np.std(downside_returns) * np.sqrt(252) if len(downside_returns) > 0 else volatility
        sortino_ratio = (np.mean(excess_returns) * 252 / downside_deviation) if downside_deviation > 0 else 0

        # 7. 偏度和峰度
        skewness = stats.skew(returns)
        kurtosis = stats.kurtosis(returns)

        metrics = {
            "volatility": float(volatility),
            "beta": float(beta),
            "var_95": float(var_95),
            "cvar_95": float(cvar_95),
            "max_drawdown": float(max_drawdown),
            "current_drawdown": float(current_drawdown),
            "sharpe_ratio": float(sharpe_ratio),
            "sortino_ratio": float(sortino_ratio),
            "downside_deviation": float(downside_deviation),
            "skewness": float(skewness),
            "kurtosis": float(kurtosis)
        }

        # 8. 风险评级
        risk_rating = self._calculate_risk_rating(metrics)

        # 9. 识别风险因素
        risk_factors = self._identify_risk_factors(ticker, metrics)

        # 10. 保存到数据库
        metric_doc = {
            "metric_id": f"risk_metric_{datetime.now().strftime('%Y%m%d_%H%M%S')}",
            "ticker": ticker,
            "metric_type": "single_stock",
            "date": datetime.utcnow(),
            "metrics": metrics,
            "risk_rating": risk_rating,
            "risk_factors": risk_factors,
            "created_at": datetime.utcnow()
        }

        self.risk_metrics.insert_one(metric_doc)

        return metric_doc

    def _calculate_risk_rating(self, metrics: Dict) -> Dict:
        """计算风险评级"""
        # 简单的评级逻辑（可以更复杂）
        volatility = metrics['volatility']
        max_drawdown = abs(metrics['max_drawdown'])

        # 波动率评级
        if volatility < 0.2:
            volatility_risk = "LOW"
        elif volatility < 0.35:
            volatility_risk = "MEDIUM"
        else:
            volatility_risk = "HIGH"

        # 回撤评级
        if max_drawdown < 0.15:
            drawdown_risk = "LOW"
        elif max_drawdown < 0.30:
            drawdown_risk = "MEDIUM"
        else:
            drawdown_risk = "HIGH"

        # 综合评级
        risk_scores = {
            "LOW": 1,
            "MEDIUM": 2,
            "HIGH": 3
        }

        avg_score = (risk_scores[volatility_risk] + risk_scores[drawdown_risk]) / 2

        if avg_score < 1.5:
            overall = "LOW"
        elif avg_score < 2.5:
            overall = "MEDIUM"
        else:
            overall = "HIGH"

        return {
            "overall": overall,
            "volatility_risk": volatility_risk,
            "drawdown_risk": drawdown_risk
        }

    def _identify_risk_factors(self, ticker: str, metrics: Dict) -> List[Dict]:
        """识别风险因素"""
        risk_factors = []

        # 高波动风险
        if metrics['volatility'] > 0.30:
            risk_factors.append({
                "factor": "高波动性",
                "description": f"年化波动率{metrics['volatility']:.1%}，高于市场平均",
                "severity": "medium" if metrics['volatility'] < 0.4 else "high",
                "impact_score": min(metrics['volatility'] / 0.5, 1.0)
            })

        # 大回撤风险
        if abs(metrics['max_drawdown']) > 0.20:
            risk_factors.append({
                "factor": "历史大幅回撤",
                "description": f"最大回撤{metrics['max_drawdown']:.1%}",
                "severity": "high",
                "impact_score": min(abs(metrics['max_drawdown']) / 0.5, 1.0)
            })

        # 左尾风险（负偏度）
        if metrics['skewness'] < -0.5:
            risk_factors.append({
                "factor": "左尾风险",
                "description": f"收益分布负偏{metrics['skewness']:.2f}，极端亏损风险较高",
                "severity": "medium",
                "impact_score": min(abs(metrics['skewness']) / 2.0, 1.0)
            })

        # 厚尾风险
        if metrics['kurtosis'] > 5:
            risk_factors.append({
                "factor": "厚尾分布",
                "description": f"峰度{metrics['kurtosis']:.2f}，极端事件概率高",
                "severity": "medium",
                "impact_score": min(metrics['kurtosis'] / 10.0, 1.0)
            })

        return risk_factors
```

---

## 6. 完整架构设计

### 6.1 系统架构图

```
┌─────────────────────────────────────────────────────────┐
│                   用户接口层 (Flask API)                 │
│  - 会话管理API                                           │
│  - 投资组合API                                           │
│  - 风险分析API                                           │
└─────────────────────────────────────────────────────────┘
                          ↓
┌─────────────────────────────────────────────────────────┐
│              业务逻辑层 (Investment Platform)            │
│  ┌────────────────┐  ┌──────────────────┐              │
│  │ MarketDataAgent│  │ NewsAnalysisAgent│              │
│  └────────────────┘  └──────────────────┘              │
│  ┌────────────────────────────────────────┐            │
│  │      InvestmentAdvisor                 │            │
│  └────────────────────────────────────────┘            │
│  ┌────────────────────────────────────────┐            │
│  │      ReportAgent                       │            │
│  └────────────────────────────────────────┘            │
└─────────────────────────────────────────────────────────┘
                          ↓
┌─────────────────────────────────────────────────────────┐
│                   管理器层                               │
│  ┌──────────────┐ ┌─────────────────┐ ┌─────────────┐ │
│  │SessionManager│ │PortfolioManager │ │RiskAnalyzer │ │
│  └──────────────┘ └─────────────────┘ └─────────────┘ │
└─────────────────────────────────────────────────────────┘
                          ↓
┌─────────────────────────────────────────────────────────┐
│              数据持久层 (MongoDB)                        │
│  ┌────────────────────────────────────────────────┐    │
│  │  Layer 1: 对话上下文记忆                       │    │
│  │  - conversations (会话)                        │    │
│  │  - messages (消息)                             │    │
│  │  - analyses (分析记录)                         │    │
│  └────────────────────────────────────────────────┘    │
│  ┌────────────────────────────────────────────────┐    │
│  │  Layer 2: 实体关系记忆                         │    │
│  │  - entities (公司、人物、概念)                │    │
│  │  - relationships (关系图谱)                   │    │
│  └────────────────────────────────────────────────┘    │
│  ┌────────────────────────────────────────────────┐    │
│  │  投资组合与风险                                │    │
│  │  - portfolios (投资组合)                      │    │
│  │  - risk_metrics (风险指标)                    │    │
│  └────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────┘
```

---

### 6.2 数据流示例

**场景：用户多轮对话分析AAPL**

```
T1: 用户: "分析AAPL的投资价值"
    ↓
    SessionManager.create_conversation() → conversation_id
    ↓
    MarketDataAgent.research()
    NewsAnalysisAgent.research()
    ↓
    分析结果保存到MongoDB:
    - messages集合（assistant消息）
    - analyses集合（完整分析）
    - entities集合（AAPL实体，如果不存在则创建）
    ↓
    返回分析报告给用户

T2: 用户: "对比AAPL和MSFT"
    ↓
    SessionManager.find_relevant_context(conversation_id, "AAPL")
    → 找到T1的AAPL分析
    ↓
    MarketDataAgent.research("MSFT", context=T1_AAPL_analysis)
    → prompt中包含"之前分析过AAPL，现在对比MSFT"
    ↓
    分析结果保存到MongoDB:
    - messages集合
    - analyses集合
    - relationships集合（创建AAPL-MSFT竞争关系）
    ↓
    返回对比分析报告

T3: 用户: "AAPL的风险有哪些"
    ↓
    SessionManager.find_relevant_context(conversation_id, "AAPL")
    → 找到T1和T2的上下文
    ↓
    RiskAnalyzer.analyze_single_stock_risk("AAPL")
    ↓
    风险分析结果保存:
    - risk_metrics集合
    - 更新analyses集合中的risk_assessment字段
    ↓
    返回风险分析报告
```

---

## 7. 实现Roadmap

### Phase 1: MongoDB基础设施（Week 1-2）

**任务**：
- [ ] 设计MongoDB schema（7个collections）
- [ ] 实现SessionManager基础功能
- [ ] 实现基础的conversation和message管理
- [ ] 测试基本的CRUD操作

**交付物**：
- MongoDB collections创建完成
- SessionManager类实现
- 单元测试通过

---

### Phase 2: 会话管理集成（Week 3-4）

**任务**：
- [ ] 修改Agent类支持conversation_id参数
- [ ] 实现历史上下文检索
- [ ] 将上下文添加到SummaryNode的prompt
- [ ] 实现会话历史查询API

**交付物**：
- Agent支持会话上下文
- 多轮对话测试通过
- API端点实现

---

### Phase 3: 实体关系记忆（Week 5-6）

**任务**：
- [ ] 实现Entity自动提取（LLM）
- [ ] 实现Relationship自动识别
- [ ] 构建实体图谱查询接口
- [ ] 实现实体关联展示

**交付物**：
- 实体自动提取功能
- 关系图谱可视化（可选）
- 知识图谱查询API

---

### Phase 4: 投资组合追踪（Week 7-8）

**任务**：
- [ ] 实现PortfolioManager
- [ ] 实现组合统计计算
- [ ] 实现自动分析调度
- [ ] 实现再平衡建议

**交付物**：
- 投资组合CRUD完成
- 自动分析功能
- 再平衡建议功能

---

### Phase 5: 风险分析（Week 9-10）

**任务**：
- [ ] 实现RiskAnalyzer
- [ ] 实现单标的风险指标计算
- [ ] 实现组合风险指标计算
- [ ] 实现风险预警

**交付物**：
- 风险分析功能完整
- 风险预警系统
- 风险可视化（可选）

---

### Phase 6: 集成测试与优化（Week 11-12）

**任务**：
- [ ] 端到端集成测试
- [ ] 性能优化（MongoDB索引）
- [ ] 添加缓存机制
- [ ] 文档完善

**交付物**：
- 完整的扩展系统上线
- 性能达标
- 用户文档

---

## 8. 关键技术点

### 8.1 MongoDB索引设计

```javascript
// 会话索引
db.conversations.createIndex({"user_id": 1, "updated_at": -1})
db.conversations.createIndex({"conversation_id": 1})

// 消息索引
db.messages.createIndex({"conversation_id": 1, "timestamp": 1})
db.messages.createIndex({"analysis_result.ticker": 1, "timestamp": -1})

// 分析索引
db.analyses.createIndex({"ticker": 1, "analysis_date": -1})
db.analyses.createIndex({"user_id": 1, "analysis_date": -1})

// 实体索引
db.entities.createIndex({"entity_type": 1, "ticker": 1})
db.entities.createIndex({"entity_type": 1, "last_analyzed": -1})

// 关系索引
db.relationships.createIndex({"source_entity_id": 1, "relationship_type": 1})

// 组合索引
db.portfolios.createIndex({"user_id": 1, "status": 1})

// 风险指标索引
db.risk_metrics.createIndex({"ticker": 1, "date": -1})
```

---

### 8.2 配置示例

```python
# config.py

# MongoDB配置
MONGODB_URI = os.getenv('MONGODB_URI', 'mongodb://localhost:27017/')
MONGODB_DB_NAME = os.getenv('MONGODB_DB_NAME', 'investment_platform')

# 会话设置
SESSION_TIMEOUT_HOURS = 24
MAX_CONTEXT_MESSAGES = 10  # 最多保留多少条历史消息作为上下文

# 投资组合设置
PORTFOLIO_AUTO_UPDATE_INTERVAL_HOURS = 24
PORTFOLIO_REBALANCE_THRESHOLD = 0.05  # 5%偏差触发再平衡

# 风险分析设置
RISK_CALCULATION_LOOKBACK_DAYS = 252  # 1年交易日
RISK_VAR_CONFIDENCE = 0.95
```

---

## 9. 总结

### 扩展后的核心优势

1. **会话连续性** ✅
   - 支持多轮对话
   - 自动引用历史分析
   - 用户偏好学习

2. **投资组合管理** ✅
   - 持仓追踪
   - 自动分析
   - 再平衡建议

3. **风险量化** ✅
   - 多维度风险指标
   - 实时风险监控
   - 风险预警

4. **知识积累** ✅
   - 双重记忆体系
   - 实体关系图谱
   - 分析历史追溯

5. **智能化提升** ✅
   - 基于历史的智能建议
   - 个性化分析
   - 持续学习

---

**下一步行动**：从Phase 1开始，逐步实现MongoDB基础设施和会话管理机制。
