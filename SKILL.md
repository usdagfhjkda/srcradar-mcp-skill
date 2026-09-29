---
name: srcradar-mcp-skill
description: How to call srcradar-mcp tools from the Hermes agent — argument shapes (NEVER pass argv arrays or {item: "..."} wrappers), when to use stage_file / stage_dir before any path-taking tool, and the rule against bypassing MCP to invoke srcradar CLI / curl daemon directly. Use when the task involves add_business, set_config, daily.run_*, dispatcher.run_confirmed, or any srcradar subprocess that takes a local file/dir path.
---

# srcradar-mcp 调用规约

> 客户端 Hermes + 服务端 srcradar-mcp daemon(部署在 deploy host,loopback 端口 8764,streamable-http)。
> 本 skill 教 agent **怎么调**,**不**教 srcradar 业务逻辑本身。
>
> **占位符约定**:`<deploy-host>` 是 srcradar 部署机(operator 在 `~/.ssh/config` 里配的 ssh 别名),`<repo>` 是 srcradar 源码根的绝对路径,`<staged-root>` 是 daemon 落盘临时文件的根目录(默认 `~/.cache/srcradar-mcp/staged/`,遵循 XDG),`<local-path>` 是客户端本机文件路径。agent 不应假设具体值,真要用 terminal 直跑前先问 operator。

## 0. 心智模型

srcradar-mcp 是一组工具,agent 通过 Hermes 在本机调用,服务端 daemon 跑在 deploy host 的 loopback 端口上(本机 agent 看就是 loopback)。

**agent 永远不要做的事**:

- ❌ 把 Windows / 本机绝对路径塞 `-s` / `-i`(daemon 在 deploy host,看不到本机 fs)
- ❌ 传 `{"args": ["--help"]}` 或 `{"args": [{"item": "--help"}]}`(Hermes 工具 schema 校验会拒)
- ❌ 跳过 stage 直接调 `manage.add_business -s "C:/.../seed.tsv"`(路径在 deploy host 不存在,daemon 报 "subprocess exit 1: file not found")
- ❌ 用 `terminal` / `curl` / `browser` 绕过 MCP 直接打 daemon(绕过所有审批 + 审计,见 §6)

## 1. 15 个工具全景

| 工具 | 类别 | 备注 |
|---|---|---|
| `db.read_business_summary` | read | 静态读 |
| `db.read_subdomains` | read | 静态读 |
| `db.read_open_ports` | read | 静态读 |
| `db.read_companies` | read | 静态读 |
| `db.read_diff` | read | 静态读 |
| `db.read_single_subdomain` | read | 静态读 |
| `dispatcher.list` | read | 列举可调脚本 |
| `ymicp.icp_mapp_query` | read | stdin 走管道 |
| `dispatcher.run_confirmed` | **write** | 通用跑 srcradar CLI,慎用 |
| `manage.add_business` | **write** | -n / -s / -i / --auto |
| `manage.set_config` | **write** | 改 enabled/web/tcp/icp |
| `daily.run_one_business` | **write** | pdtm/enscan/icp/url stages |
| `daily.run_dashboard_watchdog` | **write** | 重启 dashboard |
| **`stage_file`** | **write** | **客户端→daemon 端落盘单文件** |
| **`stage_dir`** | **write** | **客户端→daemon 端落盘目录** |

**read** 工具静默调用,**write** 工具触发 Hermes 原生审批弹窗(用户点 yes 才执行)。

## 2. 参数形状(踩过的坑)

### 2.1 db.read_* — 严格按 schema 字段名

**对**:

```python
mcp_srcradar_db_read_subdomains(
    business="Acme",
    limit=10,
    offset=0,
    since="2026-09-01",
)
```

**错**(Hermes 工具校验拒收,debug 时长):

