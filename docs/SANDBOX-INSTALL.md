# O2 沙箱检索、爬虫与 Agent 安装验收说明

本文档供安装人员、Codex、Claude Code 等自动化编码 Agent 使用。目标是在同一个 O2 沙箱中部署 `heuristics-search` 检索服务、仓库内置的 Trace Base 爬虫能力和 `agentic_heuristic_search` Agent，并在完整验收后输出用户可复制的公网试用命令。爬虫位于 `heuristics-search/tools/trace-base-crawler`，是检索仓库内的一次性任务能力，不额外监听公网端口。

## 1. 安装前必须确认

应用安装阶段唯一必须由用户提供的配置是 `LLM_API_KEY`。模型地址、模型名称和本地检索地址均已有默认值，不需要询问用户。

如果用户没有提供 `LLM_API_KEY`，必须停止并询问：

> 请提供模型 API Key。Key 只会写入沙箱内权限为 600 的 `.env`，不会写入代码、README、日志或 Git。请提供原始 token，不要带 `Bearer ` 前缀。

获得 Key 之前，不得执行依赖安装、服务启动或 API 验证。不得把真实 Key 写入 Git。

还需确认沙箱能访问两个 Git 仓库；如果 `git clone` 鉴权失败，应向用户申请仓库读取权限，不得把个人 Git 凭证写入仓库。

## 2. 克隆两个功能分支

```bash
export PROJECT_ROOT=/home/sandbox/oxygen
mkdir -p "${PROJECT_ROOT}"
cd "${PROJECT_ROOT}"

git clone --branch feat/sandbox-local-search-stack --single-branch \
  git@coding.jd.com:oxygent-java/heuristics-search.git \
  heuristics-search

git clone --branch feat/sandbox-local-search-stack --single-branch \
  git@coding.jd.com:oxygent-java/agentic_heuristic_search.git \
  agentic_heuristic_search
```

最终目录：

```text
/home/sandbox/oxygen/heuristics-search
/home/sandbox/oxygen/agentic_heuristic_search
```

## 3. 默认小数据模式

默认使用 `heuristics-search/deploy/sandbox-sample/search-materials` 中的脱敏技术验证数据：

- 仅包含少量人工构造的京麦页面；
- 不包含生产数据或用户隐私；
- 不访问 OSS，不需要 OSS 凭证；
- 仅验证安装、索引、检索、Agent 调用和公网 API 链路；
- 样例中的比例、路径等内容不得作为真实业务依据。

检索服务监听 `127.0.0.1:8010`，只供同一沙箱中的 Agent 使用。Agent 监听 `0.0.0.0:8020`，通过 O2 端口代理对外访问。Trace Base 爬虫验收只访问沙箱内临时启动的脱敏静态页，不访问 OSW、不需要登录态，也不长期运行。

## 4. 配置 Agent

```bash
cd /home/sandbox/oxygen/agentic_heuristic_search
cp .env.sandbox.example .env
chmod 600 .env
```

将用户提供的原始 token 写入 `LLM_API_KEY`，不要带 `Bearer ` 前缀。其余配置保持模板默认值：

```dotenv
LLM_API_KEY=<用户提供的原始 token>
```

## 5. 安装两个服务

```bash
cd /home/sandbox/oxygen/agentic_heuristic_search
./bin/install_sandbox_stack.sh
```

脚本按以下顺序执行：

1. 再次检查 `LLM_API_KEY`，缺失时立即终止；
2. 安装 `heuristics-search` 的 Python 3.11 环境和 server 依赖；
3. 校验 Node.js 22+，安装 Trace Base 爬虫的 npm 依赖和项目内 Playwright Chromium；
4. 安装 Agent 的 Python 3.11 环境和最小运行依赖；
5. 两个仓库分别使用项目内 `.venv`、`.uv-cache` 和 `.uv-python`，浏览器使用 `heuristics-search/.playwright-browsers`，不依赖不可写的用户缓存目录。

安装器默认使用已在 O2 沙箱验证可用的 Python 镜像。若所在环境要求使用其他包源，在安装前覆盖即可：

```bash
export UV_DEFAULT_INDEX=https://your-pypi-mirror.example/simple
./bin/install_sandbox_stack.sh
```

沙箱必须预装 Node.js 22+、npm 和 `rg`。其中 `rg` 用于规则检索，Node.js 和 npm 用于 Trace Base 爬虫；缺失时安装器会明确终止。

## 6. 启动两个服务

