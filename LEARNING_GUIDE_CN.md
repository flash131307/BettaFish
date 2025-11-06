# 🎓 BettaFish（微舆）项目学习指南

本指南将帮助你系统地学习这个多智能体舆情分析系统。

---

## 📚 目录

1. [项目概览](#项目概览)
2. [核心概念](#核心概念)
3. [系统架构](#系统架构)
4. [学习路线](#学习路线)
5. [实践练习](#实践练习)
6. [常见问题](#常见问题)

---

## 🎯 项目概览

### 什么是 BettaFish？

BettaFish（微舆）是一个基于**多智能体协作**的公共舆情分析系统，主要特点：

- **4 个专业 AI Agent**：各司其职，协同工作
- **全域数据覆盖**：微博、小红书、抖音、快手等 30+ 平台
- **AI 论坛机制**：Agent 之间通过"论坛"进行思维碰撞
- **多模态分析**：文本、图片、视频一网打尽
- **全自动化**：从数据采集到报告生成的端到端流程

### 系统能做什么？

用户只需提出一个问题，比如"分析武汉大学的舆情"，系统就会：

1. 自动搜索相关新闻、社交媒体内容
2. 分析视频、图片等多模态内容
3. 查询私有舆情数据库
4. 进行情感分析
5. 生成专业的 HTML 分析报告

---

## 🧠 核心概念

### 1. 多智能体（Multi-Agent）系统

**什么是 Agent？**

Agent = LLM（大脑）+ Tools（工具）+ Nodes（工作流）

```python
class Agent:
    def __init__(self):
        # LLM 大脑：负责理解和决策
        self.llm = LLMClient(...)

        # Tools 工具：Agent 的"手"
        self.tools = [SearchTool, DatabaseTool, ...]

        # Nodes 节点：定义工作流程
        self.workflow = [SearchNode, SummaryNode, ...]
```

**为什么要多个 Agent？**

- **专业化分工**：每个 Agent 专注一个领域
- **并行处理**：同时工作，提高效率
- **协作互补**：通过论坛机制共享信息

### 2. 四大核心 Agent

#### ① Query Agent（网页搜索专家）
- **位置**: `QueryEngine/`
- **职责**: 搜索国内外新闻、网页内容
- **工具**: Tavily API、Bocha API
- **特点**: 支持多轮反思，优化搜索策略

#### ② Media Agent（多模态专家）
- **位置**: `MediaEngine/`
- **职责**: 分析视频、图片、短视频内容
- **工具**: 抖音爬虫、快手爬虫、OCR
- **特点**: 强大的视觉理解能力

#### ③ Insight Agent（数据库专家）
- **位置**: `InsightEngine/`
- **职责**: 深度挖掘私有舆情数据库
- **工具**: MySQL 查询、情感分析模型
- **特点**: 支持自定义业务数据接入

#### ④ Report Agent（报告生成专家）
- **位置**: `ReportEngine/`
- **职责**: 整合所有信息，生成报告
- **工具**: 模板引擎、HTML 生成
- **特点**: 动态选择模板，多轮优化

### 3. ForumEngine（论坛协作机制）

这是系统的创新点！Agent 之间通过"论坛"交流：

```
Agent A: "我发现武汉大学最近在樱花季获得了大量正面评论"
Agent B: "我从视频分析中发现游客对交通管理有些不满"
HOST:    "综合来看，建议深入调查交通管理问题..."

Agent A 和 B 读取 HOST 的建议 → 调整搜索方向
```

**工作机制**：

1. 各 Agent 写日志（insight.log, media.log, query.log）
2. ForumEngine Monitor 实时监控日志
3. 提取 Agent 的阶段性总结 → 写入 forum.log
4. 每 5 条发言触发 LLM 主持人总结
5. Agent 读取 forum.log，调整策略

### 4. State（状态管理）

系统使用 State 对象追踪整个分析过程：

```python
State                    # 整个报告的状态
├── query               # 用户查询
├── report_title        # 报告标题
└── paragraphs[]        # 段落列表
    └── Paragraph       # 单个段落
        ├── title       # 段落标题
        └── research    # 研究进度
            ├── search_history[]  # 搜索记录
            └── latest_summary    # 最新总结
```

---

## 🏗️ 系统架构

### 整体架构图

```
                    用户提问
                       ↓
              ┌────────────────┐
              │  Flask 主应用   │
              │   (app.py)     │
              └────────┬───────┘
                       │
        ┌──────────────┼──────────────┐
        │              │              │
        ↓              ↓              ↓
  Query Agent    Media Agent   Insight Agent
  (网页搜索)     (多模态)      (数据库)
        │              │              │
        └──────────────┼──────────────┘
                       │
                  ForumEngine
                  (论坛协作)
                       │
                       ↓
                 Report Agent
                 (报告生成)
                       │
                       ↓
                   HTML 报告
```

### 一次完整的分析流程

| 步骤 | 阶段 | 主要操作 | 参与组件 |
|------|------|----------|---------|
| 1 | 用户提问 | Flask 接收查询 | Flask 主应用 |
| 2 | 并行启动 | 3个Agent同时开始 | Query/Media/Insight Agent |
| 3 | 初步分析 | 使用专属工具概览搜索 | 各Agent + Tools |
| 4 | 策略制定 | 制定分块研究策略 | 各Agent决策模块 |
| 5-N | **循环阶段** | **论坛协作+深度研究** | **ForumEngine + 所有Agent** |
| 5.1 | 深度研究 | 基于论坛引导专项搜索 | 各Agent + 反思机制 |
| 5.2 | 论坛协作 | ForumEngine监控并总结 | ForumEngine + LLM主持人 |
| 5.3 | 交流融合 | 根据讨论调整方向 | 各Agent + forum_reader |
| N+1 | 结果整合 | Report Agent收集结果 | Report Agent |
| N+2 | 报告生成 | 动态选择模板生成报告 | Report Agent + 模板引擎 |

### 项目目录结构

```
BettaFish/
├── QueryEngine/              # Query Agent
│   ├── agent.py             # Agent 主逻辑
│   ├── llms/                # LLM 接口
│   ├── nodes/               # 处理节点
│   ├── tools/               # 搜索工具
│   └── utils/               # 工具函数
│
├── MediaEngine/             # Media Agent
│   ├── agent.py
│   ├── nodes/
│   ├── tools/
│   └── utils/
│
├── InsightEngine/           # Insight Agent
│   ├── agent.py
│   ├── nodes/
│   ├── tools/
│   │   ├── search.py              # 数据库查询
│   │   ├── keyword_optimizer.py   # 关键词优化
│   │   └── sentiment_analyzer.py  # 情感分析
│   ├── state/state.py       # 状态管理
│   └── utils/
│
├── ReportEngine/            # Report Agent
│   ├── agent.py
│   ├── nodes/
│   ├── report_template/     # 报告模板库
│   └── flask_interface.py
│
├── ForumEngine/             # 论坛引擎
│   ├── monitor.py           # 日志监控
│   └── llm_host.py          # 论坛主持人
│
├── MindSpider/              # 爬虫系统
│   ├── main.py
│   ├── BroadTopicExtraction/      # 话题提取
│   └── DeepSentimentCrawling/     # 深度爬取
│
├── SentimentAnalysisModel/  # 情感分析模型集合
│   ├── WeiboMultilingualSentiment/  # 多语言（推荐）
│   ├── WeiboSentiment_SmallQwen/    # 小参数Qwen3
│   └── WeiboSentiment_MachineLearning/
│
├── utils/                   # 通用工具
│   ├── forum_reader.py      # 读取论坛内容
│   └── retry_helper.py      # 重试机制
│
├── app.py                   # Flask 主应用
├── config.py                # 全局配置
└── requirements.txt         # 依赖包
```

---

## 📖 学习路线

### 🎯 阶段一：快速上手（1-2天）

#### 目标
- 理解项目整体架构
- 成功运行系统
- 生成第一个分析报告

#### 学习内容

**1. 阅读文档**
```bash
# 阅读 README.md
cat README.md | head -200

# 查看示例报告
open final_reports/final_report__20250827_131630.html
```

**2. 环境配置**
```bash
# 创建虚拟环境
conda create -n bettafish python=3.11
conda activate bettafish

# 安装依赖
pip install -r requirements.txt

# 安装浏览器驱动
playwright install chromium
```

**3. 配置 API**
```bash
# 复制配置模板
cp config.py.example config.py

# 编辑配置文件，填入你的 API Keys
vim config.py
```

需要申请的 API：
- **Kimi API**（Insight Agent）: https://platform.moonshot.cn/
- **Gemini API**（Media/Report Agent）: https://www.chataiapi.com/
- **DeepSeek API**（Query Agent）: https://www.deepseek.com/
- **Qwen API**（Forum Host）: https://cloud.siliconflow.cn/
- **Tavily API**（网页搜索）: https://www.tavily.com/
- **Bocha API**（网页搜索）: https://open.bochaai.com/

**4. 首次运行**
```bash
# 启动完整系统
python app.py

# 访问 http://localhost:5000
# 输入查询，比如："分析华为的最新舆情"
```

#### 实践练习

**练习1：单独运行一个 Agent**
```bash
# 运行 Insight Agent
streamlit run SingleEngineApp/insight_engine_streamlit_app.py --server.port 8501

# 访问 http://localhost:8501
# 尝试查询："武汉大学"
```

**练习2：查看日志**
```bash
# 查看 ForumEngine 论坛日志
tail -f logs/forum.log

# 查看各 Agent 日志
tail -f logs/insight.log
tail -f logs/media.log
tail -f logs/query.log
```

---

### 🔬 阶段二：深入单个 Agent（3-5天）

#### 目标
- 理解 Agent 的工作原理
- 掌握 LLM + Tools + Nodes 架构
- 学会自定义 Agent 行为

#### 推荐学习顺序：Insight Agent → Query Agent → Media Agent → Report Agent

---

#### 深入 Insight Agent

**1. 核心文件阅读**

```bash
# Agent 主逻辑
InsightEngine/agent.py          # DeepSearchAgent 类

# 状态管理
InsightEngine/state/state.py    # State, Paragraph, Research, Search

# 处理节点
InsightEngine/nodes/search_node.py       # 搜索节点
InsightEngine/nodes/summary_node.py      # 总结节点
InsightEngine/nodes/formatting_node.py   # 格式化节点

# 工具集
InsightEngine/tools/search.py              # 数据库查询
InsightEngine/tools/keyword_optimizer.py   # 关键词优化
InsightEngine/tools/sentiment_analyzer.py  # 情感分析
```

**2. 工作流程图解**

```python
# InsightEngine/agent.py - run() 方法

def run(self, query: str):
    # 1. 初始化 State
    self.state.query = query

    # 2. 生成报告结构（调用 LLM）
    self.report_structure_node.execute(self.state)
    # → State.paragraphs = [段落1, 段落2, ...]

    # 3. 对每个段落循环
    for paragraph in self.state.paragraphs:

        # 4. 首次搜索（调用 Tools）
        self.first_search_node.execute(paragraph)
        # → paragraph.research.search_history += [搜索结果...]

        # 5. 首次总结（调用 LLM）
        self.first_summary_node.execute(paragraph)
        # → paragraph.research.latest_summary = "..."

        # 6. 反思循环（最多 max_reflections 次）
        for i in range(max_reflections):
            # 6.1 反思节点（调用 LLM）
            need_more = self.reflection_node.execute(paragraph)

            if not need_more:
                break  # 满意了，结束反思

            # 6.2 补充搜索（调用 Tools）
            self.first_search_node.execute(paragraph)

            # 6.3 反思总结（调用 LLM）
            self.reflection_summary_node.execute(paragraph)

    # 7. 格式化报告（调用 LLM）
    self.report_formatting_node.execute(self.state)
    # → State.final_report = "完整的报告内容"

    return self.state.final_report
```

**3. 关键数据结构**

```python
# State: 整个报告的状态
state = State(
    query="分析武汉大学的舆情",
    report_title="武汉大学品牌声誉分析报告",
    paragraphs=[
        Paragraph(
            title="引言",
            content="...",
            research=Research(
                search_history=[
                    Search(query="武汉大学", content="..."),
                    Search(query="武汉大学 樱花", content="..."),
                ],
                latest_summary="武汉大学是一所...",
                reflection_iteration=2,
                is_completed=True
            )
        ),
        # ... 更多段落
    ],
    final_report="# 武汉大学品牌声誉分析报告\n\n..."
)
```

**4. Tools 工具集详解**

```python
# InsightEngine/tools/search.py

class MediaCrawlerDB:
    """舆情数据库查询工具集"""

    # 工具1: 查找热点内容
    def search_hot_content(self, time_period="week", limit=100):
        """查询指定时间段内的热点内容"""

    # 工具2: 全局话题搜索
    def search_topic_globally(self, keywords, limit=200):
        """根据关键词全局搜索话题"""

    # 工具3: 按日期搜索
    def search_topic_by_date(self, keywords, start_date, end_date):
        """按时间范围搜索话题"""

    # 工具4: 获取话题评论
    def get_comments_for_topic(self, topic_id, limit=500):
        """获取指定话题的用户评论"""

    # 工具5: 平台定向搜索
    def search_topic_on_platform(self, keywords, platform):
        """在指定平台搜索话题"""
```

**5. Nodes 节点详解**

每个节点都继承自 `BaseNode`，并实现 `execute()` 方法：

```python
# InsightEngine/nodes/search_node.py

class FirstSearchNode(BaseNode):
    """首次搜索节点"""

    def execute(self, paragraph: Paragraph):
        # 1. 准备搜索参数
        query = paragraph.title

        # 2. 调用 LLM 决定使用哪个工具
        tool_choice = self.llm.decide_tool(query)

        # 3. 执行搜索
        results = self.agent.execute_search_tool(
            tool_name=tool_choice,
            query=query
        )

        # 4. 保存到 State
        paragraph.research.add_search_results(query, results)
```

**6. 实践练习**

**练习1：修改搜索限制**
```python
# 编辑 InsightEngine/utils/config.py
class Config:
    # 修改这些参数，观察效果
    default_search_topic_globally_limit = 300  # 改为300
    max_reflections = 3  # 增加到3轮反思
```

**练习2：添加自定义工具**
```python
# InsightEngine/tools/search.py

class MediaCrawlerDB:
    def custom_search_recent_trends(self, keywords, days=7):
        """自定义工具：搜索最近N天的趋势"""
        end_date = datetime.now().strftime('%Y-%m-%d')
        start_date = (datetime.now() - timedelta(days=days)).strftime('%Y-%m-%d')
        return self.search_topic_by_date(keywords, start_date, end_date)
```

**练习3：跟踪 State 变化**
```python
# 在 InsightEngine/agent.py 的 run() 方法中添加日志

def run(self, query: str):
    self.state.query = query
    print(f"[DEBUG] 初始 State: {self.state.to_json()}")

    self.report_structure_node.execute(self.state)
    print(f"[DEBUG] 生成结构后: {len(self.state.paragraphs)} 个段落")

    for idx, paragraph in enumerate(self.state.paragraphs):
        print(f"[DEBUG] 处理段落 {idx+1}: {paragraph.title}")
        self.first_search_node.execute(paragraph)
        print(f"[DEBUG] 搜索结果数: {paragraph.research.get_search_count()}")
```

---

#### 深入 Query Agent

Query Agent 负责网页搜索，架构与 Insight Agent 类似。

**核心差异**：

| 特性 | Insight Agent | Query Agent |
|------|--------------|-------------|
| 数据源 | 私有数据库 | 网页搜索 |
| 主要工具 | MySQL 查询 | Tavily/Bocha API |
| 特殊能力 | 情感分析 | 反思优化搜索词 |

**关键文件**：
```bash
QueryEngine/agent.py                 # WebSearchAgent
QueryEngine/tools/web_search.py      # 网页搜索工具
QueryEngine/nodes/reflection_node.py # 反思节点
```

---

#### 深入 Media Agent

Media Agent 负责多模态内容分析。

**特殊能力**：
- 视频内容理解（抖音、快手）
- 图片 OCR
- 结构化卡片提取

**关键文件**：
```bash
MediaEngine/agent.py
MediaEngine/tools/video_analyzer.py    # 视频分析
MediaEngine/tools/image_processor.py   # 图片处理
```

---

### 🔗 阶段三：理解协作机制（2-3天）

#### 目标
- 理解 ForumEngine 工作原理
- 掌握 Agent 间通信机制
- 学会调试多 Agent 系统

#### ForumEngine 深度解析

**1. 核心组件**

```bash
ForumEngine/monitor.py         # 日志监控器
ForumEngine/llm_host.py        # LLM 主持人
utils/forum_reader.py          # Agent 读取论坛
```

**2. 工作流程**

```python
# ForumEngine/monitor.py

class LogMonitor:
    """实时监控三个 Agent 的日志"""

    def __init__(self):
        # 监控的日志文件
        self.monitored_logs = {
            'insight': 'logs/insight.log',
            'media': 'logs/media.log',
            'query': 'logs/query.log'
        }

        # 输出文件
        self.forum_log = 'logs/forum.log'

        # 主持人相关
        self.agent_speeches_buffer = []
        self.host_speech_threshold = 5  # 每5条触发主持人

    def monitor_loop(self):
        """监控循环"""
        while self.is_monitoring:
            # 1. 检查每个日志文件
            for agent_name, log_file in self.monitored_logs.items():
                # 2. 读取新内容
                new_content = self.read_new_lines(log_file)

                # 3. 提取 SummaryNode 输出
                if 'SummaryNode' in new_content:
                    summary = self.extract_summary(new_content)

                    # 4. 写入 forum.log
                    self.write_to_forum_log(summary, agent_name.upper())

                    # 5. 添加到缓冲区
                    self.agent_speeches_buffer.append({
                        'agent': agent_name,
                        'content': summary
                    })

            # 6. 检查是否触发主持人
            if len(self.agent_speeches_buffer) >= self.host_speech_threshold:
                self.trigger_host_speech()
                self.agent_speeches_buffer.clear()

    def trigger_host_speech(self):
        """触发主持人发言"""
        # 1. 准备上下文
        context = "\n".join([
            f"{speech['agent']}: {speech['content']}"
            for speech in self.agent_speeches_buffer
        ])

        # 2. 调用 LLM 主持人
        from .llm_host import generate_host_speech
        host_response = generate_host_speech(context)

        # 3. 写入 forum.log
        self.write_to_forum_log(host_response, "HOST")
```

**3. Agent 读取论坛**

```python
# utils/forum_reader.py

def get_latest_host_speech(log_dir="logs"):
    """获取最新的主持人发言"""
    forum_log = Path(log_dir) / "forum.log"

    with open(forum_log, 'r') as f:
        lines = f.readlines()

    # 从后往前查找 HOST 发言
    for line in reversed(lines):
        if '[HOST]' in line:
            # 提取内容
            match = re.match(r'\[(\d{2}:\d{2}:\d{2})\]\s*\[HOST]\s*(.+)', line)
            if match:
                return match.group(2)

    return None
```

**4. Agent 如何使用论坛**

```python
# InsightEngine/nodes/reflection_node.py

class ReflectionNode(BaseNode):
    def execute(self, paragraph: Paragraph):
        # 1. 读取论坛最新讨论
        from utils.forum_reader import get_latest_host_speech
        host_speech = get_latest_host_speech()

        # 2. 结合论坛建议进行反思
        prompt = f"""
        当前段落: {paragraph.title}
        已有总结: {paragraph.research.latest_summary}

        论坛主持人建议: {host_speech}

        请评估：是否需要补充更多信息？
        """

        # 3. 调用 LLM
        response = self.llm.chat(prompt)

        return response['need_more_search']
```

**5. 实践练习**

**练习1：观察论坛交流**
```bash
# 终端1：启动系统
python app.py

# 终端2：实时监控论坛
tail -f logs/forum.log

# 终端3：监控 Insight Agent
tail -f logs/insight.log

# 观察：
# - Agent 何时发言？
# - 主持人何时总结？
# - Agent 如何调整策略？
```

**练习2：修改主持人触发阈值**
```python
# ForumEngine/monitor.py

class LogMonitor:
    def __init__(self):
        # 修改为每 3 条发言触发一次
        self.host_speech_threshold = 3  # 原来是5
```

**练习3：自定义主持人 Prompt**
```python
# ForumEngine/llm_host.py

def generate_host_speech(agent_speeches):
    prompt = f"""
    你是一个舆情分析论坛的主持人。

    以下是各Agent的最新发言：
    {agent_speeches}

    请：
    1. 总结各Agent的核心发现
    2. 指出可能的矛盾或盲点
    3. 提出下一步研究建议

    # 你可以在这里自定义主持人的行为！
    """

    response = llm.chat(prompt)
    return response
```

---

### 🎨 阶段四：定制与扩展（进阶）

#### 目标
- 自定义报告模板
- 接入自己的数据库
- 集成新的 LLM 模型
- 开发新的 Agent

#### 自定义报告模板

```bash
# 1. 查看现有模板
ls ReportEngine/report_template/

# 2. 创建新模板
vim ReportEngine/report_template/我的模板.md
```

模板格式示例：
```markdown
# {报告标题}

## 一、概述
{概述内容}

## 二、详细分析
{详细分析}

## 三、数据可视化
{图表占位符}

## 四、结论与建议
{结论建议}
```

#### 接入自定义数据库

```python
# 1. 修改 config.py
BUSINESS_DB_HOST = "your_host"
BUSINESS_DB_PORT = 3306
BUSINESS_DB_USER = "your_user"
BUSINESS_DB_PASSWORD = "your_password"
BUSINESS_DB_NAME = "your_database"

# 2. 创建自定义工具
# InsightEngine/tools/custom_db.py

class CustomBusinessDB:
    """自定义业务数据库工具"""

    def __init__(self):
        self.conn = mysql.connector.connect(
            host=config.BUSINESS_DB_HOST,
            user=config.BUSINESS_DB_USER,
            password=config.BUSINESS_DB_PASSWORD,
            database=config.BUSINESS_DB_NAME
        )

    def search_customer_feedback(self, product_id):
        """查询客户反馈"""
        cursor = self.conn.cursor(dictionary=True)
        cursor.execute("""
            SELECT * FROM feedback
            WHERE product_id = %s
            ORDER BY created_at DESC
        """, (product_id,))
        return cursor.fetchall()

# 3. 集成到 Agent
# InsightEngine/agent.py

class DeepSearchAgent:
    def __init__(self):
        # ... 其他初始化
        self.custom_db = CustomBusinessDB()

    def execute_custom_search(self, product_id):
        return self.custom_db.search_customer_feedback(product_id)
```

#### 更换 LLM 模型

系统支持任何 OpenAI 兼容的 API：

```python
# config.py

# 使用本地部署的模型
INSIGHT_ENGINE_API_KEY = "your_key"
INSIGHT_ENGINE_BASE_URL = "http://localhost:8000/v1"  # Ollama/vLLM
INSIGHT_ENGINE_MODEL_NAME = "qwen2.5-72b-instruct"

# 使用其他商业API
QUERY_ENGINE_BASE_URL = "https://api.anthropic.com/v1"  # Claude
QUERY_ENGINE_MODEL_NAME = "claude-3-sonnet-20240229"
```

#### 开发新的 Agent

假设你想开发一个"财务分析 Agent"：

```python
# 1. 创建目录结构
mkdir FinanceEngine
cd FinanceEngine
mkdir llms nodes tools utils

# 2. 创建 Agent 主类
# FinanceEngine/agent.py

from .llms import LLMClient
from .tools import StockDataTool, FinancialReportTool

class FinanceAgent:
    """财务分析Agent"""

    def __init__(self):
        self.llm = LLMClient(...)
        self.stock_tool = StockDataTool()
        self.report_tool = FinancialReportTool()

    def analyze_company(self, company_name):
        # 1. 获取股票数据
        stock_data = self.stock_tool.get_stock_data(company_name)

        # 2. 获取财报
        financial_report = self.report_tool.get_latest_report(company_name)

        # 3. LLM 分析
        analysis = self.llm.analyze(stock_data, financial_report)

        return analysis

# 3. 创建工具
# FinanceEngine/tools/stock_data.py

class StockDataTool:
    def get_stock_data(self, symbol):
        # 调用股票API
        import yfinance as yf
        stock = yf.Ticker(symbol)
        return stock.history(period="1y")

# 4. 集成到主应用
# app.py

from FinanceEngine import FinanceAgent

finance_agent = FinanceAgent()

@app.route('/api/finance/analyze')
def analyze_finance():
    company = request.args.get('company')
    result = finance_agent.analyze_company(company)
    return jsonify(result)
```

---

## 💡 实践练习

### 练习1：端到端运行

**目标**：完整运行系统，生成一份分析报告

```bash
# 1. 启动系统
python app.py

# 2. 访问 http://localhost:5000

# 3. 输入查询："分析华为Mate60的市场反馈"

# 4. 观察：
#    - 各 Agent 的运行状态
#    - ForumEngine 的协作过程
#    - 最终生成的报告质量

# 5. 查看日志
tail -f logs/forum.log
tail -f logs/insight.log
tail -f logs/media.log
tail -f logs/query.log
```

### 练习2：单 Agent 调试

**目标**：深入理解单个 Agent 的工作流程

```python
# 创建测试脚本：test_insight_agent.py

import sys
sys.path.append('.')

from InsightEngine import DeepSearchAgent, Config

# 1. 初始化 Agent
config = Config()
agent = DeepSearchAgent(config)

# 2. 执行查询
query = "武汉大学"
print(f"查询：{query}")

# 3. 逐步执行
state = agent.state
state.query = query

# 生成报告结构
agent.report_structure_node.execute(state)
print(f"生成了 {len(state.paragraphs)} 个段落")

# 处理第一个段落
first_para = state.paragraphs[0]
print(f"\n处理段落：{first_para.title}")

# 首次搜索
agent.first_search_node.execute(first_para)
print(f"搜索到 {first_para.research.get_search_count()} 条结果")

# 首次总结
agent.first_summary_node.execute(first_para)
print(f"总结：{first_para.research.latest_summary[:200]}...")

# 4. 保存 State
state.save_to_file("test_state.json")
print("\nState 已保存到 test_state.json")
```

运行：
```bash
python test_insight_agent.py
```

### 练习3：自定义搜索策略

**目标**：修改 Agent 的搜索行为

```python
# InsightEngine/nodes/search_node.py

class FirstSearchNode(BaseNode):
    def execute(self, paragraph: Paragraph):
        # 原始逻辑：调用 search_topic_globally

        # 自定义策略：先热点，后全局
        # 1. 先搜索热点
        hot_results = self.agent.execute_search_tool(
            tool_name="search_hot_content",
            time_period="week",
            limit=50
        )

        # 2. 过滤相关热点
        relevant_hot = self.filter_relevant(hot_results, paragraph.title)

        # 3. 如果热点不足，补充全局搜索
        if len(relevant_hot) < 20:
            global_results = self.agent.execute_search_tool(
                tool_name="search_topic_globally",
                query=paragraph.title,
                limit=100
            )
            relevant_hot.extend(global_results)

        # 4. 保存结果
        paragraph.research.add_search_results(
            paragraph.title,
            relevant_hot
        )

    def filter_relevant(self, results, keyword):
        """过滤相关结果"""
        return [r for r in results if keyword in r['title'] or keyword in r['content']]
```

### 练习4：实现简单的 Agent

**目标**：从零实现一个最小化的 Agent

```python
# minimal_agent.py - 最小化 Agent 示例

from openai import OpenAI

class MinimalAgent:
    """最小化Agent：LLM + Tool + 简单流程"""

    def __init__(self, api_key, base_url, model):
        self.llm = OpenAI(api_key=api_key, base_url=base_url)
        self.model = model

    def search_web(self, query):
        """工具：模拟网页搜索"""
        # 这里简化为返回假数据
        return f"关于'{query}'的搜索结果..."

    def analyze(self, query):
        """主流程"""
        # 1. 搜索
        search_results = self.search_web(query)

        # 2. 分析
        prompt = f"""
        用户查询：{query}
        搜索结果：{search_results}

        请生成一份分析报告。
        """

        response = self.llm.chat.completions.create(
            model=self.model,
            messages=[{"role": "user", "content": prompt}]
        )

        return response.choices[0].message.content

# 使用
agent = MinimalAgent(
    api_key="your_key",
    base_url="https://api.deepseek.com",
    model="deepseek-chat"
)

result = agent.analyze("人工智能的未来趋势")
print(result)
```

---

## ❓ 常见问题

### Q1: 如何调试 Agent 不工作的问题？

**A**: 分步骤排查：

```bash
# 1. 检查配置
cat config.py | grep API_KEY  # 确保API密钥正确

# 2. 测试 LLM 连接
python -c "
from openai import OpenAI
client = OpenAI(api_key='your_key', base_url='your_url')
response = client.chat.completions.create(
    model='your_model',
    messages=[{'role': 'user', 'content': 'Hello'}]
)
print(response.choices[0].message.content)
"

# 3. 检查数据库连接
python -c "
import mysql.connector
conn = mysql.connector.connect(
    host='your_host',
    user='your_user',
    password='your_password',
    database='your_db'
)
print('数据库连接成功！')
"

# 4. 查看日志
tail -100 logs/insight.log  # 查看最近100行日志
```

### Q2: 如何提高 Agent 的分析质量？

**A**: 多个优化方向：

1. **增加反思轮次**
```python
# InsightEngine/utils/config.py
max_reflections = 3  # 增加到3轮
```

2. **优化 Prompt**
```python
# InsightEngine/prompts/prompts.py
SUMMARY_PROMPT = """
你是一个专业的舆情分析师，拥有10年经验。

# 添加更详细的指导...
"""
```

3. **增加搜索结果数量**
```python
# InsightEngine/utils/config.py
default_search_topic_globally_limit = 500  # 增加到500
```

4. **使用更强的模型**
```python
# config.py
INSIGHT_ENGINE_MODEL_NAME = "gpt-4"  # 使用更强的模型
```

### Q3: 如何减少 API 调用成本？

**A**: 成本优化策略：

1. **缓存搜索结果**
```python
# 添加简单的缓存机制
import json
from pathlib import Path

class CachedSearchTool:
    def __init__(self, cache_dir=".cache"):
        self.cache_dir = Path(cache_dir)
        self.cache_dir.mkdir(exist_ok=True)

    def search(self, query):
        # 检查缓存
        cache_file = self.cache_dir / f"{hash(query)}.json"
        if cache_file.exists():
            with open(cache_file) as f:
                return json.load(f)

        # 实际搜索
        results = self.actual_search(query)

        # 保存缓存
        with open(cache_file, 'w') as f:
            json.dump(results, f)

        return results
```

2. **减少反思轮次**
```python
max_reflections = 1  # 只反思1轮
```

3. **使用更便宜的模型**
```python
# 对于简单任务使用便宜的模型
KEYWORD_OPTIMIZER_MODEL_NAME = "qwen-turbo"
```

### Q4: 如何并行运行多个查询？

**A**: 使用多线程或多进程：

```python
# parallel_analysis.py

import concurrent.futures
from InsightEngine import DeepSearchAgent

def analyze_query(query):
    agent = DeepSearchAgent()
    return agent.run(query)

queries = [
    "华为Mate60的市场反馈",
    "小米14的用户评价",
    "OPPO Find X7的舆情分析"
]

# 并行执行
with concurrent.futures.ThreadPoolExecutor(max_workers=3) as executor:
    results = list(executor.map(analyze_query, queries))

for query, result in zip(queries, results):
    print(f"{query}:\n{result}\n")
```

### Q5: 如何监控系统性能？

**A**: 添加性能监控：

```python
# utils/performance_monitor.py

import time
import functools

def timing_decorator(func):
    """装饰器：测量函数执行时间"""
    @functools.wraps(func)
    def wrapper(*args, **kwargs):
        start = time.time()
        result = func(*args, **kwargs)
        end = time.time()
        print(f"[PERF] {func.__name__} took {end-start:.2f}s")
        return result
    return wrapper

# 使用
class FirstSearchNode(BaseNode):
    @timing_decorator
    def execute(self, paragraph):
        # ... 原有逻辑
        pass
```

---

## 🎯 进阶主题

### 1. LangGraph 集成

本项目可以迁移到 LangGraph 框架：

```python
# langgraph_version.py

from langgraph.graph import StateGraph, END

def create_insight_graph():
    workflow = StateGraph(State)

    # 添加节点
    workflow.add_node("structure", report_structure_node)
    workflow.add_node("search", first_search_node)
    workflow.add_node("summary", first_summary_node)
    workflow.add_node("reflection", reflection_node)

    # 添加边
    workflow.set_entry_point("structure")
    workflow.add_edge("structure", "search")
    workflow.add_edge("search", "summary")
    workflow.add_edge("summary", "reflection")

    # 条件边
    workflow.add_conditional_edges(
        "reflection",
        lambda x: "search" if x.need_more else END
    )

    return workflow.compile()
```

### 2. 流式输出

实现流式输出以提升用户体验：

```python
# streaming_agent.py

class StreamingAgent:
    def run_streaming(self, query):
        """流式生成报告"""
        yield "# 分析开始\n\n"

        # 逐段落生成
        for paragraph in self.state.paragraphs:
            yield f"## {paragraph.title}\n\n"

            # 搜索
            yield "正在搜索相关信息...\n"
            self.search(paragraph)

            # 总结
            yield "正在分析...\n"
            summary = self.summarize(paragraph)
            yield f"{summary}\n\n"

# Flask 路由
@app.route('/api/analyze/stream')
def stream_analysis():
    query = request.args.get('query')
    agent = StreamingAgent()

    return Response(
        agent.run_streaming(query),
        mimetype='text/plain'
    )
```

### 3. 分布式部署

使用 Celery 实现分布式任务队列：

```python
# celery_tasks.py

from celery import Celery

app = Celery('bettafish', broker='redis://localhost:6379')

@app.task
def analyze_query_task(query):
    """异步分析任务"""
    agent = DeepSearchAgent()
    result = agent.run(query)
    return result

# 使用
result = analyze_query_task.delay("华为舆情分析")
print(f"任务ID: {result.id}")

# 检查结果
if result.ready():
    print(result.get())
```

---

## 📚 推荐资源

### 官方文档
- [项目 GitHub](https://github.com/666ghj/BettaFish)
- [简易 Demo](https://github.com/666ghj/DeepSearchAgent-Demo)

### 学习资料
- **LangChain 文档**: https://python.langchain.com/
- **LangGraph 教程**: https://langchain-ai.github.io/langgraph/
- **Streamlit 文档**: https://docs.streamlit.io/
- **Flask 文档**: https://flask.palletsprojects.com/

### 相关论文
- **Multi-Agent Systems**: "Communicative Agents for Software Development"
- **RAG**: "Retrieval-Augmented Generation for Knowledge-Intensive NLP Tasks"

---

## 🎉 总结

通过本学习指南，你应该能够：

✅ 理解多智能体系统的核心概念
✅ 掌握 BettaFish 的架构设计
✅ 运行和调试各个 Agent
✅ 理解 ForumEngine 协作机制
✅ 自定义和扩展系统功能

**下一步建议**：

1. **实践为主**：动手运行每个练习
2. **阅读代码**：仔细研读核心模块
3. **尝试改进**：提出自己的优化想法
4. **参与贡献**：向项目提交 PR

祝学习愉快！🚀

---

**有问题？**

- 📧 邮箱：670939375@qq.com
- 🐛 Issues：https://github.com/666ghj/BettaFish/issues
- 💬 Discussions：https://github.com/666ghj/BettaFish/discussions
