# 零成本公开模型中转站 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 用 Render、Supabase 和 New API 部署一个可公开注册、人工充值、单上游单模型的模型中转测试站。

**Architecture:** New API 单体容器同时提供 Web 界面和 OpenAI 兼容 API，Render 运行容器，Supabase Session Pooler 提供持久化 PostgreSQL。阿里百炼是唯一上游；Netlify 只部署无后端逻辑的静态人工充值页。

**Tech Stack:** New API `v1.0.0-rc.49`、Docker、Render Blueprint、Supabase PostgreSQL、阿里百炼 OpenAI 兼容接口、静态 HTML、Netlify。

**Spec:** `docs/superpowers/specs/2026-11-08-public-model-relay-design.md`

## Global Constraints

- 第一版不新增托管或数据库费用；不选择任何付费资源，不承诺模型调用永久免费。
- 固定 New API 版本为 `v1.0.0-rc.49`；若安全审查要求换版，必须重新验证，禁止直接使用 `latest`。
- 新用户额度为零，不赠送额度，不生成无额度限制的默认令牌。
- 只配置用户指定的阿里百炼账户和一个模型；凭据只从 `D:\secure\ali-api-zy.csv` 读取所需字段。
- 真实密钥、数据库连接串、管理员密码和收款码不得进入 Git；聊天和命令输出不得复述这些值。
- Render、Supabase、Netlify 的创建、公开和付费操作均需用户授权；本计划只使用免费资源。
- 不配置 Redis，不使用现有个人数据库，不接入其他 API 账号，不接自动支付。
- 内容安全采用百炼上游审核与管理处置；公开前必须具备邮箱验证、人机验证、限流、封禁和令牌吊销能力。
- 私密测试通过前不公开传播地址，不发生真实付款。

## Review Focus

- 新用户或余额不足用户必须不能调用模型；Task 4 用零额度、余额不足和默认令牌检查固定该行为。
- 上游已计费但本地超时、流式响应缺少最终用量时必须能发现对账差异；Task 3 用非流式、流式和失败请求对照上游用量固定该行为。
- 数据库密码中的保留字符、IPv4 网络和 TLS 不能导致连接失败或明文回退；Task 2 用 Session Pooler、URL 编码和 `sslmode=require` 检查固定该行为。
- 重复付款截图或重复人工操作不能自动形成多次入账；Task 5 用支付平台真实到账与人工记录双重核对固定该行为。
- 免费实例休眠、重启和数据库暂停恢复后不得丢失余额或令牌；Task 6 用重启和恢复后的数据检查固定该行为。

---

### Task 1: 固定容器版本并完成本机冒烟验证

**Files:**
- Create: `Dockerfile`
- Create: `.dockerignore`
- Create: `.gitignore`
- Create: `.env.example`

**Interfaces:**
- Produces: Docker 镜像入口，容器内 New API 监听 `0.0.0.0:3000`。
- Produces: 后续 Render Blueprint 使用的变量名 `SQL_DSN`、`SESSION_SECRET`、`CRYPTO_SECRET`、`TZ`、`PORT`。

- [ ] **Step 1: 检查版本和安全公告**

  打开 `https://github.com/QuantumNous/new-api/releases/tag/v1.0.0-rc.49` 与项目 Security Advisories，确认没有针对该版本且尚未修复的已知高危问题。若必须换版，先更新设计与本计划中的版本号，再继续。

- [ ] **Step 2: 创建最小容器配置**

  `Dockerfile` 只以 `calciumion/new-api:v1.0.0-rc.49` 为基础镜像，不复制任何本地密钥。`.dockerignore` 排除 `.git`、`.env*`、密钥文件和收款码；`.gitignore` 排除 `.env`、本地数据库、日志及 `recharge/assets/*.png`。

- [ ] **Step 3: 创建无真实值的变量模板**

  `.env.example` 仅列出 `SQL_DSN`、`SESSION_SECRET`、`CRYPTO_SECRET`、`TZ=Asia/Shanghai`、`PORT=3000`，值使用明显占位符，不复制 Supabase 或百炼真实信息。

- [ ] **Step 4: 构建并检查镜像固定版本**

  Run: `docker build -t model-api-relay:local .`
  Expected: 构建成功；`Dockerfile` 和构建上下文不包含密钥或收款码。