```bash
cd /home/sandbox/oxygen/agentic_heuristic_search
./bin/start_sandbox_stack.sh
```

脚本先启动并检查本地检索服务，再启动 Agent。如果 Agent 启动失败，脚本会停止刚启动的检索服务，避免留下半套服务。

可分别检查：

```bash
curl -fsS http://127.0.0.1:8010/health
curl -fsS http://127.0.0.1:8020/health
```

日志和 PID：

```text
/home/sandbox/oxygen/heuristics-search/logs/server.log
/home/sandbox/oxygen/heuristics-search/server.pid
/home/sandbox/oxygen/agentic_heuristic_search/logs/server.log
/home/sandbox/oxygen/agentic_heuristic_search/server.pid
```

## 7. 完整验收

```bash
cd /home/sandbox/oxygen/agentic_heuristic_search
./bin/verify_sandbox_stack.sh --base-url http://127.0.0.1:8020
```

验证顺序：

1. 检索服务 `GET /health`；
2. 检索服务 `POST /search`，确认小数据返回非空结果；
3. Trace Base 爬虫抓取沙箱内两页脱敏静态站点；
4. 确认两页 HTML、Markdown、JSON 证据均落盘，并成功生成 `operation_tree.json` 和每页 `summary`；
5. Agent `GET /health`；
6. Agent `POST /generate/answer`；
7. Agent `POST /generate`；
8. Agent A2A `SendMessage`；
9. Agent A2A `SendStreamingMessage`，并检查完成事件。

爬虫真实运行仍使用检索仓库内的一键脚本，例如：

```bash
cd /home/sandbox/oxygen/heuristics-search
START_URLS=https://osw.jd.com/home \
  tools/trace-base-crawler/run_osw_crawler.sh --background
tools/trace-base-crawler/run_osw_crawler.sh --status
```

标准安装验收只要求上述两页脱敏 smoke 通过，证明浏览器启动、页面抓取、证据落盘、操作树和 summary 链路可用；不要求额外抓取真实业务站点。JoySpace 等受 SSO 保护的页面需单独处理：通过沙箱 noVNC 唤起 Chromium，由用户完成登录并将状态保存在 `.trace-base-crawler/browser-profile`，关闭交互浏览器后再让爬虫复用同一 Profile。若页面仍跳转 `authme.jd.com`，不得把统一登录页作为业务 Markdown 交付。

本地运行只算第一阶段预检，不能作为交付结论。以上安装、启动和完整验收必须在 O2 sandbox 内再次执行；沙箱内验证全部成功后，Codex 必须使用当前沙箱平台提供的端口预览能力识别 Agent `8020` 的公网地址，并从沙箱外再次验证 `GET /health` 和 `POST /generate/answer`。不要向用户索取沙箱管理凭证，也不要把此类凭证写入项目文件。

外部验证成功后，Codex 再输出用户可复制的试用请求。

## 8. 用户试用请求

将 `<PUBLIC_BASE_URL>` 替换为 Codex 识别并验证通过的公网地址：

```bash
curl -X POST '<PUBLIC_BASE_URL>/generate/answer' \
  -H 'Content-Type: application/json' \
  -d '{"message":"京麦保证金比例是多少"}'
```

## 9. 停止两个服务

需要停止时执行统一脚本：

```bash
cd /home/sandbox/oxygen/agentic_heuristic_search
./bin/stop_sandbox_stack.sh
```

脚本先停止 Agent，再停止检索服务。两个停止动作都支持重复执行，并会校验 PID 对应的进程，避免误杀。

也可以分别停止：

```bash
cd /home/sandbox/oxygen/agentic_heuristic_search
./bin/stop_sandbox.sh

cd /home/sandbox/oxygen/heuristics-search
./bin/stop_sandbox.sh
```

## 10. 交付标准

完成部署必须同时满足：

- 两个仓库均从远程 `feat/sandbox-local-search-stack` 分支克隆；
- 两个服务均使用 Python 3.11；
- `heuristics-search` 使用 Node.js 22+，Playwright Chromium 安装在项目目录；
- 检索服务 `/health` 和 `/search` 通过；
- Trace Base 爬虫两页脱敏 smoke 验收通过，证据、操作树和 summary 完整；
- Agent 能正常调用同一沙箱中的检索服务；
- Agent HTTP 与 A2A 全部验收通过；
- 已识别并验证 8020 公网地址，并提供用户试用 `curl`；
- 回复、Git、日志和交付文件中不包含任何 API Key。
