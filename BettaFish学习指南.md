# 🐟 BettaFish（微舆）项目学习指南

> **作者的话**：这是一份为你量身定制的学习路线图，帮助你从零开始理解这个创新的多智能体舆情分析系统。
> **学习方式**：先通读大纲，确认后我们逐部分深入学习！

---

## 📚 学习路线总览

```
第一阶段：认识项目（基础认知）
    ↓
第二阶段：理解核心概念（理论基础）
    ↓
第三阶段：探索系统架构（宏观设计）
    ↓
第四阶段：深入四大 Agent（核心功能）
    ↓
第五阶段：学习关键技术（技术细节）
    ↓
第六阶段：实战与扩展（动手实践）
```

---

## 🎯 第一阶段：认识项目（基础认知）
**学习目标**：从宏观视角了解"微舆"是什么，能做什么

### 1.1 项目概述
- **是什么**：多智能体舆情分析系统
- **比喻理解**：像一个"AI情报分析团队"
  - 有专门搜索网络的情报员（QueryEngine）
  - 有分析图片视频的多媒体专家（MediaEngine）
  - 有挖掘数据库的数据分析师（InsightEngine）
  - 有撰写报告的文案专家（ReportEngine）
  - 还有一个主持人协调所有人讨论（ForumEngine）

### 1.2 核心价值
- **六大优势**：
  1. AI驱动的全域监控（7x24小时爬虫）
  2. 超越LLM的复合分析引擎（多Agent协作）
  3. 强大的多模态能力（图文视频都能分析）
  4. Agent"论坛"协作机制（避免思维局限）
  5. 公私域数据无缝融合（内外部数据打通）
  6. 轻量化与高扩展性（纯Python模块化）

### 1.3 应用场景
- 品牌声誉监测
- 社会热点事件分析
- 市场趋势预测
- 竞品分析

---

## 🧠 第二阶段：理解核心概念（理论基础）
**学习目标**：掌握项目涉及的关键技术概念

### 2.1 什么是"多智能体系统"（Multi-Agent System）
**大白话**：就像一个团队分工合作，每个人有自己的专业技能和工具

**专业定义**：
- **Agent（智能体）**：能够感知环境、自主决策、执行任务的软件实体
- **Multi-Agent**：多个Agent相互协作，完成单个Agent无法完成的复杂任务

**本项目的四个Agent**：
| Agent | 职责 | 类比 |
|-------|------|------|
| QueryEngine | 网络搜索 | 情报搜索员 |
| MediaEngine | 多模态分析 | 多媒体专家 |
| InsightEngine | 数据库挖掘 | 数据分析师 |
| ReportEngine | 报告生成 | 文案撰写员 |

### 2.2 什么是"论坛协作机制"（Forum Collaboration）
**大白话**：让AI们像开会一样讨论问题

**工作原理**：
1. 各Agent独立工作，记录自己的发现
2. ForumEngine（主持人）监控所有Agent的发言
3. 生成总结，引导下一轮讨论方向
4. 各Agent读取论坛内容，调整研究方向
5. 多轮迭代，直到得出结论

### 2.3 什么是"LLM"（大语言模型）
**大白话**：就是像ChatGPT那样能理解和生成文字的AI

**本项目使用**：
- OpenAI兼容接口（统一标准）
- 支持多种模型：Kimi、Gemini、DeepSeek、Qwen等

### 2.4 什么是"爬虫"（Web Crawler）
**大白话**：自动化程序，能够模拟人类浏览网页，批量收集数据

**本项目的爬虫**：
- **MindSpider**：专门爬取微博、小红书、抖音等社交媒体数据
- 使用Playwright（能执行JavaScript的高级爬虫工具）

---

## 🏗️ 第三阶段：探索系统架构（宏观设计）
**学习目标**：理解系统如何组织、各部分如何协作

### 3.1 整体架构图解读
```
用户提问（Web界面）
    ↓
Flask主应用（app.py）
    ↓
并行启动三个Agent
    ├─→ QueryEngine（网络搜索）
    ├─→ MediaEngine（多模态分析）
    └─→ InsightEngine（数据库挖掘）
    ↓
循环阶段：论坛协作 + 深度研究
    ├─→ ForumEngine监控并生成总结
    ├─→ 各Agent根据论坛引导深入研究
    └─→ 多轮迭代
    ↓
ReportEngine收集结果
    ↓
生成最终HTML报告
```