- [ ] **Step 5: 用临时 SQLite 做本机启动测试**

  使用仅本次测试存在的随机 Secret 启动容器，绑定 `127.0.0.1:3100:3000`，不挂载真实数据，不配置上游。

  Run: `curl --fail http://127.0.0.1:3100/api/status`
  Expected: HTTP 成功且返回 New API 状态信息；容器日志无数据库或端口错误。

- [ ] **Step 6: 停止并删除临时容器**

  Run: `docker rm -f model-api-relay-test`
  Expected: 临时容器及其 SQLite 数据被删除，项目目录没有生成的数据库或日志。

- [ ] **Step 7: 提交配置**

  ```bash
  git add Dockerfile .dockerignore .gitignore .env.example
  git commit -m "chore: pin New API deployment image"
  ```

### Task 2: 部署 Render 与独立 Supabase 数据库

**Files:**
- Create: `render.yaml`

**Interfaces:**
- Consumes: Task 1 的 `Dockerfile` 和变量名。
- Produces: 私有测试用 Render HTTPS 地址，以及可通过 `SQL_DSN` 访问的独立 PostgreSQL 数据库。

- [ ] **Step 1: 创建 Render Blueprint**

  `render.yaml` 定义一个 Docker Web Service，`plan` 为 `free`，`healthCheckPath` 为 `/api/status`，设置 `TZ=Asia/Shanghai` 和 `PORT=3000`。`SQL_DSN` 使用 `sync: false`；`SESSION_SECRET` 与 `CRYPTO_SECRET` 使用平台生成值，不写固定秘密。

- [ ] **Step 2: 静态检查 Blueprint 和仓库安全**

  Run: `git diff --check`
  Expected: 无空白错误。

  Run: `git status --short`
  Expected: 只有计划内文件；没有 `.env`、CSV、数据库、日志或收款码。

- [ ] **Step 3: 新建独立 Supabase 免费项目**

  经用户授权后创建专用于本站的免费项目，选择离 Render 服务较近的可用区域，生成独立高强度数据库密码。不要修改或复用已有 Supabase 项目。

- [ ] **Step 4: 获取 Session Pooler 连接串**

  在 Supabase `Connect` 中选择 **Session pooler**，使用端口 `5432`，用户名采用 `postgres.[PROJECT-REF]`。将密码中的 URL 保留字符百分号编码，并添加 `sslmode=require`；不要手猜 Pooler 主机名。

- [ ] **Step 5: 部署 Render 服务并注入 Secret**

  经用户授权后，通过 GitHub Blueprint 部署。把 Session Pooler 连接串只写入 Render Secret `SQL_DSN`，确认没有选择付费实例、磁盘或数据库。

- [ ] **Step 6: 检查迁移与 HTTPS 状态**

  查看 Render 构建和运行日志，确认数据库表迁移完成且没有输出完整 `SQL_DSN`。

  Run: `curl --fail https://<render-test-host>/api/status`
  Expected: HTTPS 请求成功，日志无连接、TLS 或迁移错误。

- [ ] **Step 7: 初始化管理员并保持站点私有**

  创建独立 Root 管理员和强密码，不使用现有密码。只记录 Render 测试地址，不公开传播。

- [ ] **Step 8: 提交 Blueprint**

  ```bash
  git add render.yaml
  git commit -m "chore: add free Render deployment blueprint"
  ```

### Task 3: 接入一个百炼模型并核对计费

**Files:**
- 无文件修改；只操作 New API 管理后台和平台 Secret。

**Interfaces:**
- Consumes: Task 2 的管理后台；用户指定的 `D:\secure\ali-api-zy.csv`。
- Produces: 一个可供下游令牌调用的模型名，以及与百炼价格一致的计费配置。

- [ ] **Step 1: 最小化读取百炼配置**

  经用户确认后，只读取 CSV 中一个指定账户的 API Key、兼容接口地址和模型名字段；不输出完整 Key，不读取其他账户，不把 CSV 复制进项目。

- [ ] **Step 2: 配置唯一渠道**

  在 New API 创建 OpenAI 兼容渠道，使用 CSV 对应的百炼兼容地址，只启用一个选定模型。密钥只保存在服务端渠道配置中，其他渠道保持禁用。

