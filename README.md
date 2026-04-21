# Replit AI 多模型代理 (replit-2api-Deployer)

一个运行在 Replit 上的 OpenAI/Anthropic 兼容 API 代理服务，把官方 SDK 请求透明转发到 Replit AI Integrations，免费使用 OpenAI、Anthropic、Gemini 三家提供商的官方模型。

> **本仓库目标**：任何 Replit Agent 拉取仓库后，按本文档操作即可在 Replit 上部署出**完全一致**的服务。请严格按顺序执行，不要跳步、不要重写代码、不要乱改 `.replit` 和 `artifact.toml`。

---

## 一、项目结构

pnpm monorepo，包含 3 个 artifact：

| 目录 | 类型 | 端口 | 路由 | 说明 |
|---|---|---|---|---|
| `artifacts/api-server` | API 后端 | `8080` | `/api`, `/v1` | Express 5，转发 OpenAI/Anthropic/Gemini 兼容请求 |
| `artifacts/api-portal` | React 前端 | `24927` | `/`（根路径） | 管理门户（查看模型/用量/配置） |
| `artifacts/mockup-sandbox` | 设计沙盒 | 任意 | — | **跟代理无关，部署时忽略即可** |

后端核心文件：
- `artifacts/api-server/src/routes/proxy.ts` — `/v1/chat/completions`、`/v1/messages`、`/v1/responses`、`/v1/models` 实现
- `artifacts/api-server/src/lib/model-sync.ts` — 启动时同步上游模型清单 + 静态 fallback
- `artifacts/api-server/src/lib/model-catalog.ts` — 模型路由能力表（思考模式、视觉支持）
- `artifacts/api-server/src/lib/config.ts` — 配置加载，默认 `PROXY_API_KEY=admin999`
- `artifacts/api-server/.replit-artifact/artifact.toml` — `paths = ["/api", "/v1"]`，端口 8080
- `artifacts/api-portal/.replit-artifact/artifact.toml` — `previewPath = "/"`，端口 24927

---

## 二、Agent 部署 SOP（请按顺序执行）

### Step 1 · 准备代码

如果是从 zip 解压：保留原始目录结构原样放进 `/home/runner/workspace/`，**不要重写任何源码**，项目已是完整实现。

如果是 `git clone`：直接进入仓库根目录即可。

### Step 2 · 添加 Replit AI Integrations（关键，必须做对）

读 `.local/skills/integrations/SKILL.md` 了解整合的添加方式。需要通过 `searchIntegrations({ query: "..." })` 然后 `addBlueprint(...)` 添加以下 **3 个** 整合（OpenRouter 已弃用，不要再加）：

| 名称 | searchIntegrations 关键词 | 必须使用的 slug |
|---|---|---|
| OpenAI | `openai` | `openai` |
| Anthropic | `anthropic` | `anthropic` |
| **Gemini** | `gemini` | **`gemini`**（**不是** `google`！） |

> ⚠️ **踩过的坑**：Gemini 整合的 provider slug 必须是 `gemini`。如果用 `google` 添加，环境变量虽然会被注入，但运行时所有 Gemini 模型都会报 `"Provider google is not supported"` 错误。

添加成功后，环境会自动注入这些环境变量，你**不需要手动设置**：
- `AI_INTEGRATIONS_OPENAI_BASE_URL` / `AI_INTEGRATIONS_OPENAI_API_KEY`
- `AI_INTEGRATIONS_ANTHROPIC_BASE_URL` / `AI_INTEGRATIONS_ANTHROPIC_API_KEY`
- `AI_INTEGRATIONS_GEMINI_BASE_URL` / `AI_INTEGRATIONS_GEMINI_API_KEY`

### Step 3 · 添加 Secret

读 `.local/skills/environment-secrets/SKILL.md`，添加一个 secret：

| Key | Value |
|---|---|
| `SESSION_SECRET` | 任意随机字符串（例如 `my-secret-2025`） |

> 不需要设 `PROXY_API_KEY`，已经在 `artifacts/api-server/config.json` 里默认 `admin999`。

### Step 4 · 安装依赖

```bash
pnpm install --no-frozen-lockfile
```

> ⚠️ 必须用 `pnpm`，**不要用 `npm install`**。`--no-frozen-lockfile` 用来兼容 lockfile 与新 Node 版本之间的细微差异。

### Step 5 · 验证 artifact 配置（不要改动）

```bash
cat artifacts/api-server/.replit-artifact/artifact.toml | grep paths
# 应输出：paths = ["/api", "/v1"]
```

如果不对，按照上面的值修正；其他字段一律不动。

### Step 6 · 启动 workflow 并自检

读 `.local/skills/workflows/SKILL.md`，确保以下 workflow 都在运行：
- `artifacts/api-server: API Server` — 命令 `pnpm --filter @workspace/api-server run dev`
- `artifacts/api-portal: web` — 命令 `pnpm --filter @workspace/api-portal run dev`

启动后 8 秒内做以下三项验证（**全部必须通过**才能进入下一步）：

```bash
# 1. 模型列表必须返回 JSON（不能是 HTML）
curl -s http://localhost:80/v1/models -H "Authorization: Bearer admin999" | head -c 300

# 2. OpenAI 通道
curl -s -X POST http://localhost:80/v1/chat/completions \
  -H "Authorization: Bearer admin999" -H "Content-Type: application/json" \
  -d '{"model":"gpt-4o-mini","messages":[{"role":"user","content":"hi"}],"max_tokens":5}'

# 3. Anthropic 通道
curl -s -X POST http://localhost:80/v1/chat/completions \
  -H "Authorization: Bearer admin999" -H "Content-Type: application/json" \
  -d '{"model":"claude-haiku-4-5","messages":[{"role":"user","content":"hi"}],"max_tokens":5}'

# 4. Gemini 通道
curl -s -X POST http://localhost:80/v1/chat/completions \
  -H "Authorization: Bearer admin999" -H "Content-Type: application/json" \
  -d '{"model":"gemini-2.5-flash","messages":[{"role":"user","content":"hi"}],"max_tokens":5}'
```