### 3.2 完整分析流程（12步）
| 步骤 | 阶段名称 | 主要操作 |
|------|----------|----------|
| 1 | 用户提问 | 在Web界面输入分析需求 |
| 2 | 并行启动 | 三个Agent同时开始工作 |
| 3 | 初步分析 | 各Agent用专属工具概览搜索 |
| 4 | 策略制定 | 基于初步结果制定分块研究策略 |
| 5-N | **循环阶段** | **论坛协作 + 深度研究（多轮）** |
| N+1 | 结果整合 | ReportEngine收集所有分析结果 |
| N+2 | 报告生成 | 动态选择模板生成HTML报告 |

### 3.3 项目代码结构
```
BettaFish/
├── app.py                         # 【核心入口】Flask主应用
├── config.py                      # 【全局配置】统一管理所有配置
│
├── QueryEngine/                   # 【Agent 1】网络搜索引擎
│   ├── agent.py                   # Agent主逻辑
│   ├── tools/                     # 搜索工具（Tavily、Bocha等）
│   └── llms/                      # LLM接口封装
│
├── MediaEngine/                   # 【Agent 2】多模态分析引擎
│   ├── agent.py                   # Agent主逻辑
│   └── tools/                     # 多模态工具（图片、视频分析）
│
├── InsightEngine/                 # 【Agent 3】数据库挖掘引擎
│   ├── agent.py                   # Agent主逻辑
│   ├── tools/                     # 数据库查询工具
│   │   ├── search.py              # 核心搜索工具
│   │   ├── sentiment_analyzer.py  # 情感分析
│   │   └── keyword_optimizer.py   # 关键词优化
│   └── prompts/                   # 提示词模板
│
├── ReportEngine/                  # 【Agent 4】报告生成引擎
│   ├── agent.py                   # Agent主逻辑
│   ├── report_template/           # 报告模板库
│   └── nodes/                     # 报告生成节点
│
├── ForumEngine/                   # 【协作引擎】论坛机制
│   ├── monitor.py                 # 监控Agent发言
│   └── llm_host.py                # 主持人LLM模块
│
├── MindSpider/                    # 【爬虫系统】数据采集
│   ├── BroadTopicExtraction/      # 热点话题提取
│   └── DeepSentimentCrawling/     # 深度舆情爬取
│
├── SentimentAnalysisModel/        # 【情感分析模型】
│   ├── WeiboMultilingualSentiment/  # 多语言情感分析
│   └── WeiboSentiment_SmallQwen/    # 小参数Qwen微调
│
└── utils/                         # 【通用工具】
    ├── forum_reader.py            # Agent间通信
    └── retry_helper.py            # 网络重试机制
```

---

## 🤖 第四阶段：深入四大 Agent（核心功能）
**学习目标**：理解每个Agent的工作原理和代码实现

### 4.1 QueryEngine（网络搜索Agent）
**职责**：在国内外网络上搜索相关信息

**工具箱**：
- Tavily API（国际搜索）
- Bocha API（国内搜索）
- 网页内容提取工具

**工作流程**：
1. 接收搜索任务
2. 生成搜索关键词
3. 调用搜索API
4. 提取和清洗内容
5. 反思和优化（最多2轮）

**关键代码位置**：
- `QueryEngine/agent.py` - Agent主逻辑
- `QueryEngine/tools/` - 搜索工具实现
- `QueryEngine/nodes/` - 各个处理节点

### 4.2 MediaEngine（多模态分析Agent）
**职责**：分析图片、视频等多媒体内容

**能力**：
- 图片OCR文字提取
- 视频关键帧分析
- 结构化信息卡片提取（天气、日历、股票等）

**工作流程**：
1. 识别内容类型（文本/图片/视频）
2. 调用多模态LLM（如Gemini）
3. 提取关键信息
4. 结构化存储

**关键代码位置**：
- `MediaEngine/agent.py` - Agent主逻辑
- `MediaEngine/tools/` - 多模态工具

### 4.3 InsightEngine（数据库挖掘Agent）
**职责**：从私有舆情数据库中挖掘深度信息

**工具箱**：
- `search_hot_content()` - 搜索热榜内容
- `search_topic_globally()` - 全局话题搜索
- `get_comments_for_topic()` - 获取话题评论
- `search_topic_on_platform()` - 特定平台搜索
- `sentiment_analyzer` - 情感分析工具
- `keyword_optimizer` - 关键词优化（Qwen模型）

**工作流程**：
1. 理解用户需求，制定搜索策略
2. 使用多种工具从数据库查询
3. 情感分析（调用微调模型）
4. 关键词优化（提高搜索精度）
5. 总结和反思

**关键代码位置**：
- `InsightEngine/agent.py` - Agent主逻辑
- `InsightEngine/tools/search.py` - 数据库查询
- `InsightEngine/tools/sentiment_analyzer.py` - 情感分析
- `InsightEngine/tools/keyword_optimizer.py` - 关键词优化