- [ ] **Step 3: 验证渠道与模型目录**

  执行后台渠道测试，并确认普通用户模型目录只显示该模型。
  Expected: 渠道测试成功，不出现其他百炼模型或其他上游模型。

- [ ] **Step 4: 配置价格与调用限制**

  以百炼控制台当前输入、输出及缓存价格配置 New API 计费规则；设置令牌总额度、可用模型和适当速率限制。价格来源和配置时间记录在管理备注中，不硬编码到仓库。

- [ ] **Step 5: 给测试用户增加小额测试额度**

  不经过支付，直接为专用测试用户设置仅够数次调用的额度，创建仅限该模型的下游令牌。

- [ ] **Step 6: 验证非流式调用**

  向 `https://<render-test-host>/v1/chat/completions` 发送短提示，`stream` 为 `false`。
  Expected: 返回选定模型的有效答案；响应不包含上游 Key；New API 和百炼均产生一条可核对用量。

- [ ] **Step 7: 验证流式调用与用量**

  发送相同级别的短提示，`stream` 为 `true` 并请求用量信息。
  Expected: SSE 正常结束；New API 记录输入/输出 Token 和扣费，与百炼用量不存在无法解释的差异。

- [ ] **Step 8: 验证失败请求与余额边界**

  分别测试余额不足、错误模型和上游临时不可用的请求。
  Expected: 请求被明确拒绝或报错；检查失败请求是否产生上游费用，若产生则核对站内处理，禁止在未查明前开放充值。

### Task 4: 配置公开注册与防滥用

**Files:**
- 无文件修改；只操作 New API、SMTP 和 Cloudflare Turnstile 配置。

**Interfaces:**
- Consumes: Task 2 的 Render 主机名和 Task 3 的受额度保护模型。
- Produces: 可公开注册但零初始额度、可验证邮箱、可限流和可封禁的用户流程。

- [ ] **Step 1: 配置独立 SMTP**

  使用用户确认可用于本站的 SMTP 账号，不把授权码写入 Git。发送注册验证码到两个不同邮箱。
  Expected: 两个邮箱均收到验证码，过期或错误验证码不能通过。

- [ ] **Step 2: 配置 Turnstile**

  为 Render 测试主机名创建 Cloudflare Turnstile 免费站点，将 Site Key 和 Secret 分别写入对应安全位置，启用注册人机验证。
  Expected: 正常浏览器可完成验证；跳过验证不能注册。

- [ ] **Step 3: 固定零额度注册策略**

  关闭注册赠送、邀请奖励和默认无限令牌，确认 `GENERATE_DEFAULT_TOKEN` 不用于绕过额度检查。
  Expected: 新注册用户余额为零，未充值调用返回额度不足。

- [ ] **Step 4: 设置基础限流**

  保留 `GLOBAL_API_RATE_LIMIT_ENABLE`、`GLOBAL_WEB_RATE_LIMIT_ENABLE` 和 `CRITICAL_RATE_LIMIT_ENABLE` 开启，在后台设置适合单免费实例的保守限制。
  Expected: 连续超额请求收到限流响应，正常低频注册和调用不受影响。

- [ ] **Step 5: 验证管理处置**

  对测试用户执行令牌吊销和用户封禁，再分别尝试调用与登录。
  Expected: 被吊销令牌立即或在文档规定的缓存窗口后失效；被封禁用户不能继续调用。

- [ ] **Step 6: 检查日志与敏感信息**

  检查 Web、API 和错误日志设置。
  Expected: 默认不记录完整提示词、响应正文、上游 Key 或 `Authorization` 头；普通用户不能查看管理日志。

### Task 5: 发布静态人工充值页并接入钱包

**Files:**
- Create: `recharge/index.html`
- Create: `netlify.toml`
- Modify: `.gitignore`

**Interfaces:**
- Consumes: 用户提供的站点名称、管理员联系方式、充值规则及专用收款码图片。
- Produces: 一个公开静态充值说明 URL，供 New API 的“充值链接”使用；不产生支付订单或自动调额接口。

- [ ] **Step 1: 编写无脚本静态说明页**

  `recharge/index.html` 展示微信、支付宝收款码、金额与额度规则、人工到账说明、站点账号填写提示和管理员联系方式。页面明确“付款不等于自动到账”，不收集密码或 API Key，不包含第三方支付脚本。

