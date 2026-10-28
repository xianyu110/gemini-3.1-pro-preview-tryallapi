# Gemini 3.1 Pro Preview API 调用教程：用 TryAllAPI 做 OpenAI 兼容对话接入

> **Gemini 3.1 Pro Preview**（TryAllAPI 模型 ID：`gemini-3.1-pro-preview`）是 Google Gemini 3.1 系列偏智能 / 推理的 Pro 档预览模型，面向 agentic workflows、自主编码与复杂多模态任务。价格接口标签为「对话、思考、多模态」。本文记录如何通过聚合站 **[TryAllAPI](https://tryallapi.com/)**，从模型广场 / 价格页选型、读基价与分组倍率，到 `POST /v1/chat/completions` 的完整接入流程。**调用时务必使用含 `-preview` 的精确 model 字符串。**价格与分组倍率取数 **2026-10-28（北京）**，**以页面实时为准**；示例仅为请求形态说明，未跑通真实长任务计费。

**Keywords:** Gemini 3.1 Pro Preview · gemini-3.1-pro-preview · TryAllAPI · OpenAI 兼容 · gemini · `/v1/chat/completions` · preview

---

## 目录

- [一、为什么用聚合站调 Gemini](#一为什么用聚合站调-gemini)
- [二、在价格页 / 模型广场找到 Gemini 3.1 Pro Preview](#二在价格页-模型广场找到-gemini-31-pro-preview)
- [三、模型怎么选：与相邻档对照](#三模型怎么选与相邻档对照)
- [四、价格怎么读](#四价格怎么读)
- [五、调用流程：Key → Base URL → chat/completions](#五调用流程key-base-url-chatcompletions)
- [六、curl / SDK 最小示例](#六curl-sdk-最小示例)
- [七、常见注意](#七常见注意)
- [FAQ](#faq)
- [八、小结与试用入口](#八小结与试用入口)

---

## 一、为什么用聚合站调 Gemini

只用 Google AI Studio / Vertex、能走官方控制台充值开票时，直连往往最干净；如果项目里还要同时调 GPT、Claude、Gemini、国内模型，为每家维护一套 Key 和重试逻辑也比较散。

聚合站（如 [TryAllAPI](https://tryallapi.com/)）的常见价值：

1. **统一鉴权**：一个 Bearer Key，切换 `model` 字段即可试不同模型；
2. **OpenAI 兼容形态**：Base URL 固定为 `https://tryallapi.com/v1`，Cursor / Dify / 各类 SDK 改地址即可；
3. **模型广场可对照**：标签、分组倍率、端点类型一目了然。

代价是多一跳：延迟与可用性取决于分组上游，建议先用小流量探针。注册试用入口：[https://tryallapi.com/register?aff=5A6A](https://tryallapi.com/register?aff=5A6A)。

---

## 二、在价格页 / 模型广场找到 Gemini 3.1 Pro Preview

打开 [价格页](https://tryallapi.com/pricing) 或模型广场，按供应商 **Google**、类型 **对话**、关键词 `3.1-pro` / `gemini-3.1-pro-preview` 筛选。详情卡截图如下：

![Gemini 3.1 Pro Preview 模型广场详情卡：模型 ID、API 与分组价格](model-square.png)

详情卡要点（取自价格接口 2026-10-28（北京） 快照）：

| 字段 | 内容 |
| --- | --- |
| **模型 ID** | `gemini-3.1-pro-preview`（必须含 `-preview`） |
| **供应商** | Google |
| **标签** | 对话、思考、多模态 |
| **端点类型** | `gemini`、`openai`（`POST /v1/chat/completions`） |
| **可用分组** | Aistudio-Gemini-1、Aistudio-Gemini-2、Aistudio-Gemini-3、Aistudio-Gemini-4、Vertex-Gemini-1、Vertex-Gemini-2、Vertex-Gemini-3 |
| **计费** | 按量（token） |

文中 UI 品牌可能显示为模型广场 / 第三方壳，实际对接产品以 [TryAllAPI](https://tryallapi.com/) 控制台与价格页为准。

---

## 三、模型怎么选：与相邻档对照

只根据价格接口可见信息做对照，**不编造未出现的能力差异**。基价按价格接口换算（$/1M，未乘分组倍率）：

| 模型 ID（可见） | 基价 输入 / 输出 / 缓存读 | 怎么理解 |
| --- | --- | --- |
| **`gemini-3.1-pro-preview`** | $2 / $12 / $0.2 | 本文主线：Gemini 3.1 Pro 预览档；复杂推理 / 多模态常用 |
| `gemini-3-flash-preview` | $0.5 / $3 / $0.05 | 同系列可见 Flash 预览档；换档只改 model |
| `gemini-3.8-flash` | $0.75 / $3.75 / $0.075 | 同供应商 GA Flash 旗舰档；生产吞吐常用 |
| `gemini-3.5-flash` | $1.5 / $9 / $0.15 | 同供应商可见 Flash 档；对照价位用 |

实务上：经 TryAllAPI 调用统一写精确字符串 `gemini-3.1-pro-preview`；遇限流或分组无货，再在同厂商可见 ID 之间切换，而不是改 Base URL。

---

## 四、价格怎么读

对话模型常见公式：**≈ 基价 × 分组倍率**（以控制台为准）。取数快照：**2026-10-28（北京）**；**以页面实时为准**。

换算约定：基价输入 $= model_ratio×2；输出 $= 输入×completion_ratio；缓存读 $= 输入×cache_ratio。分组估价 = 基价 × group_ratio。

### 基价（每百万 tokens，由价格接口换算）

| 计费项 | 基价（$/1M，未乘分组） |
| --- | --- |
| 输入 | **$2** |
| 输出 | **$12** |
| 缓存读 | **$0.2** |

公开接口按 **基价 × group_ratio** 估算即可；**实际扣费以控制台为准**。

另：接口可见 `step_ratios`——输入超过约 200K tokens 时，官方长上下文加价结构在接口有 step（prompt×2 / completion×1.5）。具体阶梯边界与演算以价格页为准，本文不展开计费演算。

### 分组价格估算（基价 × group_ratio，$/1M）

| 分组 | group_ratio | 输入估 | 缓存读估 | 输出估 |
| --- | --- | --- | --- | --- |
| **Aistudio-Gemini-1** | 0.388234 | ≈ $0.7765 | ≈ $0.0776 | ≈ $4.6588 |
| **Aistudio-Gemini-2** | 0.582362 | ≈ $1.1647 | ≈ $0.1165 | ≈ $6.9883 |
| **Aistudio-Gemini-3** | 0.970596 | ≈ $1.9412 | ≈ $0.1941 | ≈ $11.65 |
| **Aistudio-Gemini-4** | 2.102958 | ≈ $4.2059 | ≈ $0.4206 | ≈ $25.24 |
| **Vertex-Gemini-1** | 0.44 | ≈ $0.88 | ≈ $0.088 | ≈ $5.28 |
| **Vertex-Gemini-2** | 0.66 | ≈ $1.32 | ≈ $0.132 | ≈ $7.92 |
| **Vertex-Gemini-3** | 1.98 | ≈ $3.96 | ≈ $0.396 | ≈ $23.76 |

**最终扣费以控制台流水为准**。

---

## 五、调用流程：Key → Base URL → chat/completions

```text
控制台创建 API Key（选本模型 enable_groups 中的分组）
  → Base URL = https://tryallapi.com/v1
  → POST /v1/chat/completions   （model=gemini-3.1-pro-preview）
  → 读 choices[0].message.content（或流式 delta）
```

建议：

- **超时**：长输出 / 思考模式建议开流式并调高 timeout；
- **密钥**：只放环境变量（如 `TRYALLAPI_KEY`），勿提交仓库；
- **精确 model 字符串**：必须带 `-preview` 后缀；漏写会 404 或落到别的档。

---

## 六、curl / SDK 最小示例

占位 Key `sk-xxx`（**不要**填入真实 Key）。

### 1）OpenAI 兼容 chat/completions

```bash
curl -sS https://tryallapi.com/v1/chat/completions \
  -H 'Authorization: Bearer sk-xxx' \
  -H 'Content-Type: application/json' \
  -d '{
    "model": "gemini-3.1-pro-preview",
    "messages": [
      {"role": "user", "content": "把这份多模态需求拆成可执行的验收清单（不超过 8 条）"}
    ]
  }'
```

### 2）Python（OpenAI SDK）

```python
from openai import OpenAI

client = OpenAI(
    api_key="sk-xxx",  # 建议改为读取环境变量
    base_url="https://tryallapi.com/v1",
    timeout=180,
)

r = client.chat.completions.create(
    model="gemini-3.1-pro-preview",
    messages=[{"role": "user", "content": "用三句话总结：如何给这段 agent 工作流写回归用例。"}],
)
print(r.choices[0].message.content)
```

Cursor：Settings → Models → OpenAI API Key + Override Base URL `https://tryallapi.com/v1` → 自定义模型 `gemini-3.1-pro-preview`（含 `-preview`）。Dify：OpenAI-API-compatible，endpoint 填 `https://tryallapi.com/v1`。

---

## 七、常见注意

1. **模型名**：请求体 `"model"` 必须写广场上的精确 ID `gemini-3.1-pro-preview`（含点号与 `-preview`，大小写敏感）；写混成 `gemini-3.1-pro` 容易 404。
2. **基价 × 倍率**：估算用价格接口快照；长上下文 step 加价以页面为准。
3. **分组**：403 多为分组无货，换 `enable_groups` 里其他分组（Aistudio / Vertex）；低倍率分组先抽检质量与延迟。
4. **报错速查**：401 查 Bearer / 令牌；404 确认模型名且 base_url 含 `/v1`；429 退避或换分组。
5. **合规**：第三方聚合不是 Google 官方服务，敏感代码先看条款。

---

## FAQ

### Gemini 3.1 Pro Preview 的模型 ID 是什么？

TryAllAPI 上是 **`gemini-3.1-pro-preview`**（必须含 `-preview`）。请求体 `"model"` 填该字符串。

### TryAllAPI 的对话 endpoint 是什么？

- Base URL：`https://tryallapi.com/v1`
- 对话：`POST /v1/chat/completions`
- 价格接口同时标注 `gemini` 端点类型；本文以 OpenAI 兼容为主。

### 基价大概是多少？

按 2026-10-28（北京） 价格接口快照：输入约 $2 / 输出约 $12 / 缓存读约 $0.2（$/1M，未乘分组）。再 × `group_ratio` 得分组估价；**以页面实时为准**。

### 如何试用 TryAllAPI？

站点首页：[https://tryallapi.com/](https://tryallapi.com/)；注册试用入口：[https://tryallapi.com/register?aff=5A6A](https://tryallapi.com/register?aff=5A6A)。

---

## 八、小结与试用入口

把 Gemini 3.1 Pro Preview 接到业务里，核心是：**在价格页确认精确 model id（含 `-preview`）与 OpenAI 兼容 Base URL → 读懂基价 × 分组倍率 → 用 chat/completions 先跑通非流式 ping，再开流式 / 接多模态输入**。

| 资源 | 链接 |
| --- | --- |
| TryAllAPI 首页 | [https://tryallapi.com/](https://tryallapi.com/) |
| 注册试用（aff） | [https://tryallapi.com/register?aff=5A6A](https://tryallapi.com/register?aff=5A6A) |
| 价格页 | [tryallapi.com/pricing](https://tryallapi.com/pricing) |

价格取数 2026-10-28（北京），以页面实时为准。

---

## License

本文档以个人接入笔记形式发布，示例代码可自由复用。截图与模型广场文案归原平台；价格以实时页面为准。TryAllAPI 为第三方聚合，非 Google 官方产品。