```python
mcp_srcradar_db_read_subdomains(args=["--business", "Acme"])      # 错形态
mcp_srcradar_db_read_subdomains({"business": "Acme"})              # 包了一层
mcp_srcradar_db_read_subdomains(business_name="Acme")              # 字段名错
```

### 2.2 manage.* / daily.* / dispatcher.* / ymicp.* — `args: dict`

**调用方把 argv 平铺到 dict 里**:

```python
# 看 help(注意:仍然走 e1-confirmed 弹窗,daemon 不区分 --help)
mcp_srcradar_manage_add_business(
    args={"help": True},                  # ✓ boolean flag
)
# 或者直接以字符串形式传整条 args
mcp_srcradar_manage_add_business(
    args="--help",                        # ✓ 单字符串
)

# 真要加企业
mcp_srcradar_manage_add_business(
    args={
        "-n": "Acme",
        "-s": "<staged-root>/<upload_id>.seed.tsv",  # 来自 stage_file
    },
    _staging_ref="<upload_id>",            # 调用结束 daemon 自动清
)
```

**永远不要**:

```python
mcp_srcradar_manage_add_business(args=["--help"])                # ❌ array of string
mcp_srcradar_manage_add_business(args=[{"item": "--help"}])      # ❌ array of object (Hermes 会发这种然后校验拒)
mcp_srcradar_manage_add_business(args="--help", name="manage.add_business")  # ❌ params 多了 name(不是 tools.call 的字段)
```

> **绕开 `--help` 弹窗**:优先走 SSH + 本地 terminal 工具直跑
> `terminal(ssh <deploy-host> '<repo>/srcradar manage add_business --help')`。
> 不走 MCP = 不弹窗 = 不撞 schema = 调试最快路径。

### 2.3 stage_file — 三种输入模式(三选一)

| 字段 | 形态 | 何时用 |
|---|---|---|
| `stdin` | inline string | LLM 现场生成的小内容 |
| `base64` | base64 编码 | 客户端读文件 → 转码 → 上传(**最常用**) |
| `file_path` + `base64` | 客户端路径 + base64 字节 | 同 base64,额外声明本机路径(daemon 仅记录,不访问) |

**约束**(全部硬性):
- `file_path` 必须配 `base64`,**单独 file_path 一律拒收**
- `filename` 走严格白名单 `[A-Za-z0-9._-]+`,最长 255,无 `..` 无路径分隔符
- payload ≤ 100 MiB(STAGE_MAX_BYTES)
- daemon 不访问客户端 fs,只收字节

**调用模板**(客户端 Python):

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

**Hermes 端典型调用**:

```python
# 1. 读本地文件 → base64
file_bytes = read_local_file("<local-path>/seed.tsv")
b64 = base64_encode(file_bytes)

# 2. 调 stage_file(弹窗后点 yes)
mcp_srcradar_stage_file(
    filename="seed.tsv",
    base64=b64,
    file_path="<local-path>/seed.tsv",
)
# 拿到 staged_path + upload_id

# 3. 调下游工具,把 staged_path 当普通路径用
mcp_srcradar_manage_add_business(
    args={"-n": "Acme", "-s": staged_path},
    _staging_ref=upload_id,   # daemon 在 finally 删
)
```

### 2.4 stage_dir — 目录形态