### 4.4 ReportEngine（报告生成Agent）
**职责**：收集所有Agent的结果，生成专业报告

**能力**：
- 模板选择（根据场景自动选择）
- 多轮生成（保证质量）
- HTML美化

**报告模板类型**：
- 社会公共热点事件分析
- 商业品牌舆情监测
- 政府政策影响评估
- （更多模板...）

**工作流程**：
1. 收集三个Agent的分析结果
2. 读取论坛讨论记录
3. 选择合适的报告模板
4. 多轮生成和优化
5. 输出HTML报告

**关键代码位置**：
- `ReportEngine/agent.py` - Agent主逻辑
- `ReportEngine/report_template/` - 模板库
- `ReportEngine/nodes/` - 报告生成节点

---

## 🔧 第五阶段：学习关键技术（技术细节）
**学习目标**：掌握项目中的关键技术实现

### 5.1 LLM统一接口封装
**问题**：不同LLM厂商API格式不统一，如何管理？

**解决方案**：OpenAI兼容接口
```python
# 所有Agent的LLM调用都使用这种统一格式
from openai import OpenAI

client = OpenAI(
    api_key="your_api_key",
    base_url="https://api.example.com/v1"
)

response = client.chat.completions.create(
    model="model-name",
    messages=[{"role": "user", "content": "your prompt"}]
)
```

**配置文件**（`config.py`）：
- 每个Agent有独立的API配置
- 支持环境变量和.env文件

### 5.2 数据库设计与查询优化
**数据库选择**：PostgreSQL（推荐）或MySQL

**核心表结构**：
- 热榜内容表（hot_content）
- 话题表（topics）
- 评论表（comments）
- 平台数据表（platform_data）

**查询优化技术**：
- 索引优化
- 连接池管理（SQLAlchemy）
- 异步查询（aiomysql/asyncpg）
- 分批查询（避免内存溢出）

### 5.3 情感分析技术
**多种方法对比**：
| 方法 | 优点 | 缺点 | 适用场景 |
|------|------|------|----------|
| BERT微调 | 准确度高 | 速度较慢 | 高精度需求 |
| Qwen微调 | 中文理解好 | 模型较大 | 中文为主场景 |
| 多语言模型 | 支持多语言 | 通用性强但不够精准 | 国际化场景 |
| 机器学习（SVM/XGBoost） | 速度快 | 准确度一般 | 实时性要求高 |

**本项目默认**：多语言情感分析模型（推荐）

### 5.4 爬虫技术（MindSpider）
**技术栈**：
- Playwright（支持JavaScript渲染）
- 反爬虫策略（User-Agent、代理、限速）
- 数据清洗（BeautifulSoup、正则表达式）

**两大模块**：
1. **BroadTopicExtraction**：热点话题提取
   - 获取今日新闻
   - 提取关键词
   - 存入数据库

2. **DeepSentimentCrawling**：深度舆情爬取
   - 根据关键词爬取各平台内容
   - 支持微博、小红书、抖音、快手
   - 爬取评论数据

### 5.5 论坛协作机制（ForumEngine）
**核心思想**：通过日志文件实现Agent间通信

**实现方式**：
1. 每个Agent将发现写入日志
2. ForumEngine监控日志文件
3. 提取关键信息，调用LLM生成总结
4. 各Agent通过`forum_reader.py`读取论坛内容
5. 根据论坛引导调整研究方向

**关键文件**：
- `ForumEngine/monitor.py` - 日志监控
- `ForumEngine/llm_host.py` - 主持人LLM
- `utils/forum_reader.py` - 论坛读取工具

---

## 💻 第六阶段：实战与扩展（动手实践）
**学习目标**：动手运行项目，尝试扩展功能

### 6.1 环境搭建实战
**任务清单**：
- [ ] 安装Python 3.9+
- [ ] 创建虚拟环境（Conda或venv）
- [ ] 安装依赖包（`pip install -r requirements.txt`）
- [ ] 安装Playwright浏览器驱动
- [ ] 配置数据库（PostgreSQL/MySQL）
- [ ] 配置LLM API密钥（.env文件）
- [ ] 运行测试（`python app.py`）

### 6.2 运行第一个完整分析
**步骤**：
1. 启动主应用：`python app.py`
2. 访问：http://localhost:5000
3. 输入分析需求：例如"分析武汉大学品牌声誉"
4. 观察各Agent工作日志
5. 查看最终生成的HTML报告（`final_reports/`目录）

