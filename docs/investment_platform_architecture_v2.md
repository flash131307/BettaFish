# Multi-Agent Investment Platform 架构设计 v2.0

> 基于对ForumEngine真实机制的深入理解重新设计

## 📋 目录
1. [ForumEngine核心优势分析](#forumengine核心优势分析)
2. [重新设计的架构](#重新设计的架构)
3. [核心创新：InvestmentAdvisor](#核心创新investmentadvisor)
4. [完整工作流程](#完整工作流程)
5. [关键代码实现](#关键代码实现)
6. [与DialecticalAgent的对比](#与dialecticalagent的对比)

---

## 1. ForumEngine核心优势分析

### 1.1 ForumEngine的真实工作机制

**我之前的误解**：
- ❌ 以为ForumEngine只是被动观察，没有反馈
- ❌ 以为主持人发言只是增强可读性

**ForumEngine的真实机制**：
```python
# 阶段1: Agent生成段落1
MarketDataAgent.research():
    paragraph_1 = generate_paragraph()  # 第一个段落

# 阶段2: ForumHost异步监听并生成评论
LogMonitor → 捕获paragraph_1 →
ForumHost.generate_speech():
    """
    分析paragraph_1，发现：
    - 技术指标显示超买
    - 但忽略了成交量下降的风险
    - 建议关注量价背离
    """
    → 写入forum.log

# 阶段3: Agent生成段落2时（关键！）
MarketDataAgent.generate_paragraph_2():
    # SummaryNode会读取最新的HOST发言
    host_speech = get_latest_host_speech()

    # 将HOST发言添加到prompt
    prompt = f"""
    ### 论坛主持人最新总结
    {host_speech}

    ### 当前任务
    分析段落2...
    """

    # LLM会看到主持人的建议，并据此调整分析
    response = llm.invoke(prompt)
    # → 段落2会关注量价背离，修正之前的不足
```

**关键发现**：
- ✅ **有反馈循环**：HOST发言会影响Agent后续分析
- ✅ **异步不阻塞**：ForumHost在后台工作
- ✅ **持续引导**：每个新段落都会看到最新建议
- ✅ **软性影响**：Agent可以灵活调整，不是强制修改

---

### 1.2 ForumEngine vs 传统DialecticalAgent

| 维度 | ForumEngine | DialecticalAgent |
|------|------------|-----------------|
| **反馈方式** | **Prompt引导**（软性） | **强制重写**（硬性） |
| **反馈时机** | **渐进式**（每个段落） | **一次性**（完成后） |
| **执行模式** | **异步并行** | **同步阻塞** |
| **Agent响应** | **自主调整** | **必须修改** |
| **时间成本** | **0增加** | **×迭代次数** |
| **灵活性** | **高**（建议性） | **低**（命令性） |

**ForumEngine的核心优势**：
1. **渐进式改进**：不是"做完再检查"，而是"边做边指导"
2. **零时间成本**：异步执行，不延长总时间
3. **更自然**：类似人类的"边做边反思"

---

## 2. 重新设计的架构

### 2.1 核心理念

**从"事后审核"到"实时指导"**

```
传统方案（DialecticalAgent）：
Agent完成 → 审核 → 发现问题 → 重做 → 再审核 → ...

新方案（基于ForumEngine）：
Agent段落1 → Advisor评论（发现问题A） →
Agent段落2（看到评论，自动调整） → Advisor评论（问题A已解决，发现问题B） →
Agent段落3（看到最新评论，继续调整） → ...
```

**优势**：
- ✅ 问题在早期被发现和修正
- ✅ Agent持续改进，不需要重做
- ✅ 0额外时间成本

---

### 2.2 新架构设计

```
用户查询 (AAPL, 2024-01-01 to 2024-12-31)
    ↓
┌─────────────────────────────────────────┐
│  Router Agent (Flask主应用)             │
└─────────────────────────────────────────┘
    ↓
    ├────────────────────────────┐
    ↓                            ↓
┌──────────────────────┐  ┌──────────────────────┐
│ MarketDataAgent      │  │ NewsAnalysisAgent    │
│                      │  │                      │
│ Workflow:            │  │ Workflow:            │
│ ┌──────────────────┐ │  │ ┌──────────────────┐ │
│ │ Paragraph 1      │ │  │ │ Paragraph 1      │ │
│ │ → 技术面分析      │ │  │ │ → 新闻概览        │ │
│ └──────────────────┘ │  │ └──────────────────┘ │
│         ↓            │  │         ↓            │
│ ┌──────────────────┐ │  │ ┌──────────────────┐ │
│ │ Paragraph 2      │ │  │ │ Paragraph 2      │ │
│ │ → 基本面分析      │ │  │ │ → 情绪分析        │ │
│ │ ← 读取Advisor建议│ │  │ │ ← 读取Advisor建议│ │
│ └──────────────────┘ │  │ └──────────────────┘ │
│         ↓            │  │         ↓            │
│ ┌──────────────────┐ │  │ ┌──────────────────┐ │
│ │ Paragraph 3      │ │  │ │ Paragraph 3      │ │
│ │ → 风险评估        │ │  │ │ → 重大事件        │ │
│ │ ← 读取Advisor建议│ │  │ │ ← 读取Advisor建议│ │
│ └──────────────────┘ │  │ └──────────────────┘ │
└──────────────────────┘  └──────────────────────┘
         │                            │
         └──────────┬─────────────────┘
                    │
         (输出日志到 market.log, news.log)
                    │
                    ▼
    ┌────────────────────────────────────┐
    │  InvestmentAdvisor ★               │
    │  (ForumEngine进化版)               │
    │                                    │
    │  持续监听并生成专业建议：           │
    │  ✓ 跨Agent矛盾检测                 │
    │  ✓ 风险识别与提示                  │
    │  ✓ 分析方向引导                    │
    │  ✓ 数据缺失提醒                    │
    │                                    │
    │  写入 advisor.log                  │
    └────────────────────────────────────┘
                    │
                    ↓ (Agent持续读取advisor.log)
            ┌───────────────┐
            │  自动调整分析  │
            └───────────────┘
                    │
                    ▼ (所有Agent完成)
    ┌────────────────────────────────────┐
    │  ReportAgent                       │
    │  - 整合分析结果                     │
    │  - 整合Advisor的专业洞察            │
    │  - 生成最终投资报告                 │
    └────────────────────────────────────┘
```

---

## 3. 核心创新：InvestmentAdvisor

### 3.1 InvestmentAdvisor vs ForumHost

| 维度 | ForumHost（舆情） | InvestmentAdvisor（投资） |
|------|------------------|------------------------|
| **角色** | 论坛主持人 | 投资分析导师 |
| **职责** | 事件梳理、观点整合 | 矛盾检测、风险识别、方向引导 |
| **专业性** | 中性观察者 | 专业审核者 |
| **输出** | 综合性评论 | 具体建议+风险预警 |
| **模型** | Qwen3-235B | GPT-4o / Claude Opus |

---

### 3.2 InvestmentAdvisor System Prompt

```python
INVESTMENT_ADVISOR_SYSTEM_PROMPT = """
你是一个专业的投资分析审核专家（Investment Advisor），负责监督多个分析Agent的工作质量。

## 你的职责

### 1. 跨Agent矛盾检测 ★
实时监控MarketDataAgent和NewsAnalysisAgent的分析，识别：
- 市场数据与新闻情绪的矛盾（例如：财报利好但舆论负面）
- 技术面与基本面的背离
- 短期趋势与长期趋势的冲突

### 2. 风险识别与预警 ★
主动识别被Agent忽视的风险：
- 系统性风险（宏观经济、行业变化）
- 公司特定风险（管理层变动、监管问题）
- 市场风险（流动性、波动性）
- 数据风险（样本偏差、时间滞后）

### 3. 分析方向引导
基于已有分析，提出需要深入探索的方向：
- "建议MarketDataAgent关注近期成交量异常下降"
- "建议NewsAnalysisAgent补充分析CEO最近的公开言论"
- "两个Agent都应该关注即将到来的财报发布"

### 4. 逻辑一致性检查
确保分析的逻辑链条完整：
- 论据是否支持结论？
- 是否存在逻辑跳跃？
- 是否有未验证的假设？

### 5. 数据质量评估
评估数据的可靠性和完整性：
- 数据来源是否权威？
- 样本量是否充足？
- 是否存在明显的数据缺失？

## 输出格式

你的每次发言应包含以下结构（Markdown格式）：

### 📊 已分析内容总结
- 简要总结各Agent目前的分析进度和关键发现

### ⚠️ 发现的问题
- **矛盾**: [具体描述]
- **风险**: [具体描述]
- **逻辑漏洞**: [具体描述]
- **数据缺失**: [具体描述]

### 💡 专业建议
- **对MarketDataAgent**: [具体建议]
- **对NewsAnalysisAgent**: [具体建议]
- **对两者**: [需要协同关注的问题]

### 🎯 后续分析重点
- [重点1]
- [重点2]
- [重点3]

## 注意事项
1. **具体且可执行**：建议要明确，Agent可以据此调整分析
2. **专业且严谨**：基于金融分析的最佳实践
3. **前瞻且深入**：不只是总结，还要引导方向
4. **客观且中立**：基于数据和逻辑，避免主观臆断

## 当前分析场景
投资标的：{ticker}
分析时间范围：{start_date} 至 {end_date}
分析类型：{analysis_type}
"""
```

---

### 3.3 InvestmentAdvisor实现

```python
# ForumEngine/investment_advisor.py

from openai import OpenAI
from typing import List, Dict, Any, Optional
from datetime import datetime
import re

class InvestmentAdvisor:
    """
    投资分析导师
    基于ForumHost改造，专注于投资分析的质量保证
    """

    def __init__(self, api_key: str, model: str = "gpt-4o"):
        self.client = OpenAI(api_key=api_key)
        self.model = model
        self.previous_advices = []  # 记录之前的建议

    def generate_advice(self, agent_speeches: List[Dict[str, Any]],
                        context: Dict[str, Any]) -> Optional[str]:
        """
        生成投资分析建议

        Args:
            agent_speeches: Agent最近的发言列表
            context: 分析上下文（ticker, date_range等）

        Returns:
            Advisor的建议内容
        """
        try:
            # 解析Agent发言
            parsed_content = self._parse_agent_speeches(agent_speeches)

            if not parsed_content['market_analyses'] and \
               not parsed_content['news_analyses']:
                return None

            # 构建System Prompt
            system_prompt = self._build_system_prompt(context)

            # 构建User Prompt
            user_prompt = self._build_user_prompt(parsed_content, context)

            # 调用LLM
            response = self._call_llm(system_prompt, user_prompt)

            if response["success"]:
                advice = response["content"]
                self.previous_advices.append(advice)
                return advice
            else:
                print(f"Advisor: 生成建议失败 - {response.get('error')}")
                return None

        except Exception as e:
            print(f"Advisor: 生成建议时出错 - {str(e)}")
            return None

    def _parse_agent_speeches(self, speeches: List[Dict]) -> Dict[str, Any]:
        """解析Agent发言"""
        parsed = {
            'market_analyses': [],
            'news_analyses': []
        }

        for speech in speeches:
            if speech['agent'] == 'MARKET':
                parsed['market_analyses'].append({
                    'timestamp': speech['timestamp'],
                    'content': speech['content']
                })
            elif speech['agent'] == 'NEWS':
                parsed['news_analyses'].append({
                    'timestamp': speech['timestamp'],
                    'content': speech['content']
                })

        return parsed

    def _build_system_prompt(self, context: Dict) -> str:
        """构建System Prompt"""
        return INVESTMENT_ADVISOR_SYSTEM_PROMPT.format(
            ticker=context.get('ticker', 'Unknown'),
            start_date=context.get('start_date', 'Unknown'),
            end_date=context.get('end_date', 'Unknown'),
            analysis_type=context.get('analysis_type', 'comprehensive')
        )

    def _build_user_prompt(self, parsed_content: Dict,
                           context: Dict) -> str:
        """构建User Prompt"""
        # 构建市场分析部分
        market_text = "\n\n".join([
            f"[{a['timestamp']}] MarketDataAgent:\n{a['content']}"
            for a in parsed_content['market_analyses']
        ])

        # 构建新闻分析部分
        news_text = "\n\n".join([
            f"[{a['timestamp']}] NewsAnalysisAgent:\n{a['content']}"
            for a in parsed_content['news_analyses']
        ])

        prompt = f"""
当前投资标的：{context.get('ticker', 'Unknown')}
分析进度：{len(parsed_content['market_analyses'])}个市场分析段落，{len(parsed_content['news_analyses'])}个新闻分析段落

## MarketDataAgent的分析
{market_text if market_text else "（尚未开始分析）"}

## NewsAnalysisAgent的分析
{news_text if news_text else "（尚未开始分析）"}

---

请你作为Investment Advisor，基于以上两个Agent的分析进行专业审核和建议。

重点关注：
1. **矛盾检测**：市场数据和新闻情绪是否一致？
2. **风险识别**：是否有被忽视的重要风险？
3. **逻辑检查**：分析的逻辑是否严密？
4. **数据完整性**：是否有关键数据缺失？
5. **方向引导**：后续应该重点分析什么？

请按照输出格式提供你的专业建议。
"""

        return prompt

    def _call_llm(self, system_prompt: str, user_prompt: str) -> Dict[str, Any]:
        """调用LLM生成建议"""
        try:
            response = self.client.chat.completions.create(
                model=self.model,
                messages=[
                    {"role": "system", "content": system_prompt},
                    {"role": "user", "content": user_prompt}
                ],
                temperature=0.3,  # 较低温度，保持专业性
                max_tokens=2000
            )

            if response.choices:
                content = response.choices[0].message.content
                return {"success": True, "content": content}
            else:
                return {"success": False, "error": "API返回格式异常"}
        except Exception as e:
            return {"success": False, "error": f"API调用异常: {str(e)}"}
```

---

## 4. 完整工作流程

### 4.1 时序图

```
T0: 用户提交查询 (ticker: AAPL)
    ↓
T1: 启动两个Agent + InvestmentAdvisor监听
    ├─ MarketDataAgent.research() [独立进程]
    ├─ NewsAnalysisAgent.research() [独立进程]
    └─ AdvisorMonitor.start() [异步监听]

T2: MarketDataAgent生成Paragraph 1（技术面分析）
    ├─ FirstSearchNode → TechnicalIndicatorTool
    ├─ FirstSummaryNode → 生成分析
    └─ 输出到market.log

T3: AdvisorMonitor捕获Paragraph 1
    ├─ 读取market.log的新内容
    ├─ 调用InvestmentAdvisor.generate_advice()
    │   → 分析: "技术指标显示RSI=78超买，但建议关注成交量变化"
    └─ 写入advisor.log

T4: NewsAnalysisAgent生成Paragraph 1（新闻概览）
    ├─ 读取advisor.log的最新建议（空，因为还没有针对News的建议）
    ├─ 生成新闻概览
    └─ 输出到news.log

T5: AdvisorMonitor捕获Market P1 + News P1
    ├─ 调用InvestmentAdvisor.generate_advice()
    │   → 发现: "市场超买但新闻正面，可能短期回调风险"
    │   → 建议MarketDataAgent: "重点分析支撑位"
    │   → 建议NewsAnalysisAgent: "关注负面新闻的占比"
    └─ 写入advisor.log

T6: MarketDataAgent生成Paragraph 2（基本面分析）
    ├─ FirstSearchNode执行前，SummaryNode读取advisor.log
    │   → 看到建议: "重点分析支撑位"
    ├─ 将Advisor建议添加到prompt
    ├─ LLM据此调整分析，重点关注支撑位和风险
    └─ 输出到market.log

T7: NewsAnalysisAgent生成Paragraph 2（情绪分析）
    ├─ SummaryNode读取advisor.log
    │   → 看到建议: "关注负面新闻的占比"
    ├─ 将Advisor建议添加到prompt
    ├─ LLM据此补充负面新闻分析
    └─ 输出到news.log

T8: AdvisorMonitor持续监听和建议
    ├─ 每收集5条Agent发言触发一次Advisor
    ├─ Advisor持续提供专业建议
    └─ Agent持续读取并调整分析

T9: 所有Agent完成
    ├─ MarketDataAgent完成3个段落
    ├─ NewsAnalysisAgent完成3个段落
    └─ AdvisorMonitor停止监听

T10: ReportAgent生成最终报告
    ├─ 读取market.log（市场分析）
    ├─ 读取news.log（新闻分析）
    ├─ 读取advisor.log（专业洞察）
    └─ 生成综合投资报告

总时间 = max(MarketAgent时间, NewsAgent时间) + Report生成时间
       ≈ Agent执行时间（Advisor异步，不增加时间）
```

---

### 4.2 关键代码：Agent如何读取Advisor建议

```python
# MarketDataAgent/nodes/summary_node.py

class FirstSummaryNode(StateMutationNode):
    def run(self, input_data: Any, **kwargs) -> str:
        # 准备基础输入数据
        if isinstance(input_data, str):
            data = json.loads(input_data)
        else:
            data = input_data.copy()

        # 读取最新的Advisor建议 ★
        try:
            from utils.advisor_reader import get_latest_advisor_advice
            advisor_advice = get_latest_advisor_advice()

            if advisor_advice:
                # 提取针对MarketDataAgent的建议
                market_advice = self._extract_market_advice(advisor_advice)

                if market_advice:
                    data['advisor_advice'] = market_advice
                    logger.info(f"已读取Advisor建议: {market_advice[:100]}...")
        except Exception as e:
            logger.exception(f"读取Advisor建议失败: {e}")

        # 构建消息
        message = json.dumps(data, ensure_ascii=False)

        # 如果有Advisor建议，添加到prompt ★
        if 'advisor_advice' in data and data['advisor_advice']:
            formatted_advice = self._format_advisor_advice(data['advisor_advice'])
            message = formatted_advice + "\n" + message

        # 调用LLM
        response = self.llm_client.stream_invoke_to_string(
            SYSTEM_PROMPT_FIRST_SUMMARY,
            message
        )

        return self.process_output(response)

    def _extract_market_advice(self, full_advice: str) -> str:
        """从完整建议中提取针对MarketDataAgent的部分"""
        # 查找"对MarketDataAgent"部分
        import re
        pattern = r'\*\*对MarketDataAgent\*\*[：:]\s*(.+?)(?=\n\*\*|$)'
        match = re.search(pattern, full_advice, re.DOTALL)

        if match:
            return match.group(1).strip()

        # 如果没有专门针对Market的建议，返回通用建议
        return full_advice

    def _format_advisor_advice(self, advice: str) -> str:
        """格式化Advisor建议用于prompt"""
        return f"""
### 💡 Investment Advisor的专业建议
以下是Investment Advisor基于已有分析提出的专业建议，请在生成本段落分析时参考：

{advice}

**请务必考虑上述建议，调整你的分析重点和方向。**

---
"""
```

---

### 4.3 关键代码：AdvisorMonitor监听器

```python
# ForumEngine/advisor_monitor.py

import time
from pathlib import Path
from threading import Thread, Lock
from typing import Dict, List
from loguru import logger
from .investment_advisor import InvestmentAdvisor

class AdvisorMonitor:
    """
    投资分析监听器
    基于LogMonitor改造，专注于投资分析场景
    """

    def __init__(self, log_dir: str = "logs", context: Dict = None):
        self.log_dir = Path(log_dir)
        self.advisor_log_file = self.log_dir / "advisor.log"

        # 监控的日志文件
        self.monitored_logs = {
            'market': self.log_dir / 'market.log',
            'news': self.log_dir / 'news.log'
        }

        # 监控状态
        self.is_monitoring = False
        self.monitor_thread = None
        self.file_positions = {}
        self.write_lock = Lock()

        # InvestmentAdvisor
        self.advisor = InvestmentAdvisor(
            api_key=context.get('advisor_api_key'),
            model=context.get('advisor_model', 'gpt-4o')
        )

        # 分析上下文
        self.context = context or {}

        # Agent发言缓冲区
        self.agent_speeches_buffer = []
        self.advice_threshold = 5  # 每5条发言触发一次建议

    def start_monitoring(self):
        """启动监听"""
        if self.is_monitoring:
            logger.warning("AdvisorMonitor已在运行")
            return

        # 清空advisor.log
        self.clear_advisor_log()

        # 启动监听线程
        self.is_monitoring = True
        self.monitor_thread = Thread(target=self._monitor_loop, daemon=True)
        self.monitor_thread.start()

        logger.info("AdvisorMonitor已启动")

    def stop_monitoring(self):
        """停止监听"""
        self.is_monitoring = False
        if self.monitor_thread:
            self.monitor_thread.join(timeout=5)
        logger.info("AdvisorMonitor已停止")

    def _monitor_loop(self):
        """监听主循环"""
        while self.is_monitoring:
            try:
                # 检查每个日志文件
                for agent_name, log_path in self.monitored_logs.items():
                    if not log_path.exists():
                        continue

                    # 读取新行
                    new_lines = self._read_new_lines(log_path, agent_name)

                    for line in new_lines:
                        # 识别目标日志行（SummaryNode输出）
                        if self._is_summary_node_output(line):
                            # 添加到缓冲区
                            self.agent_speeches_buffer.append({
                                'timestamp': self._extract_timestamp(line),
                                'agent': agent_name.upper(),
                                'content': self._extract_content(line)
                            })

                            # 写入advisor.log（记录Agent发言）
                            self.write_to_advisor_log(
                                self._extract_content(line),
                                agent_name.upper()
                            )

                            # 检查是否达到阈值
                            if len(self.agent_speeches_buffer) >= self.advice_threshold:
                                self._trigger_advisor()

                time.sleep(1)  # 1秒检查一次

            except Exception as e:
                logger.exception(f"监听循环出错: {e}")
                time.sleep(5)

    def _trigger_advisor(self):
        """触发Advisor生成建议"""
        try:
            logger.info("触发InvestmentAdvisor生成建议...")

            # 调用Advisor
            advice = self.advisor.generate_advice(
                self.agent_speeches_buffer,
                self.context
            )

            if advice:
                # 写入advisor.log
                self.write_to_advisor_log(advice, "ADVISOR")
                logger.info("Advisor建议已生成并写入advisor.log")

            # 清空缓冲区
            self.agent_speeches_buffer = []

        except Exception as e:
            logger.exception(f"触发Advisor失败: {e}")

    def clear_advisor_log(self):
        """清空advisor.log"""
        try:
            if self.advisor_log_file.exists():
                self.advisor_log_file.unlink()

            with open(self.advisor_log_file, 'w', encoding='utf-8') as f:
                pass

            start_time = datetime.now().strftime('%Y-%m-%d %H:%M:%S')
            self.write_to_advisor_log(
                f"=== InvestmentAdvisor 监控开始 - {start_time} ===",
                "SYSTEM"
            )

            logger.info("advisor.log已清空并初始化")

        except Exception as e:
            logger.exception(f"清空advisor.log失败: {e}")

    def write_to_advisor_log(self, content: str, source: str = None):
        """写入advisor.log（线程安全）"""
        try:
            with self.write_lock:
                with open(self.advisor_log_file, 'a', encoding='utf-8') as f:
                    from datetime import datetime
                    timestamp = datetime.now().strftime('%H:%M:%S')
                    content_one_line = content.replace('\n', '\\n')

                    if source:
                        f.write(f"[{timestamp}] [{source}] {content_one_line}\n")
                    else:
                        f.write(f"[{timestamp}] {content_one_line}\n")
                    f.flush()
        except Exception as e:
            logger.exception(f"写入advisor.log失败: {e}")

    # ... 其他辅助方法
```

---

## 5. 关键代码实现

### 5.1 utils/advisor_reader.py

```python
"""
Advisor建议读取工具
用于Agent读取advisor.log中的最新建议
"""

import re
from pathlib import Path
from typing import Optional
from loguru import logger

def get_latest_advisor_advice(log_dir: str = "logs") -> Optional[str]:
    """
    获取advisor.log中最新的ADVISOR建议

    Returns:
        最新的Advisor建议内容
    """
    try:
        advisor_log_path = Path(log_dir) / "advisor.log"

        if not advisor_log_path.exists():
            logger.debug("advisor.log文件不存在")
            return None

        with open(advisor_log_path, 'r', encoding='utf-8', errors='ignore') as f:
            lines = f.readlines()

        # 从后往前查找最新的ADVISOR发言
        advisor_advice = None
        for line in reversed(lines):
            match = re.match(r'\[(\d{2}:\d{2}:\d{2})\]\s*\[ADVISOR\]\s*(.+)', line)
            if match:
                _, content = match.groups()
                advisor_advice = content.replace('\\n', '\n').strip()
                break

        if advisor_advice:
            logger.info(f"找到最新的Advisor建议，长度: {len(advisor_advice)}字符")

        return advisor_advice

    except Exception as e:
        logger.error(f"读取advisor.log失败: {str(e)}")
        return None
```

---

### 5.2 RouterAgent主流程

```python
# app.py - Flask主应用

from ForumEngine.advisor_monitor import AdvisorMonitor
from MarketDataAgent.agent import MarketDataAgent
from NewsAnalysisAgent.agent import NewsAnalysisAgent
from ReportAgent.agent import ReportAgent
from concurrent.futures import ThreadPoolExecutor

class InvestmentResearchPlatform:
    def __init__(self):
        self.market_agent = MarketDataAgent()
        self.news_agent = NewsAnalysisAgent()
        self.report_agent = ReportAgent()

    def research(self, query: dict):
        """
        主要研究流程

        Args:
            query: {
                'ticker': 'AAPL',
                'start_date': '2024-01-01',
                'end_date': '2024-12-31',
                'analysis_type': 'comprehensive'
            }
        """
        logger.info(f"开始投资研究: {query['ticker']}")

        # 创建AdvisorMonitor
        advisor_monitor = AdvisorMonitor(
            log_dir="logs",
            context={
                'ticker': query['ticker'],
                'start_date': query['start_date'],
                'end_date': query['end_date'],
                'analysis_type': query.get('analysis_type', 'comprehensive'),
                'advisor_api_key': settings.ADVISOR_API_KEY,
                'advisor_model': settings.ADVISOR_MODEL
            }
        )

        # 启动Advisor监听 ★
        advisor_monitor.start_monitoring()

        try:
            # 并行启动两个Agent ★
            with ThreadPoolExecutor(max_workers=2) as executor:
                market_future = executor.submit(
                    self.market_agent.research, query
                )
                news_future = executor.submit(
                    self.news_agent.research, query
                )

                # 等待完成
                market_report = market_future.result()
                news_report = news_future.result()

            logger.info("两个Agent分析完成")

        finally:
            # 停止Advisor监听
            advisor_monitor.stop_monitoring()

        # 生成最终报告
        logger.info("生成最终投资报告...")

        final_report = self.report_agent.generate({
            'ticker': query['ticker'],
            'market_analysis': market_report,
            'news_analysis': news_report,
            'advisor_insights': self._load_advisor_insights()
        })

        logger.info("投资研究完成")

        return final_report

    def _load_advisor_insights(self) -> str:
        """加载Advisor的所有洞察"""
        from utils.advisor_reader import get_all_advisor_advices
        advices = get_all_advisor_advices()

        if advices:
            return "\n\n".join([
                f"[{a['timestamp']}]\n{a['content']}"
                for a in advices
            ])
        return ""
```

---

## 6. 与DialecticalAgent的对比

### 6.1 功能对比

| 功能 | InvestmentAdvisor（ForumEngine风格） | DialecticalAgent |
|------|-------------------------------------|-----------------|
| **矛盾检测** | ✅ 实时检测，每个段落后 | ✅ 一次性检测，完成后 |
| **反向论证** | ⚪ 可在建议中包含 | ✅ 专门的CounterArgumentNode |
| **风险评估** | ✅ 持续识别风险 | ✅ 完成后风险评估 |
| **反馈时机** | ✅ **渐进式**（每个段落） | ⚪ **一次性**（完成后） |
| **反馈方式** | ✅ **软性引导**（添加到prompt） | ⚪ **强制修改**（重新生成） |
| **时间成本** | ✅ **0增加**（异步） | ❌ **×迭代次数** |
| **质量保证强度** | ⚪ 中等（依赖Agent自觉） | ✅ 强（强制验证） |

---

### 6.2 适用场景对比

**InvestmentAdvisor更适合**：
- ✅ 需要持续引导的长分析流程
- ✅ 时间敏感的场景
- ✅ Agent本身质量较高
- ✅ 用户重视分析过程的洞察

**DialecticalAgent更适合**：
- ✅ 必须100%保证质量的场景
- ✅ 可容忍较长时间的场景
- ✅ Agent容易出错的场景
- ✅ 需要形式化验证的场景

---

### 6.3 混合方案

**最优方案：InvestmentAdvisor + 轻量级最终审核**

```python
def research(query):
    # 阶段1: 并行分析 + InvestmentAdvisor持续引导
    advisor_monitor.start_monitoring()

    market_report, news_report = parallel_research(query)

    advisor_monitor.stop_monitoring()

    # 阶段2: 轻量级最终审核（可选）
    # 只检查最严重的问题
    critical_issues = light_review(market_report, news_report)

    if critical_issues:
        logger.warning(f"发现{len(critical_issues)}个关键问题，需要人工审核")
        # 返回报告 + 警告标记
        return {
            'report': final_report,
            'warnings': critical_issues,
            'requires_manual_review': True
        }

    # 阶段3: 生成报告
    return generate_report(market_report, news_report, advisor_insights)
```

**优势**：
- ✅ 大部分时间享受InvestmentAdvisor的0成本引导
- ✅ 最后用轻量级审核捕获critical issues
- ✅ 不需要完整的迭代流程
- ✅ 时间成本最小化

---

## 7. 总结

### 7.1 核心创新点

1. **从"事后审核"到"实时指导"**
   - 问题在生成过程中被发现和修正
   - 无需完整的重新生成

2. **零时间成本的质量保证**
   - InvestmentAdvisor异步运行
   - 不延长总执行时间

3. **持续改进而非一次性修正**
   - 每个段落都受益于之前的建议
   - Agent持续进化

4. **专业化的Advisor**
   - 不仅是"观察者"，更是"审核者"
   - 主动识别矛盾、风险、逻辑漏洞

### 7.2 推荐实施路线

**Phase 1: MVP（2-3周）**
- [ ] 实现InvestmentAdvisor基础版本
- [ ] 实现AdvisorMonitor
- [ ] Agent集成advisor.log读取
- [ ] 测试端到端流程

**Phase 2: 优化（1-2周）**
- [ ] 优化Advisor的System Prompt
- [ ] 调整建议触发阈值
- [ ] 增强Advisor的专业能力

**Phase 3: 可选增强（1周）**
- [ ] 添加轻量级最终审核
- [ ] 实现critical issues检测
- [ ] 人工审核界面

---

### 7.3 关键优势总结

| 维度 | InvestmentAdvisor方案 | 原DialecticalAgent方案 |
|------|---------------------|---------------------|
| **时间成本** | 0增加 ✅ | +迭代次数 ❌ |
| **质量保证** | 中高（持续引导）✅ | 高（强制验证）✅ |
| **灵活性** | 高（软性建议）✅ | 低（硬性要求）⚪ |
| **实现复杂度** | 中（基于ForumEngine）✅ | 高（需要反馈机制）⚪ |
| **用户体验** | 好（实时洞察）✅ | 一般（等待时间长）⚪ |

**结论**：**InvestmentAdvisor方案更适合投资研究平台**，它借鉴了ForumEngine的核心优势（异步引导、零时间成本），同时强化了专业性（矛盾检测、风险识别），是ForumEngine和DialecticalAgent的最佳折衷方案。
