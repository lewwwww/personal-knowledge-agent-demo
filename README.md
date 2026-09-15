# Personal Knowledge Agent Demo

> 基于 Dify + Docker + 大模型 API 的个人知识库问答 Demo，用于验证文档向量化、知识检索、提示词约束和问答生成流程。示例数据已脱敏，不包含真实个人隐私信息。

## 项目简介

本项目演示了如何从零搭建一个**个人知识库问答 Agent**（也可称为"数字分身"）：将结构化的个人资料文档向量化存入向量数据库，通过 RAG（检索增强生成）技术，让大模型基于检索到的知识回答关于"这个人"的所有问题。

### 核心技术能力展示

| 能力 | 实现方式 |
|---|---|
| **文档向量化** | Dify 内置 Embedding 模型，自动分段 + 向量化 |
| **知识检索** | 语义检索 + 全文检索混合模式，Top-K 召回 |
| **提示词约束** | System Prompt 限定回答范围，防幻觉 |
| **多模型支持** | DeepSeek / OpenAI-compatible 接口灵活切换 |
| **工作流编排** | Dify Chatflow：用户输入 → 知识检索 → LLM → 回复 |
| **本地私有化部署** | Docker Compose 一键启动，数据全在本地 |

## 技术栈

- **应用平台**：[Dify](https://github.com/langgenius/dify) 1.17.0（开源 LLM 应用开发平台）
- **容器化**：Docker + Docker Compose
- **大模型**：DeepSeek（deepseek-v4-flash / deepseek-v4-pro），兼容 OpenAI API 格式
- **向量数据库**：Dify 内置（Weaviate / Qdrant 可切换）
- **数据库**：PostgreSQL（元数据）+ Redis（缓存）
- **前端**：Dify 内置 Web App（可生成分享链接）

## 架构与工作流程

```
用户提问
   │
   ▼
┌─────────────┐
│  用户输入    │
└──────┬──────┘
       │
       ▼
┌─────────────────┐
│  知识库检索节点  │  ←  向量数据库（已向量化的个人文档）
│  (RAG Retriever)│     语义检索 + 全文检索混合
└──────┬──────────┘
       │ 检索到的知识片段
       ▼
┌─────────────────┐
│   LLM 节点       │  ←  System Prompt（人设 + 约束 + 防幻觉）
│  (DeepSeek API) │     结合检索结果生成回答
└──────┬──────────┘
       │
       ▼
┌─────────────┐
│  直接回复    │
└─────────────┘
```

### 关键流程说明

1. **知识库上传**：将 Markdown 格式的个人资料文档上传至 Dify 知识库，系统自动分段（Chunking）并调用 Embedding 模型向量化，存入向量数据库。
2. **检索（Retrieval）**：用户提问时，检索节点将问题向量化，在向量数据库中计算余弦相似度，召回 Top-K 个最相关的文档片段。
3. **提示词约束（Prompt Engineering）**：System Prompt 明确限定 Agent 身份、回答规则（只基于检索内容、不知道就说不知道、口语化、控制长度），有效防止大模型幻觉。
4. **LLM 生成**：将检索到的知识片段 + 用户问题 + System Prompt 一起发送给大模型，生成最终回答。

## 快速开始

### 环境要求

- Docker Desktop 24.0+（Windows/macOS）或 Docker Engine（Linux）
- 至少 4GB 可用内存（推荐 8GB+）
- 至少 10GB 可用磁盘空间
- DeepSeek API Key（或其他 OpenAI-compatible 模型的 API Key）

### 1. 克隆本项目

```bash
git clone <your-repo-url>.git
cd personal-knowledge-agent-demo
```

### 2. 部署 Dify（本地 Docker）

详细步骤见 [docs/dify-deployment-guide.md](docs/dify-deployment-guide.md)，简要流程：

```bash
# 下载 Dify 官方 docker-compose.yaml 和 .env.example
# 复制 .env.example 为 .env，修改端口等配置
cp .env.example .env

# 一键启动（首次会拉取约 12 个镜像，约 3-5GB）
docker compose up -d

# 查看服务状态
docker compose ps
```

启动后访问 `http://localhost:8080`，设置管理员账号。

### 3. 配置大模型

详细步骤见 [docs/model-configuration.md](docs/model-configuration.md)：

- 在 Dify 「集成」→「模型供应商」中安装 DeepSeek 插件（或 OpenAI-API-compatible 插件）
- 填入 API Key（**不要将真实 Key 提交到 Git**）
- 启用 `deepseek-chat` 或 `deepseek-v4-flash` 模型
- 设为默认推理模型

### 4. 创建知识库并上传示例文档

1. Dify 左侧菜单 → 「知识库」→「创建即用型知识库」
2. 上传 [examples/knowledge-base-sample.md](examples/knowledge-base-sample.md)（虚拟人物示例，已脱敏）
3. 分段方式选「自动分段」，等待向量化完成
4. 用「召回测试」验证检索效果

### 5. 创建 Chatflow 应用

1. 「工作室」→「创建空白应用」→ 选 **Chatflow**
2. 编排工作流：`用户输入 → 知识检索 → LLM → 直接回复`
3. 知识检索节点关联刚创建的知识库
4. LLM 节点填入 [prompts/system-prompt-template.md](prompts/system-prompt-template.md) 中的提示词模板
5. 选择 DeepSeek 模型
6. 保存并发布，获取 Web App 分享链接

### 6. 测试问答

在预览窗口或分享链接中提问：

- `你是谁？用一句话介绍自己`
- `你有什么项目经验？`
- `你的 GPA 是多少？`
- `你抗压能力怎么样？讲一个具体的例子`（测试 L6 软素质答案）
- `你最不擅长什么？`（测试边界约束）

## 项目结构

```
personal-knowledge-agent-demo/
├── README.md                          # 本文件：项目说明、架构、快速开始
├── .env.example                       # Dify 环境变量模板（无真实 Key）
├── .gitignore                         # Git 忽略规则（保护敏感文件）
├── examples/
│   └── knowledge-base-sample.md       # 示例知识库（虚拟人物，已脱敏）
├── prompts/
│   └── system-prompt-template.md      # 脱敏后的系统提示词模板
├── docs/
│   ├── dify-deployment-guide.md       # Dify 本地 Docker 部署详细指南
│   └── model-configuration.md         # 大模型配置说明（DeepSeek / OpenAI 兼容）
└── screenshots/
    └── README.md                      # 截图放置说明（Chatflow 流程、效果演示）
```

## 提示词设计（防幻觉核心）

本项目的核心技术亮点之一是**通过提示词工程约束大模型行为**，防止幻觉。关键约束包括：

1. **身份锚定**：明确 Agent 是"某个人的数字分身"，用第一人称回答
2. **知识边界**：只回答知识库中有依据的内容，不知道就明确说"资料里没有记录"
3. **一致性约束**：回答口径必须与知识库（简历）完全一致，不新增、不夸大、不编造
4. **风格控制**：口语化、真诚、避免 AI 腔，回答控制在 2-4 句话
5. **敏感信息保护**：不主动透露联系方式、薪资等敏感信息
6. **软素质兜底（v2 新增）**：软素质类问题（抗压、协作、缺点、职业规划）优先从知识库 L6 层提取现成答案，修复"行为面试题答不上来"的翻车点
7. **数字证据优先（v2 新增）**：回答优先引用量化成果（准确率、耗时、吞吐、排名），增强说服力
8. **追问闭环（v2 新增）**：技术细节追问时引用 L2 经历的"可追问细节"，防止细节层面编造

知识库采用 **L1-L6 分层结构**（身份定位 / 经历 / 硬指标 / 技能 / 亮点 / 面试问答），完整模板见 [prompts/system-prompt-template.md](prompts/system-prompt-template.md)，示例知识库见 [examples/knowledge-base-sample.md](examples/knowledge-base-sample.md)。

## 隐私保护说明

### 本仓库已脱敏

- 示例知识库使用**虚拟人物**（张明，计算机科学与技术专业），所有经历、成绩、项目均为虚构示例
- **不包含**任何真实个人信息：姓名、身份证号、手机号、邮箱、家庭住址、真实简历
- **不包含**任何 API Key、密码、Token 等敏感凭证
- `.env` 文件已加入 `.gitignore`，不会被提交

### 使用本项目时的隐私建议

1. **不要将真实简历直接上传到公开仓库**，真实资料仅在本地 Dify 中使用
2. **API Key 只存在于本地 `.env`**，绝不提交到 Git，也不要在截图中暴露
3. 如需分享演示效果，使用本仓库提供的**虚拟人物示例数据**
4. 部署到云服务器时，配置访问密码或 IP 白名单，避免被陌生人滥用消耗 API 额度
5. 定期检查 Dify 日志，确认无异常访问

## 常见问题

### Q: Docker 启动后访问不了？
A: 检查端口是否冲突（默认 8080），确认 15 个容器全部 Up，参考 [docs/dify-deployment-guide.md](docs/dify-deployment-guide.md) 的排错部分。

### Q: 模型调用报错 404？
A: 确认 API Base URL 格式正确（通常以 `/v1` 结尾），模型名称与供应商支持的一致。参考 [docs/model-configuration.md](docs/model-configuration.md)。

### Q: 回答与知识库不符（幻觉）？
A: 检查提示词中的约束是否生效，确认知识检索节点已正确关联知识库，适当提高 Top-K 或调整检索阈值。

### Q: 可以不用 Dify，自己写代码实现吗？
A: 可以。Dify 是低代码验证方案，技术原理相同：文档分段 → Embedding 向量化 → 向量数据库存储 → 检索 → LLM 生成。可使用 LangChain/LlamaIndex + Chroma/Faiss + FastAPI 自建。

## 技术延伸方向

- **自建方案**：用 Chroma（向量数据库）+ FastAPI（后端）+ DeepSeek API 从零实现，面试时可讲解 RAG 全链路技术细节
- **多模型对比**：接入 GPT、Claude、DeepSeek 等多模型，对比回答质量与成本
- **语音问答**：集成语音识别 + TTS，实现语音交互
- **简历自动解析**：PDF 简历自动解析入库，减少手动整理
- **面试反馈分析**：记录问答日志，分析高频问题与回答质量

## 许可证

MIT License

## 免责声明

本项目仅用于技术学习与演示，示例数据均为虚构，不代表任何真实人物。使用本项目时请遵守相关法律法规，尊重个人隐私。大模型生成内容仅供参考，不构成任何专业建议。
