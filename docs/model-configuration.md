# 大模型配置说明

> 本指南说明如何在 Dify 中配置大模型供应商，支持 DeepSeek 官方 API 和任意 OpenAI-compatible 接口。
> ⚠️ **重要**：不要将真实 API Key 提交到 Git，只存在于本地 `.env` 或 Dify 数据库中。

---

## 目录

1. [支持的模型供应商](#1-支持的模型供应商)
2. [DeepSeek 官方 API 配置](#2-deepseek-官方-api-配置)
3. [OpenAI-API-compatible 配置](#3-openai-api-compatible-配置)
4. [本地代理 / 中转服务配置](#4-本地代理--中转服务配置)
5. [模型选择建议](#5-模型选择建议)
6. [常见问题](#6-常见问题)

---

## 1. 支持的模型供应商

Dify 1.17.0 采用插件化模型供应商架构，支持以下方式接入大模型：

| 方式 | 适用场景 | 复杂度 |
|---|---|---|
| **DeepSeek 插件** | 使用 DeepSeek 官方 API | 简单 |
| **OpenAI-API-compatible 插件** | 使用任意 OpenAI 兼容接口（中转、本地模型等） | 中等 |
| **其他官方插件** | Anthropic、Google、Azure、通义千问、智谱等 | 简单 |
| **Custom Endpoint** | Dify 特定接口（不推荐用于通用 OpenAI 兼容） | 不推荐 |

> ⚠️ 注意：Dify 的「Custom Endpoint」期望的是 Dify 特定的 API 格式，**不是**通用的 OpenAI 兼容端点。如果你要接 OpenAI 兼容接口（包括 DeepSeek、中转服务等），请使用 **OpenAI-API-compatible 插件**，不要用 Custom Endpoint。

---

## 2. DeepSeek 官方 API 配置

### 2.1 获取 API Key

1. 访问 DeepSeek 开放平台：https://platform.deepseek.com/
2. 注册/登录账号
3. 进入「API Keys」页面，创建新的 API Key
4. 复制 Key（格式：`sk-xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx`）
5. 充值（DeepSeek 支持按量付费，价格低廉）

> DeepSeek API 价格参考（2026 年）：
> - deepseek-chat（V3）：输入 ¥1/百万 token，输出 ¥2/百万 token
> - deepseek-reasoner（R1）：输入 ¥4/百万 token，输出 ¥16/百万 token
> - 个人知识库问答场景，月成本通常在个位数元。

### 2.2 在 Dify 中安装 DeepSeek 插件

1. Dify 左侧菜单 → 「集成」→「模型供应商」
2. 点击右上角「+ 安装」或「添加模型供应商」
3. 在 Marketplace 中搜索「DeepSeek」
4. 点击安装

> 如果 Marketplace 无法访问（国内网络问题），可以：
> - 方式一：使用本地集成包（从 Dify 官方插件仓库下载 .difypkg 文件上传）
> - 方式二：使用 OpenAI-API-compatible 插件替代（见下一节）

### 2.3 配置 DeepSeek 供应商

1. 在模型供应商页面找到已安装的 DeepSeek
2. 点击「配置」或「添加模型」
3. 填入：
   - **API Key**：你的 DeepSeek API Key
   - **API Base URL**：`https://api.deepseek.com`（插件通常已预设，不需要手动填）
4. 点击保存/验证

### 2.4 添加模型

1. 在 DeepSeek 供应商页面点击「添加模型」
2. 模型类型选 **LLM**
3. 模型名称选：
   - `deepseek-chat`（通用对话模型，性价比高，推荐日常使用）
   - `deepseek-reasoner`（推理模型，适合复杂问题，但更贵）
   - 或新版命名：`deepseek-v4-flash`（快速便宜）、`deepseek-v4-pro`（更强）
4. 凭据选择刚才配置的 API Key
5. 点击保存

### 2.5 设为默认模型

1. 模型供应商页面右上角 → 「默认模型设置」
2. 推理模型（LLM）选 `deepseek-chat` 或 `deepseek-v4-flash`
3. 保存

---

## 3. OpenAI-API-compatible 配置

适用于：
- 使用 OpenAI 官方 API
- 使用其他 OpenAI 兼容的中转服务
- 使用本地部署的模型（如 Ollama、vLLM 等，需提供 OpenAI 兼容接口）

### 3.1 安装插件

1. 「集成」→「模型供应商」→「+ 安装」
2. Marketplace 搜索「OpenAI-API-compatible」
3. 安装

### 3.2 配置供应商

1. 找到 OpenAI-API-compatible，点击配置
2. 填入：
   - **API Base URL**：你的 OpenAI 兼容接口地址，必须以 `/v1` 结尾
     - OpenAI 官方：`https://api.openai.com/v1`
     - 本地代理：`http://host.docker.internal:端口/v1`（见下一节）
     - 其他中转：`https://你的中转地址/v1`
   - **API Key**：你的 API Key（如果本地代理不需要认证，可随便填）
3. 保存

### 3.3 添加模型

1. 点击「添加模型」
2. 模型类型：LLM
3. 模型名称：填你的供应商支持的模型名（如 `gpt-4o-mini`、`deepseek-chat`、`qwen-plus` 等）
4. 凭据选择刚才的配置
5. 保存

---

## 4. 本地代理 / 中转服务配置

如果你使用本地代理工具（如 CC-Switch、one-api、new-api 等）转发模型请求，需要注意 Docker 容器内的网络访问问题。

### 4.1 关键概念：host.docker.internal

Docker 容器内的 `127.0.0.1` 指向的是**容器自己**，不是你的宿主机。要从容器内访问宿主机上的服务，必须使用：

```
http://host.docker.internal:端口
```

`host.docker.internal` 是 Docker 提供的特殊域名，自动解析到宿主机 IP。

### 4.2 配置示例（CC-Switch 本地代理）

假设 CC-Switch 本地代理运行在宿主机的 15721 端口：

1. 在 Dify 中使用 OpenAI-API-compatible 插件
2. API Base URL 填：`http://host.docker.internal:15721/v1`
3. API Key：如果 CC-Switch 没设认证，随便填（如 `sk-123456`）；如果设了认证，填对应的 Key
4. 模型名称：填 CC-Switch 中配置的模型映射名

### 4.3 验证容器内能否访问宿主机

```bash
# 进入 Dify 的 api 容器
docker compose exec api bash

# 在容器内测试能否访问宿主机的代理
curl http://host.docker.internal:15721/v1/models
```

如果返回 JSON 数据，说明网络通了。

### 4.4 常见问题

| 问题 | 原因 | 解决 |
|---|---|---|
| Connection refused | 端口不对或代理没启动 | 确认代理在运行，端口正确 |
| SSL 证书错误 | 中转服务的 HTTPS 证书有问题 | 联系中转服务商，或改用 HTTP（本地代理） |
| 404 Not Found | API Base URL 缺少 `/v1` 或模型名不对 | 检查 URL 格式和模型名 |
| 401 Unauthorized | API Key 不对 | 检查 Key，确认代理是否需要认证 |

---

## 5. 模型选择建议

### 个人知识库问答场景

| 需求 | 推荐模型 | 理由 |
|---|---|---|
| **日常问答（性价比优先）** | deepseek-chat / deepseek-v4-flash | 便宜快速，满足基本问答 |
| **复杂推理（面试难题）** | deepseek-reasoner / deepseek-v4-pro | 推理能力强，但更贵 |
| **多语言 / 英文场景** | gpt-4o-mini（通过中转） | 英文理解更好 |
| **本地部署 / 隐私优先** | Ollama + Qwen2.5 / Llama3 | 数据不出本地，但需要足够硬件 |

### 成本控制建议

1. **日常开发测试用便宜模型**（deepseek-v4-flash / gpt-4o-mini）
2. **正式演示再切换到强模型**
3. 在 Dify 应用设置中配置模型降级（主模型失败时自动切换到备用模型）
4. 监控 API 调用量，设置用量告警

---

## 6. 常见问题

### Q: Dify 里搜不到 DeepSeek 供应商？
A: Dify 1.17.0 模型供应商插件化，需要先安装插件。「集成」→「模型供应商」→「+ 安装」→ Marketplace 搜索 DeepSeek。如果 Marketplace 访问不了，用 OpenAI-API-compatible 插件替代。

### Q: 测试模型连接报 404？
A: 检查 API Base URL 是否以 `/v1` 结尾，模型名称是否与供应商支持的一致。不要用 Custom Endpoint，要用 OpenAI-API-compatible 插件。

### Q: 容器内访问不了本地代理？
A: 确认用了 `host.docker.internal` 而不是 `127.0.0.1`，确认代理在运行且端口正确。

### Q: 可以同时配置多个模型供应商吗？
A: 可以。Dify 支持同时配置多个供应商和多个模型，在应用中可以选择使用哪个，也可以配置模型降级策略。

### Q: API Key 存在哪里？安全吗？
A: Dify 将 API Key 加密存储在 PostgreSQL 数据库中（使用 PKCS1_OAEP 加密）。本地部署时数据全在本地，不会上传到 Dify 官方。但仍需注意：不要将 `.env` 或包含 Key 的截图提交到公开仓库。

---

## 参考链接

- DeepSeek API 文档：https://platform.deepseek.com/docs
- Dify 模型配置文档：https://docs.dify.ai/guides/model-configuration
- OpenAI API 参考：https://platform.openai.com/docs/api-reference

---

*本指南基于 Dify 1.17.0 版本编写。*