### 6.3 单独测试某个Agent
**示例：测试InsightEngine**
```bash
streamlit run SingleEngineApp/insight_engine_streamlit_app.py --server.port 8501
```

### 6.4 扩展实战：接入自定义数据源
**场景**：你有一个客户反馈数据库，想接入系统

**步骤**：
1. 在`config.py`添加数据库配置
2. 创建自定义工具类（参考`InsightEngine/tools/search.py`）
3. 在`InsightEngine/agent.py`中集成工具
4. 修改Prompt，让Agent知道新工具的用法
5. 测试运行

### 6.5 扩展实战：自定义报告模板
**步骤**：
1. 在`ReportEngine/report_template/`创建新模板
2. 使用Markdown格式编写
3. 在Web界面上传并选择
4. 生成报告

### 6.6 爬虫实战：爬取微博数据
**步骤**：
```bash
cd MindSpider
python main.py --setup              # 初始化
python main.py --broad-topic        # 爬取热点话题
python main.py --deep-sentiment --platforms wb  # 深度爬取微博
```

---

## 📖 学习资源推荐

### 推荐阅读顺序
1. **先读文档**：`README.md`（项目概述）
2. **再看架构**：`app.py`（入口文件）
3. **深入Agent**：从`InsightEngine/agent.py`开始
4. **理解工具**：`InsightEngine/tools/search.py`
5. **学习协作**：`ForumEngine/monitor.py`

### 相关技术学习
- **Flask**：https://flask.palletsprojects.com/
- **Streamlit**：https://docs.streamlit.io/
- **LangGraph**（类似架构）：https://langchain-ai.github.io/langgraph/
- **Playwright**：https://playwright.dev/python/

---

## ❓ 常见问题预设

### Q1: 为什么要用多Agent而不是单个LLM？
**A**：单个LLM容易产生"思维局限"和"信息茧房"，多Agent通过不同工具和视角，能产生更全面、更客观的分析。

### Q2: 论坛协作机制的意义是什么？
**A**：让不同Agent的发现互相启发，避免重复劳动，提高分析深度。就像团队开会讨论，比单打独斗更高效。

### Q3: 为什么需要那么多LLM模型？
**A**：不同模型有不同优势：
- Kimi长上下文能力强
- Gemini多模态能力强
- DeepSeek推理能力强
- Qwen中文理解好

### Q4: 如何降低API成本？
**A**：
- 使用国内中转服务（价格更低）
- 优化Prompt减少token消耗
- 使用小参数模型处理简单任务
- 缓存重复查询结果

---

## 🎓 学习检查清单

完成以下清单，说明你已经掌握了BettaFish项目！

### 基础理解（第1-3阶段）
- [ ] 能用自己的话解释什么是多智能体系统
- [ ] 能画出系统架构图
- [ ] 理解论坛协作机制的工作原理
- [ ] 知道四个Agent的职责分工

### 代码理解（第4阶段）
- [ ] 能读懂`app.py`主流程
- [ ] 理解任意一个Agent的代码结构
- [ ] 知道LLM如何被调用
- [ ] 理解数据库查询逻辑

### 实战能力（第5-6阶段）
- [ ] 成功搭建开发环境
- [ ] 运行过完整分析流程
- [ ] 尝试修改配置参数
- [ ] 能接入自定义数据源
- [ ] 能创建自定义报告模板

### 高级能力（可选）
- [ ] 添加新的搜索工具
- [ ] 开发新的Agent
- [ ] 优化情感分析模型
- [ ] 扩展爬虫支持新平台

---

## 📝 学习建议

### 时间规划
- **第1-2阶段**：2-3天（理论学习）
- **第3阶段**：1-2天（架构理解）
- **第4阶段**：5-7天（代码深入，每个Agent 1-2天）
- **第5阶段**：3-5天（技术细节）
- **第6阶段**：根据实践项目而定

### 学习方法
1. **先整体后局部**：不要一开始就陷入代码细节
2. **动手实践**：看懂≠会用，一定要运行起来
3. **问题驱动**：遇到不懂的先记录，再集中解决
4. **对比学习**：对比不同Agent的实现，理解设计思路
5. **扩展练习**：尝试改造成其他领域的分析系统

---

## 🚀 准备好了吗？

**下一步**：请告诉我你是否同意这个学习大纲？
- 如果同意，我们从**第一阶段**开始深入学习！
- 如果有调整建议，请告诉我你更关注哪些部分！

**我的承诺**：
- 每个知识点都会用"大白话+专业术语"讲解
- 遇到复杂逻辑会画流程图
- 关键代码会逐行剖析
- 随时欢迎你的提问！

让我们一起攻克这个项目吧！💪