```python
# 方式 1: 客户端打 tar.gz
import tarfile, io
buf = io.BytesIO()
with tarfile.open(fileobj=buf, mode="w:gz") as tar:
    tar.add("/local/scopedir/target.txt", arcname="target.txt")
    tar.add("/local/scopedir/exclude.txt", arcname="exclude.txt")
b64 = base64.b64encode(buf.getvalue()).decode()

mcp_srcradar_stage_dir(base64=b64)
# 拿到 staged_dir = <staged-root>/<id>/

# 方式 2: 显式 entries list(平铺,无子目录)
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

## 3. 临时文件清理(agent 视角)

agent 唯一要做的:**任何从 `stage_file` / `stage_dir` 拿到 `upload_id` 的下游调用,都顺手带上 `_staging_ref: <upload_id>` 字段**。daemon 在下游工具调用返回时自动清掉临时文件,不需要 agent 二次操心。

```python
mcp_srcradar_manage_add_business(
    args={"-n": "Acme", "-s": staged_path},
    _staging_ref=upload_id,   # ← 这一行永远带上
)
```

`_staging_ref` 是 16-hex 字符串。daemon 对它做 `^[0-9a-f]{16}$` 严格校验,agent 不会因为传错格式炸掉。

agent **不**需要关心 24h ttl、启动 sweep、手工清理 — 那是 daemon / operator 的事。

## 4. 实战场景速查

| 场景 | 工具链 |
|---|---|
| 看某业务子域列表 | `db.read_subdomains(business="X")` |
| 看 `--help` | 走 `terminal(ssh <deploy-host> '<repo>/srcradar manage add_business --help')`,**不走 MCP** |
| 加新业务(纯 db 行,无 seed / scope) | `manage.add_business(args={"-n":"Acme"})` |
| 加新业务 + seed TSV | `stage_file` → `manage.add_business(args={"-n":"Acme","-s":staged}, _staging_ref=id)` |
| 加新业务 + scope 目录 | `stage_dir` → `manage.add_business(args={"-n":"Acme","-i":staged}, _staging_ref=id)` |
| 改某业务 config | `manage.set_config(args={"-n":"Acme","--web":"0"})` |
| 跑某业务一轮 recon | `daily.run_one_business(args={"-n":"Acme"})` |
| 调任意 srcradar 子命令 | `dispatcher.run_confirmed(args="--list")` 或 `args={"list":True}` |
| **绝不走 MCP** | 任何看 `--help` / 调试性查询 → 用 `terminal(ssh <deploy-host> '...')` 直跑(详见 §6 唯一例外)|

## 5. 调试清单(agent 自己撞到的错误)

| 症状 | 原因 | 修法 |
|---|---|---|
| `arguments.args[0] (type): {'item': '--help'} is not of type 'string'` | LLM 把 argv 包成 `[{item: "--help"}]` 撞 schema | 改用 `args={"help": True}` 或 `args="--help"` |
| `subprocess exit 1: [-n <业务名>] is required` | 忘了 `-n` 或拼错 flag 名 | 调 `--help` 先看 flag 表(走 terminal,不走 MCP) |
| `subprocess exit 1: [-s] seed file not found: /local/path/...` | 直接传了客户端路径给 `-s` | 先 `stage_file` 再传 `staged_path` |
| `filename must match [A-Za-z0-9._-]+` | filename 含 `/` `\` `..` 或 unicode | 重命名为 `seed.tsv` 这种纯 ASCII |
| `file_path MUST be paired with base64` | 只传了 `file_path` 不带 `base64` | 加 base64 字节 |
| `one of stdin / base64 / file_path+base64 required` | 三个模式都没传 | 至少传一个 |
| `payload too large: N > 104857600` | 单文件 > 100 MiB | 拆小 / 走 tar.gz |

> **弹窗不出现 / daemon 缺工具 / daemon 拒收调用** — 这些是 operator 配置问题(`trust: untrusted` 没设 / daemon 没重启),**告诉 user** 而不是 agent 自己 debug。

## 6. 🚫 禁止绕过 MCP 直接调 srcradar CLI / curl daemon

**这是硬性规则,agent 永远不要**:
- ❌ 用 `terminal` 工具 `ssh <deploy-host> '<repo>/srcradar manage add_business ...'` 直跑
- ❌ 用 `terminal` 工具 `ssh <deploy-host> 'curl -X POST http://127.0.0.1:8764/mcp ...'` 直接打 daemon
- ❌ 用 `browser` / `web_extract` 访问 daemon 端点
- ❌ 任何不经过 MCP 协议层、直接驱动 srcradar 业务逻辑的路径
