# srcradar-mcp-skill —— srcradar-mcp-server 的客户端约定

> 给 agent 看的 skill:从 Hermes(或任何 streamable-http MCP 客户端)调 srcradar-mcp-server 的 15 个工具时,什么是对的、什么是错的、什么会被 daemon 拒。

<p align="left">
  <a href="https://github.com/usdagfhjkda/srcradar-mcp-skill/blob/main/LICENSE"><img src="https://img.shields.io/badge/license-Apache--2.0-blue.svg" alt="License: Apache-2.0"></a>
  <a href="https://github.com/usdagfhjkda/srcradar-mcp-skill"><img src="https://img.shields.io/badge/repo-srcradar--mcp--skill-181717.svg" alt="Repo"></a>
  <img src="https://img.shields.io/badge/role-client--side--conventions-9cf.svg" alt="Role: client-side conventions">
  <img src="https://img.shields.io/badge/peer-srcradar--mcp--server-success.svg" alt="peer: srcradar-mcp-server">
  <img src="https://img.shields.io/badge/parent-srcradar-success.svg" alt="parent: srcradar">
</p>

<br>

**一句话**:`srcradar-mcp-server` 是 daemon(服务端,绑 loopback 8764,转 srcradar 子命令);**`srcradar-mcp-skill` 是这个 skill**(客户端侧,教 agent 怎么正确调它)。

agent 经常踩两类错:

1. **参数形态错** —— `args=[{"item": "--help"}]` 这种 LLM 经常误生成的 array-of-object 形态,被 daemon 的 schema 校验当场拒
2. **路径形态错** —— 把客户端 Windows 路径 `C:\...\seed.tsv` 直接塞 `-s`,daemon 在 deploy host 看不到这个路径,`subprocess exit 1: file not found`

这个 skill 把这两类错(以及它们的修法)写成 agent 一眼能查的清单。

---

## 使用前提与合规

本 skill 是 `srcradar-mcp-server` 的客户端规约文档,不替代任何上游条款。完整使用前提、免责声明、上游致谢以主仓为准:

- 主仓条款:[`srcradar/LICENSE`](https://github.com/usdagfhjkda/srcradar/blob/main/LICENSE)(Apache-2.0)
- 附加使用限制与免责声明:[`srcradar/TERMS_ADDENDUM.md`](https://github.com/usdagfhjkda/srcradar/blob/main/TERMS_ADDENDUM.md)
- 上游致谢:[`srcradar/NOTICE`](https://github.com/usdagfhjkda/srcradar/blob/main/NOTICE)

skill 内提及的所有 15 个工具、协议头、端口、字段名,都是 [`srcradar-mcp-server`](https://github.com/usdagfhjkda/srcradar-mcp-server) 的公开契约;daemon 自身的 README 在那边,本仓只讲客户端怎么调。

本 skill **仅供已获合法书面授权的场景**(SRC 协议 / 渗透测试授权 / 自有资产白名单)使用;skill 本身不参与、不背书、不知情任何具体使用场景。

---

## 是什么

三条契约,违反任一条都按错误处理:

- **协议契约**:客户端通过 `streamable-http` 单 endpoint `POST http://127.0.0.1:8764/mcp` 发送 JSON-RPC 2.0,带五项头:`Content-Type: application/json` / `Accept: application/json` / `MCP-Protocol-Version: 2026-07-28` / `Mcp-Method: tools.invoke` / `Mcp-Name: <tool-name>`;响应是 SSE 或纯 JSON(由 Accept 协商)
- **传输契约**:daemon **只绑 loopback**,远程调用必须先 SSH 隧道(`ssh -L 8764:127.0.0.1:8764 <deploy-host>`);客户端永远把 daemon 当 `127.0.0.1:8764` 看待
- **参数契约**:**永远不要**把客户端绝对路径塞 `-s` / `-i`(daemon 看不见客户端 fs);**永远不要**传 `args` array-of-object / array-of-string(daemon 的 JSON Schema 拒收);写路径类参数前先 `stage_file` / `stage_dir` 落盘到 daemon 侧,再传 `staged_path` / `staged_dir`

**主从关系**:服务端是 `srcradar-mcp-server`(daemon,绑 loopback),客户端是本 skill 描述的调用形态;**两者是同一个 MCP 协议的两端**。

---

## 快速开始

**客户端侧没有 install** —— 这是一个 skill 文档仓,不需要 `git clone` 后跑 `./install.sh`。Agent 在自己运行的 LLM 上下文里加载 `SKILL.md`(或等价的 YAML frontmatter + Markdown body),按里面的约束调 15 个工具即可。

```text
1. 确认 daemon 在 deploy host 的 loopback 上跑着(ssh <deploy-host> 'curl -s http://127.0.0.1:8764/health')
2. 在客户端机上开 SSH 隧道(ssh -L 8764:127.0.0.1:8764 <deploy-host>)
3. agent 加载本 skill(SKILL.md)
4. agent 按 §2 参数形状调 db.read_* / manage.* / daily.* / dispatcher.* / ymicp.*
```

---

## 架构

```
┌──────────────────┐         ┌────────────────────────────────────────────┐
│  MCP client      │         │  daemon host (loopback 127.0.0.1)          │
│  (agent / IDE /  │         │                                            │
│   harness / LLM  │         │  ┌──────────────────────────┐              │
│  + SKILL.md      │         │  │  daemon.py (streamable-   │              │
│  (本 skill)      │  POST   │  │   http :8764, stdlib)     │              │
│                  │ ──────► │  │                          │              │
│                  │ /mcp    │  │  GET  /health  ── 200 OK │              │
│                  │         │  │  POST /mcp     ──┐       │              │
│                  │ ◄────── │  └─────────────────┼───────┘              │
│  (SSH -L 8764    │  SSE /  │                    │ whitelist.json       │
│   <deploy-host>) │  JSON   │                    ▼                      │
└──────────────────┘         │  ┌──────────────────────────────────────┐   │
                             │  │ _stage_file / _stage_dir (e1-confirmed)│  │
                             │  │   ↳ 落 ~/.cache/srcradar-mcp/staged/  │   │
                             │  └──────────────────────────────────────┘   │
                             │                    │                      │
                             │                    ▼                      │
                             │  ┌──────────────────────────────────────┐   │
                             │  │ db/*.py  (auth: none, 只读)          │   │
                             │  │  ↳ 直查 srcradar 主仓 SQLite          │   │
                             │  └──────────────────────────────────────┘   │
                             │                    │                      │
                             │                    ▼                      │
                             │  ┌──────────────────────────────────────┐   │
                             │  │ ./srcradar <subcmd>  (主仓 dispatcher) │   │
                             │  │   ↳ manage.* / daily.* / ymicp.*     │   │
                             │  │   ↳ dispatcher.list / run_confirmed   │   │
                             │  └──────────────────────────────────────┘   │
                             │                    │                      │
                             │                    ▼                      │
                             │  ┌──────────────────────────────────────┐   │
                             │  │ srcradar 主仓 SQLite (只读 / 写)    │   │
                             │  │   db/recon.sqlite3 (WAL)             │   │
                             │  └──────────────────────────────────────┘   │
                             └────────────────────────────────────────────┘
```

### 客户端侧的不变量(踩过就炸)

- **永远不要把客户端 fs 路径直接塞 `-s` / `-i`** —— daemon 在 deploy host,看不见 Windows / 本机 fs。**先 `stage_file` / `stage_dir` 落盘到 daemon 侧,再传 `staged_path` / `staged_dir`**
- **永远不要传 `args` 数组形态** —— `args=["--help"]` / `args=[{"item": "--help"}]` 都会被 daemon 的 JSON Schema 拒收。**用 `args={"help": True}`(dict boolean flag)或 `args="--help"`(单字符串)**
- **永远不要绕过 MCP 直跑 srcradar CLI / curl daemon** —— 跳过所有审批 + 审计,且会撞 daemon schema;绕开审批的**唯一例外**是看 `--help` / 调试性查询,走 `terminal(ssh <deploy-host> '<repo>/srcradar <subcmd> --help')` 直跑(详见 §6)
- **agent 是调 srcradar-mcp-server 的客户端,不是它的 owner** —— daemon / operator 的配置决策不归 agent 管;`trust: untrusted` 没设、daemon 没重启、daemon 拒收调用 —— 这些告诉 user,不是 agent 自己 debug

---

## 工具清单(客户端视角)

15 个工具,按调用形态分类。**实际数量以服务端 [`srcradar-mcp-server/whitelist.json`](https://github.com/usdagfhjkda/srcradar-mcp-server/blob/main/whitelist.json) 为准**。

| 工具 | 客户端调用形态 | 备注 |
|---|---|---|
| `db.read_business_summary` | `args={business: "Acme"}` | 严格按 schema 字段名,**不是** `--business` argv |
| `db.read_subdomains` | `args={business: "Acme", limit: 10, offset: 0, since: "2026-09-01"}` | 同上 |
| `db.read_open_ports` | `args={business: "Acme", ...}` | 同上 |
| `db.read_companies` | `args={business: "Acme", ...}` | 同上 |
| `db.read_diff` | `args={run_id: "..."}` 或 `{business: "...", since: "..."}` | 同上 |
| `db.read_single_subdomain` | `args={business: "Acme", subdomain: "x.example.com"}` | 同上 |
| `dispatcher.list` | `args={"list": True}` 或 `args="--list"` | 不要传 `args=["--list"]` |
| `ymicp.icp_mapp_query` | `args={keyword: "..."}` | stdin 走管道 |
| `stage_file` | `args={filename, base64, file_path?}` | **3 种输入模式三选一**(详见 §stage_file) |
| `stage_dir` | `args={base64}` 或 `args={entries: [...]}` | tar.gz base64 或 entries list |
| `dispatcher.run_confirmed` | `args="<argv>"` 或 `args={...}` | 通用跑 srcradar CLI,慎用 |
| `manage.add_business` | `args={"-n": "Acme", "-s": staged_path}` | `-s` 必须是 `stage_file` 产出的 `staged_path` |
| `manage.set_config` | `args={"-n": "Acme", "--web": "0"}` | 改 enabled / web / tcp / icp |
| `daily.run_one_business` | `args={"-n": "Acme"}` | pdtm / enscan / icp / url stages |
| `daily.run_dashboard_watchdog` | `args={}` 或空 | 重启 dashboard(可选用) |

**read / write 比** = 8 读 + 7 写(其中 2 个 stage 原语 + 5 个 `./srcradar` 写子命令)。**read 工具静默调用;write 工具触发 Hermes 原生审批弹窗**(用户点 yes 才执行)。

---

## 客户端参数形状(踩过的坑)

### `db.read_*` —— 严格按 schema 字段名

**对**:

```python
mcp_srcradar_db_read_subdomains(
    business="Acme",
    limit=10,
    offset=0,
    since="2026-09-01",
)
```

**错**(daemon 校验拒收):

```python
mcp_srcradar_db_read_subdomains(args=["--business", "Acme"])      # 错形态:argv 数组
mcp_srcradar_db_read_subdomains({"business": "Acme"})              # 错形态:包了一层
mcp_srcradar_db_read_subdomains(business_name="Acme")              # 错形态:字段名错
```

### `manage.*` / `daily.*` / `dispatcher.*` / `ymicp.*` —— `args: dict`

**调用方把 argv 平铺到 dict 里**:

```python
# 看 help(注意:仍然走 e1-confirmed 弹窗,daemon 不区分 --help)
mcp_srcradar_manage_add_business(args={"help": True})              # ✓ boolean flag
mcp_srcradar_manage_add_business(args="--help")                    # ✓ 单字符串

# 真要加企业
mcp_srcradar_manage_add_business(
    args={
        "-n": "Acme",
        "-s": "<staged-root>/<upload_id>.seed.tsv",                # 来自 stage_file
    },
    _staging_ref="<upload_id>",                                    # 调用结束 daemon 自动清
)
```

**永远不要**:

```python
mcp_srcradar_manage_add_business(args=["--help"])                  # ❌ array of string
mcp_srcradar_manage_add_business(args=[{"item": "--help"}])        # ❌ array of object(LLM 经常误生成)
mcp_srcradar_manage_add_business(args="--help", name="manage.add_business")  # ❌ params 多了 name
```

> **绕开 `--help` 弹窗**:优先走 SSH + 本地 terminal 工具直跑
> `terminal(ssh <deploy-host> '<repo>/srcradar manage add_business --help')`。
> 不走 MCP = 不弹窗 = 不撞 schema = 调试最快路径。

### `stage_file` —— 三种输入模式(三选一)

| 字段 | 形态 | 何时用 |
|---|---|---|
| `stdin` | inline string | LLM 现场生成的小内容 |
| `base64` | base64 编码 | 客户端读文件 → 转码 → 上传(**最常用**) |
| `file_path` + `base64` | 客户端路径 + base64 字节 | 同 `base64`,额外声明本机路径(daemon 仅记录,不访问) |

**约束**(全部硬性):

- `file_path` 必须配 `base64`,**单独 `file_path` 一律拒收**
- `filename` 走严格白名单 `[A-Za-z0-9._-]+`,最长 255,无 `..` 无路径分隔符
- payload ≤ 100 MiB(STAGE_MAX_BYTES)
- daemon 不访问客户端 fs,只收字节

**客户端调用模板**(Python):

```python
import base64, json, urllib.request

payload = open("/local/path/seed.tsv", "rb").read()
body = {
    "jsonrpc": "2.0", "id": 1, "method": "tools.invoke",
    "params": {
        "name": "stage_file",
        "args": {
            "filename": "seed.tsv",
            "base64": base64.b64encode(payload).decode(),
            "file_path": "/local/path/seed.tsv",   # 审计字段
        },
    },
}
resp = urllib.request.urlopen(urllib.request.Request(
    "http://127.0.0.1:8764/mcp",
    data=json.dumps(body).encode(),
    headers={
        "Content-Type": "application/json",
        "Accept": "application/json",
        "MCP-Protocol-Version": "2026-07-28",
        "Mcp-Method": "tools.invoke",
        "Mcp-Name": "stage_file",
    },
)).read()

sc = json.loads(resp)["result"]["structuredContent"]
upload_id   = sc["upload_id"]     # 16-hex
staged_path = sc["staged_path"]   # <staged-root>/<id>.seed.tsv
```

**调下游工具**时,把 `staged_path` 当普通路径塞 `args`,顺手加 `_staging_ref: <upload_id>` 让 daemon 在调用结束后清掉临时文件:

```python
mcp_srcradar_manage_add_business(
    args={"-n": "Acme", "-s": staged_path},
    _staging_ref=upload_id,   # daemon 在 finally 删
)
```

### `stage_dir` —— 目录形态

```python
# 方式 1:客户端打 tar.gz
import tarfile, io
buf = io.BytesIO()
with tarfile.open(fileobj=buf, mode="w:gz") as tar:
    tar.add("/local/scopedir/target.txt", arcname="target.txt")
    tar.add("/local/scopedir/exclude.txt", arcname="exclude.txt")
b64 = base64.b64encode(buf.getvalue()).decode()

mcp_srcradar_stage_dir(base64=b64)
# 拿到 staged_dir = <staged-root>/<id>/

# 方式 2:显式 entries list(平铺,无子目录)
mcp_srcradar_stage_dir(entries=[
    {"filename": "target.txt", "content_b64": "..."},
    {"filename": "exclude.txt", "content_b64": "..."},
])
```

下游工具:

```python
mcp_srcradar_manage_add_business(
    args={"-n": "Acme", "-i": staged_dir},
    _staging_ref=upload_id,
)
```

---

## 临时文件清理(客户端视角)

agent 唯一要做的:**任何从 `stage_file` / `stage_dir` 拿到 `upload_id` 的下游调用,都顺手带上 `_staging_ref: <upload_id>` 字段**。daemon 在下游工具调用返回时自动清掉临时文件,不需要 agent 二次操心。

```python
mcp_srcradar_manage_add_business(
    args={"-n": "Acme", "-s": staged_path},
    _staging_ref=upload_id,   # ← 这一行永远带上
)
```

`_staging_ref` 是 16-hex 字符串。daemon 对它做 `^[0-9a-f]{16}$` 严格校验,agent 不会因为传错格式炸掉。

agent **不**需要关心 24h ttl、启动 sweep、手工清理 —— 那是 daemon / operator 的事。

---

## 实战场景速查

| 场景 | 客户端调用链 |
|---|---|
| 看某业务子域列表 | `db.read_subdomains(business="X")` |
| 看 `--help` | 走 `terminal(ssh <deploy-host> '<repo>/srcradar manage add_business --help')`,**不走 MCP** |
| 加新业务(纯 db 行,无 seed / scope) | `manage.add_business(args={"-n":"Acme"})` |
| 加新业务 + seed TSV | `stage_file` → `manage.add_business(args={"-n":"Acme","-s":staged}, _staging_ref=id)` |
| 加新业务 + scope 目录 | `stage_dir` → `manage.add_business(args={"-n":"Acme","-i":staged}, _staging_ref=id)` |
| 改某业务 config | `manage.set_config(args={"-n":"Acme","--web":"0"})` |
| 跑某业务一轮 recon | `daily.run_one_business(args={"-n":"Acme"})` |
| 调任意 srcradar 子命令 | `dispatcher.run_confirmed(args="--list")` 或 `args={"list":True}` |
| **绝不走 MCP** | 任何看 `--help` / 调试性查询 → 用 `terminal(ssh <deploy-host> '...')` 直跑(详见 §6 唯一例外) |

---

## 调试清单(客户端踩到的错误)

| 症状 | 原因 | 修法 |
|---|---|---|
| `arguments.args[0] (type): {'item': '--help'} is not of type 'string'` | LLM 把 argv 包成 `[{item: "--help"}]` 撞 daemon schema | 改用 `args={"help": True}` 或 `args="--help"` |
| `subprocess exit 1: [-n <业务名>] is required` | 忘了 `-n` 或拼错 flag 名 | 走 `terminal(ssh ...)` 跑 `--help` 先看 flag 表 |
| `subprocess exit 1: [-s] seed file not found: /local/path/...` | 直接传了客户端路径给 `-s` | 先 `stage_file` 再传 `staged_path` |
| `filename must match [A-Za-z0-9._-]+` | filename 含 `/` `\` `..` 或 unicode | 重命名为 `seed.tsv` 这种纯 ASCII |
| `file_path MUST be paired with base64` | 只传了 `file_path` 不带 `base64` | 加 base64 字节 |
| `one of stdin / base64 / file_path+base64 required` | 三个模式都没传 | 至少传一个 |
| `payload too large: N > 104857600` | 单文件 > 100 MiB | 拆小 / 走 tar.gz |
| 连接被 daemon reset / 401 / 403 | SSH 隧道断了 / 头校验失败 / Origin 不在白名单 | 重连隧道;核对五项头;`Origin ∈ {null, 127.0.0.1, ::1, localhost}` |

> **弹窗不出现 / daemon 缺工具 / daemon 拒收调用** —— 这些是 operator 配置问题(`trust: untrusted` 没设 / daemon 没重启),**告诉 user** 而不是 agent 自己 debug。

---

## 🚫 禁止绕过 MCP 直接调 srcradar CLI / curl daemon

**这是硬性规则,agent 永远不要**:

- ❌ 用 `terminal` 工具 `ssh <deploy-host> '<repo>/srcradar manage add_business ...'` 直跑
- ❌ 用 `terminal` 工具 `ssh <deploy-host> 'curl -X POST http://127.0.0.1:8764/mcp ...'` 直接打 daemon
- ❌ 用 `browser` / `web_extract` 访问 daemon 端点
- ❌ 任何不经过 MCP 协议层、直接驱动 srcradar 业务逻辑的路径

**唯一例外**:**看 `--help` 或调试性查询**可以走 `terminal(ssh <deploy-host> '<repo>/srcradar <subcmd> --help')` —— 这是只读、不动 daemon / 业务状态、也不撞 schema。

---

## 约束(客户端必须知道)

- **永远把 daemon 当 loopback**。`127.0.0.1:8764` 是唯一调用地址;远程一律走 SSH 隧道 `ssh -L 8764:127.0.0.1:8764 <deploy-host>`
- **永远不要传 argv 数组形态**。`args=["..."]` / `args=[{...}]` 都会被 daemon 的 JSON Schema 拒收
- **永远不要把客户端 fs 路径直接塞 `-s` / `-i`**。daemon 在 deploy host,看不见客户端 fs;先 `stage_file` / `stage_dir`
- **Origin 头三件套**:`Content-Type` / `Accept` / `MCP-Protocol-Version` / `Mcp-Method` / `Mcp-Name` 五项是 daemon 入口必校验,任何一项错都直接拒
- **不绕过 MCP 直跑 daemon 业务逻辑**。审批 + 审计由 MCP 层兜,跳过就丢两条

---

## 上游致谢与 License

本 skill 是纯客户端规约文档,不包含 daemon / srcradar / 上游工具的实现代码。所有能力由下列上游支撑:

- **服务端** — [`usdagfhjkda/srcradar-mcp-server`](https://github.com/usdagfhjkda/srcradar-mcp-server),本 skill 描述的 15 个工具的 daemon 实现
- **srcradar 主仓** — [`usdagfhjkda/srcradar`](https://github.com/usdagfhjkda/srcradar),所有业务数据由主仓持有;daemon 是它的 MCP 适配层
- **上游致谢** — 见 [`srcradar/NOTICE`](https://github.com/usdagfhjkda/srcradar/blob/main/NOTICE),ENScan_GO / pdtm / dnsx / httpx / naabu / subfinder / alterx / cdnmatch 等

本 skill 与 daemon / 主仓同 License(Apache-2.0),条款以 [`srcradar/LICENSE`](https://github.com/usdagfhjkda/srcradar/blob/main/LICENSE) 为准;附加使用限制与免责声明见 [`srcradar/TERMS_ADDENDUM.md`](https://github.com/usdagfhjkda/srcradar/blob/main/TERMS_ADDENDUM.md);上游致谢见 [`srcradar/NOTICE`](https://github.com/usdagfhjkda/srcradar/blob/main/NOTICE)。

---

## License

srcradar-mcp-skill 以 Apache License 2.0 分发,完整条款见 [`LICENSE`](https://github.com/usdagfhjkda/srcradar-mcp-skill/blob/main/LICENSE)。

服务端、上游项目与各自 License 详见 [`srcradar-mcp-server`](https://github.com/usdagfhjkda/srcradar-mcp-server) 与 [`srcradar/NOTICE`](https://github.com/usdagfhjkda/srcradar/blob/main/NOTICE)。

工具使用前提与免责声明见 [`srcradar/TERMS_ADDENDUM.md`](https://github.com/usdagfhjkda/srcradar/blob/main/TERMS_ADDENDUM.md)。