- [ ] **Step 2: 配置 Netlify 静态发布**

  `netlify.toml` 将发布目录设为 `recharge`。收款码放在 `recharge/assets/wechat.png` 和 `recharge/assets/alipay.png`，这两个 PNG 继续由 `.gitignore` 排除；只使用用户专门确认可公开的收款码。

- [ ] **Step 3: 本地检查页面和资源**

  使用 Python 标准库解析 HTML，并检查两个图片资源存在、页面没有外部脚本和表单提交地址。
  Expected: HTML 可解析，本地打开能显示两种收款码及完整人工充值说明。

- [ ] **Step 4: 部署 Netlify 免费站点**

  经用户授权后部署 `recharge` 目录到免费 Netlify 站点。使用 CLI 时仅执行交互式登录，不把访问令牌写进项目；也可以使用 Netlify 网页手动部署。

- [ ] **Step 5: 配置 New API 充值链接**

  将 Netlify HTTPS 地址填入 New API 的充值链接，关闭在线支付网关。
  Expected: 用户钱包能打开静态说明页，没有自动支付入口。

- [ ] **Step 6: 演练人工充值且不发生真实付款**

  使用一笔明确标记的模拟记录，依次演练“定位站点账号—后台调额—记录操作—用户确认余额”，并核对真实运营时必须先在支付平台确认到账、不能仅凭截图。重复处理同一模拟记录时必须被人工记录识别，不重复入账；本步骤不发生真实付款。

### Task 6: 端到端验收、备份恢复与公开门槛

**Files:**
- 无文件修改；只执行验证和平台配置。

**Interfaces:**
- Consumes: Tasks 1–5 完成的应用、数据库、模型、注册和人工充值流程。
- Produces: 可判断是否允许公开测试的验收结果；任何关键项失败都保持私有。

- [ ] **Step 1: 从零执行用户流程**

  用新邮箱注册、完成验证、确认零余额、模拟充值、创建受限令牌并完成普通和流式调用。
  Expected: 全流程无需管理员修改数据库，余额和调用日志在用户页面可见且一致。

- [ ] **Step 2: 验证拒绝与越权行为**

  测试错误令牌、已吊销令牌、余额不足、未开放模型、普通用户访问管理接口和连续超限请求。
  Expected: 全部被正确拒绝或限流，不返回上游凭据或其他用户数据。

- [ ] **Step 3: 验证重启持久化**

  记录测试用户余额、令牌标识和日志数量，重启 Render 服务后重新检查。
  Expected: 用户、余额、令牌和日志保持；应用可从 Supabase 正常恢复。

- [ ] **Step 4: 验证数据库备份与恢复路径**

  按 Supabase 当前免费功能导出本站数据库，并在独立临时数据库执行恢复验证；恢复完成即删除临时数据。若免费网络或工具限制导致无法备份恢复，保持站点私有且不接受真实付款。

- [ ] **Step 5: 验证休眠与 Supabase 暂停恢复**

  让免费服务自然休眠后重新访问；如 Supabase 项目暂停，则按官方 Resume 流程恢复后再测调用。
  Expected: 唤醒后数据不丢失；页面明确接受冷启动延迟，不把休眠当作数据故障。

- [ ] **Step 6: 实测中国大陆访问**

  至少从两个不同的大陆网络打开 Web 页面并发起一次 API 调用，记录成功率、首字延迟和冷启动表现。
  Expected: 两个网络均能完成流程；否则不公开推广，先重新选择托管方案。

- [ ] **Step 7: 核对公开运营前提**

  确认百炼条款允许当前使用方式，上游内容安全已启用，投诉与封禁流程可执行，充值规则和管理员联系方式准确。
  Expected: 所有前提均有明确结果；任何一项不满足都保持私有测试。

- [ ] **Step 8: 最终安全检查**

  Run: `git status --short`
  Run: `git diff --check`
  Expected: 没有未说明的生成文件、密钥、数据库、日志或收款码；最终变更与设计一致。

- [ ] **Step 9: 由用户决定是否公开**

  只在用户明确确认后公开地址或接受真实付款。免费资源不足、访问不稳定或计费对账失败时，不通过购买资源或添加其他个人 Key 擅自继续。
