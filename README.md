# dsh-codex-provider

> **This is a community fork that ports the plugin to the dsh 0.2.x generation.**
>
> Upstream `Hu9956/dsh-codex-provider` was archived by its author and now recommends
> [Magpie](https://github.com/yetone/magpie) instead. This fork exists because the upstream
> `0.1.0` build is refused by dsh 0.2.x's plugin compatibility gate and therefore never loads:
>
> ```
> dsh: skipping profile bundle "dsh-codex-provider": Error: Plugin
> dsh-codex-provider@0.1.0 is incompatible with dsh 0.2.1-rc.3
> ```
>
> ### What this fork changes
>
> | Change | Why |
> | --- | --- |
> | `0.2.0` version; dsh peers widened from `^0.1.0-rc.6` to `>=0.1.0-rc.6 <0.3.0` | `^0.1.0-rc.6` pins the `0.1` minor, so every `0.2.x` runtime failed the gate. `<0.3.0` keeps the gate meaningful for the next generation. |
> | `create: () => schema` on every strict remote codec | dsh made strict codecs lazy in `0.1.6-alpha.2`; 0.2.x calls `codec.create()` and refuses a descriptor that only carries `schema`. |
>
> Nothing else changed: the OAuth flow, token refresh, credential refs, and the settings
> panel are the upstream implementation. Verified by booting a real dsh `0.2.1-rc.3` profile,
> which now composes `codex-provider` with no compatibility warning.
>
> ### One requirement on 0.2.x
>
> The route must name the credential reference, otherwise the plugin's tokens are never read:
>
> ```yaml
> - id: llm-pi-ai
>   config:
>     providers:
>       openai-codex:
>         apiKeyEnv: OPENAI_CODEX_API_KEY
> ```
>
> The plugin stores `OPENAI_CODEX_API_KEY` / `OPENAI_CODEX_REFRESH_TOKEN` as flat credential
> refs. A 0.2.x runtime with no `apiKeyEnv` resolves Codex auth only through a provider-scoped
> `grant` record and will ignore them, leaving the route unauthenticated.

---

DeepSeek Harness (DSH) 供应商插件：**OpenAI Codex（ChatGPT Plus/Pro 订阅）** 的设备码 OAuth 登录、令牌自动刷新与供应商管理。

在 DSH 设置中新增 **“供应商”** 分区，通过设备码授权登录 OpenAI 账号后，即可在模型选择器中使用 DSH 内置的 `openai-codex` 模型（`gpt-5.4`、`gpt-5.5`、`gpt-5.6-*` 等），消费你的 **ChatGPT Plus / Pro 订阅额度**。

## ✨ 功能

- **设备码 OAuth 登录**：在界面点击“使用 OpenAI 账号登录”，按提示完成设备码授权，无需 API Key
- **导入 Codex CLI 登录态**：直接复用本机 `~/.codex/auth.json` 中已有的登录（无需重新授权）
- **令牌自动刷新**：后台定期检查 access token，过期前自动用 refresh token 轮换（refresh_token 也会一并轮换）
- **供应商管理界面**：设置 → 供应商，展示登录状态、账号、套餐（Plus/Pro）、令牌过期时间，支持退出登录
- **激活内置 Codex 模型**：登录后 `openai-codex` 路由自动激活，模型选择器直接可选


<img width="1590" height="776" alt="ScreenShot_2026-08-14_020002_683_副本" src="https://github.com/user-attachments/assets/4c2d8b0e-edb8-46a8-876d-f297f0ec8f54" />
<img width="1892" height="1020" alt="ScreenShot_2026-08-14_043716_413" src="https://github.com/user-attachments/assets/09abd005-4432-4eb9-b9fc-6acab9772361" />


## 📦 安装

```bash
dsh plugin --profile web add github:ObnubiladO/dsh-codex-provider
```

插件包自带 DSH bundle 配置，安装时会自动加入 profile，无需手动编辑 `cordis.patch.yml`。

重启 dsh web 服务（Ctrl+C 后重新运行启动命令），刷新页面即可在 **设置 → 供应商** 看到入口。

也可以从 GitHub 安装当前源码版本：

```bash
dsh plugin --profile web add github:ObnubiladO/dsh-codex-provider
```

## 🚀 使用

1. 打开 **设置 → 供应商**
2. 点击 **“使用 OpenAI 账号登录”**，按页面提示完成设备码授权
3. 登录成功后，在模型选择器中选择 Codex 模型（如 `gpt-5.4`）开始使用

### 设备码授权注意事项（页面也会显示）

1. **登录前**：请先在 **ChatGPT 网页端 → 设置 → 账户安全与登录** 中打开“为 Codex 启用设备代码授权”开关（未开启将无法授权）；
2. **授权**：先复制页面显示的设备代码，再点击“使用 OpenAI 账号登录”，按提示填入该设备代码；
3. **登录后**：建议返回 ChatGPT 网页端关闭该开关——不影响本次登录，但下次重新登录前需重新开启。

## ⚙️ 工作原理

- 设备码 OAuth 流程直接对接 `auth.openai.com`，令牌直连 `chatgpt.com/backend-api`（与官方 Codex CLI 相同的通道与认证方式）
- access token / refresh token 存入 DSH 凭证库（`OPENAI_CODEX_API_KEY` / `OPENAI_CODEX_REFRESH_TOKEN`）；仅 Host 侧在请求和刷新时解析，绝不返回浏览器
- 凭证引用写入 `llm-pi-ai.providers.openai-codex.apiKeyEnv`，激活 DSH 内置的 `openai-codex` 路由（`routeAuth` 会在 OAuth 旁附加 harness apiKey 通道）
- 后台任务每分钟检查 access token 过期时间，剩余不足 10 分钟时自动刷新并轮换；同机多实例通过刷新锁串行处理

## ⚠️ 注意事项

- **与 Codex CLI 共享同一 OAuth 会话**：任一侧刷新令牌后，另一侧持有的旧 refresh token 可能失效（届时在 Codex CLI 侧重新 `codex login` 即可）
- **不需要 API Key**：本插件消费的是 ChatGPT 订阅额度，不是 OpenAI API 计费
- 需要 DSH `>= 0.1.0-rc.6 < 0.3.0`（本 fork 已在 `0.2.1-rc.3` 上验证）
- 可用模型由当前 DSH 版本和 OpenAI 账户权限决定

## 🔒 安全与隐私

- access token 与 refresh token 只写入 DSH 凭证库；浏览器端只接收脱敏账号信息和登录状态
- 请勿在 Issue、Discussion、日志或截图中公开设备代码、OAuth token、`~/.dsh/.credentials.yaml` 或 `~/.codex/auth.json` 的内容
- 安全问题请按 [SECURITY.md](SECURITY.md) 通过 GitHub 私下报告

## 兼容性

本 fork 针对 `@deepseek-ai/dsh 0.2.1-rc.3`（DeepDive 0.2.7）和 Node.js 22 验证；上游 0.1.0 在该版本会被兼容性检查拒绝加载。对 `0.3.0` 及以后仍然会触发检查，届时需要重新验证。

## 🛠️ 开发

| 文件 | 说明 |
|---|---|
| `lib/index.js` | host 插件：设备码 OAuth、凭证存取、令牌刷新、Typert Remote 服务（`codexProvider`） |
| `lib/client.js` | client 插件：设置页“供应商”分区 UI（ModuleLoader bundle 格式） |

## 📄 License

[MIT](LICENSE)

## 免责声明

这是社区维护的非官方插件，与 OpenAI、DeepSeek 或 DeepSeek Harness 官方没有隶属或背书关系。OpenAI、ChatGPT、Codex 和 DeepSeek 等名称与商标归各自权利人所有。