每个请求都应返回 `"choices":[{...}]`。出现 `"unknown provider"` 或 `"Provider ... is not supported"` 表示某个整合没装好或 slug 错了，回 Step 2 检查。

### Step 7 · 发布

调用 `suggestDeploy()` 让用户点 **Publish → Autoscale → Deploy**。Autoscale 适用于无状态 API。

部署完成后再次跑 Step 6 的 4 个 curl，把 URL 换成 `https://<部署域名>` 复测一遍。

---

## 三、模型清单（截至当前部署）

`/v1/models` 返回 35 个模型，**全部走官方直连**，无 OpenRouter 中转，无前缀重复版本：

- **OpenAI（20）**：gpt-5.4 / gpt-5.2 / gpt-5.1 / gpt-5 / gpt-5-mini / gpt-5-nano / gpt-5.3-codex / gpt-5.2-codex / gpt-4.1 / gpt-4.1-mini / gpt-4.1-nano / gpt-4o / gpt-4o-mini / o4-mini / o3 / o3-mini / o1 / o1-mini / o1-preview / chatgpt-4o-latest（外加 gpt-4-turbo / gpt-4 / gpt-3.5-turbo）
- **Anthropic（7）**：claude-opus-4-7 / 4-6 / 4-5 / 4-1，claude-sonnet-4-6 / 4-5，claude-haiku-4-5
- **Gemini（5）**：gemini-3.1-pro-preview / gemini-3-pro-preview / gemini-3-flash-preview / gemini-2.5-pro / gemini-2.5-flash

> Replit AI 代理只支持上述较新模型，旧版（claude-3.x、claude-2.x、gemini-2.0、gemini-1.5）已从静态 fallback 中移除，请勿再加回来——上游会拒绝。

### 路由规则（`proxy.ts:507-517`）
| 模型 ID 特征 | 走哪个上游 |
|---|---|
| 以 `claude-` 开头 | Anthropic 直连 |
| 以 `gemini-` 开头 | Gemini 直连 |
| 其他（`gpt-` / `o1-o4` / `chatgpt-`） | OpenAI 直连 |

---

## 四、客户端使用示例

部署完成后任意域名 + `admin999` 即可使用：

**OpenAI 兼容（chat completions）**
```bash
curl https://你的域名.replit.app/v1/chat/completions \
  -H "Authorization: Bearer admin999" \
  -H "Content-Type: application/json" \
  -d '{"model":"claude-opus-4-7","messages":[{"role":"user","content":"hi"}]}'
```

**Anthropic 原生（messages）**
```bash
curl https://你的域名.replit.app/v1/messages \
  -H "x-api-key: admin999" \
  -H "anthropic-version: 2023-06-01" \
  -H "Content-Type: application/json" \
  -d '{"model":"claude-opus-4-7","max_tokens":100,"messages":[{"role":"user","content":"hi"}]}'
```

**让 claude-cli 直接连本服务**
```bash
export ANTHROPIC_BASE_URL="https://你的域名.replit.app"
export ANTHROPIC_AUTH_TOKEN="admin999"
claude
```

**前端门户**：`https://你的域名.replit.app/` — 默认登录密码 `admin999`

---

## 五、常见问题排查

| 现象 | 真正原因 | 解决 |
|---|---|---|
| Gemini 全报 `Provider google is not supported` | Gemini 整合 slug 用了 `google` | 删除整合，重新用 `gemini` slug 添加 |
| `claude-3-5-haiku` / `gemini-1.5-pro` 等老模型调用失败 | Replit AI 代理已不支持 | 改用上面"模型清单"里的新模型 |
| 客户端报 `unknown provider for model XXX`，但本地 curl 直连本服务正常 | 客户端到本服务之间有第三方中转代理（如 `wx-1.9` 系列），它有自己的模型白名单 | 让客户端直连部署域名，绕过中转代理 |
| `/v1/models` 返回 HTML 而不是 JSON | api-server workflow 没启动，请求被前端门户接管 | 启动 api-server workflow，并确认 `artifact.toml` 里 `paths = ["/api","/v1"]` |
| 启动报 `Cannot find module ...` | 没装依赖或装成 npm 了 | 删 `node_modules` 后重新 `pnpm install --no-frozen-lockfile` |

---

## 六、添加新模型

仅当 Replit AI 上游真正新支持某个模型时才添加。**两个文件都要改**：

1. `artifacts/api-server/src/lib/model-sync.ts` — 加进对应 provider 的 `_STATIC` 数组
2. `artifacts/api-server/src/lib/model-catalog.ts` — 加进 `PROXY_MODEL_CATALOG`，指定 `routes`、`thinking`、`vision`

改完后重启 `artifacts/api-server: API Server` workflow，用 `curl /v1/chat/completions` 验证模型可用，再触发重新部署。

---

## 七、技术栈

- pnpm workspaces / Node 24 / TypeScript 5.9 / Express 5
- 官方 SDK：`openai`、`@anthropic-ai/sdk`、`@google/genai`
- 构建：esbuild（产物在 `artifacts/api-server/dist/`）
- 部署：Replit Autoscale（按请求扩缩容，无状态）

详细工作区约定见 `.local/skills/pnpm-workspace/SKILL.md`。
