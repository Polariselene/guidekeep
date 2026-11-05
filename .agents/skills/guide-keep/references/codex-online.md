# Codex 已配置 MCP 未出现在对话工具列表时

来源说明：下列标准库协议客户端由本项目开发过程中编写，按 [OpenAI 官方 App Server 协议](https://learn.chatgpt.com/docs/app-server) 调用宿主，不包含 Codex 或 TinyFish 的实现代码。外部程序、服务及其许可证不随本项目 MIT 改变；分发边界见 [third-party-notices.md](third-party-notices.md)。

仅在 Codex 宿主已安装、用户已授权 guidekeep_online 免费读取、有效项目配置存在且终端可运行时使用。本路径通过官方 App Server 调用同一 MCP，不安装抓取器、不访问 OAuth 凭据、不启动模型任务，也不修复 Edge 或绕过网站访问限制。非 Codex 宿主不适用。

当直接工具调用不存在或返回 Unknown tool 时，不要把同一个失效包装入口重复调用，也不要仅要求用户另开对话。先用现有终端执行下面的内存客户端，取得真实正文后继续本次归档流程。Windows 可把 Python 代码放入 PowerShell 的单引号 here-string，并通过标准输入传给现有 Python：`python -X utf8 - URL CWD RESULT_JSON FORMAT`。FORMAT 为 markdown 或 html；需要图注、图片位置与完整代码结构时使用 html。不要新增脚本文件，不将第三方全文写入技能目录。

先定位本机 Codex：优先使用宿主安装的 `.codex/plugins/.plugin-appserver/codex.exe`；其不存在时才使用 PATH 中的 codex。路径由本机环境解析，不把某台机器的绝对路径写成通用依赖。沙箱不能访问宿主配置或联网时，使用宿主正常的命令许可流程；不得自行关闭审批、改用其他身份或读取认证文件。只有新的 OAuth 登录或权限请求才需要用户操作，已有授权不重复索取。

```python
import json, os, queue, shutil, subprocess, sys, threading, time
from pathlib import Path
from urllib.parse import urlparse

url, cwd, output = sys.argv[1:4]
fmt = sys.argv[4] if len(sys.argv) > 4 else 'markdown'
assert urlparse(url).scheme == 'https' and fmt in ('markdown', 'html')
codex_home = Path(os.environ.get('CODEX_HOME') or Path.home() / '.codex')
bundled = codex_home / 'plugins' / '.plugin-appserver' / 'codex.exe'
exe = str(bundled) if bundled.is_file() else shutil.which('codex')
if not exe:
    raise RuntimeError('Codex executable unavailable')

# This client starts only the protocol service, never a model turn.
proc = subprocess.Popen(
    [exe, '-c', 'mcp_servers.node_repl.enabled=false', 'app-server', '--stdio'],
    cwd=cwd, stdin=subprocess.PIPE, stdout=subprocess.PIPE,
    stderr=subprocess.DEVNULL, text=True, encoding='utf-8',
    creationflags=getattr(subprocess, 'CREATE_NO_WINDOW', 0))
inbox = queue.Queue()
seq = 0

def receive():
    for line in proc.stdout:
        try:
            inbox.put(json.loads(line))
        except ValueError:
            continue

threading.Thread(target=receive, daemon=True).start()

def send(message):
    proc.stdin.write(json.dumps(message) + '\n')
    proc.stdin.flush()

def rpc(method, params):
    global seq
    seq += 1
    request_id = seq
    send({'id': request_id, 'method': method, 'params': params})
    deadline = time.monotonic() + 100
    while time.monotonic() < deadline:
        try:
            message = inbox.get(timeout=1)
        except queue.Empty:
            if proc.poll() is not None:
                raise RuntimeError('App Server exited')
            continue
        if 'method' in message and 'id' in message:
            raise RuntimeError('User action required: ' + message['method'])
        if message.get('id') == request_id:
            if 'error' in message:
                raise RuntimeError(str(message['error']))
            return message.get('result')
    raise TimeoutError(method)

try:
    rpc('initialize', {
        'clientInfo': {'name': 'guidekeep_online_reader', 'version': '1.0'},
        'capabilities': {'experimentalApi': True}})
    send({'method': 'initialized'})
    effective = rpc('config/read', {'cwd': cwd, 'includeLayers': True})
    config = effective['config'].get('mcp_servers', {}).get('guidekeep_online')
    if not config or config.get('enabled_tools') != ['fetch_content']:
        raise RuntimeError('Expected project free-reader allowlist missing')
    if config.get('url') != 'https://agent.tinyfish.ai/mcp':
        raise RuntimeError('Unexpected reader endpoint')
    thread = rpc('thread/start', {'cwd': cwd, 'ephemeral': True})['thread']['id']
    status = rpc('mcpServerStatus/list', {
        'threadId': thread, 'serverName': 'guidekeep_online',
        'detail': 'toolsAndAuthOnly'})
    if not any(x.get('name') == 'guidekeep_online'
               and x.get('runtimeStatus') == 'connected'
               for x in status.get('data', [])):
        raise RuntimeError('Reader did not connect')
    response = rpc('mcpServer/tool/call', {
        'threadId': thread, 'server': 'guidekeep_online',
        'tool': 'fetch_content', 'arguments': {
            'urls': [url], 'format': fmt, 'links': True,
            'image_links': True, 'iframe_links': False,
            'page_metadata': True, 'per_url_timeout_ms': 30000,
            'purpose': 'Archive this public guide. Read actual text, code and image references; stop at access walls or verification challenges.'}})
    if response.get('isError'):
        raise RuntimeError('MCP call returned an error')
    payload = json.loads(next(x['text'] for x in response['content']
                              if x.get('type') == 'text'))
    Path(output).write_text(json.dumps(payload, ensure_ascii=False, indent=2),
                            encoding='utf-8')
    if payload.get('errors') or not payload.get('results'):
        raise RuntimeError('Target read failed; details saved to result JSON')
    for item in payload['results']:
        print(json.dumps({
            'title': item.get('title'), 'final_url': item.get('final_url'),
            'text_chars': len(item.get('text', ''))}, ensure_ascii=False))
finally:
    proc.terminate()
    try:
        proc.wait(timeout=8)
    except subprocess.TimeoutExpired:
        proc.kill()
        proc.wait(timeout=8)
```

结果 JSON 是本次真实在线响应，不是用户离线导入。检查 results 中的正文、最终 URL、代码和图片，不能把无 errors 但只有提示页的响应当成功。正常读取后继续主流程的结构化、归档与打开验证；不要到“工具读取成功”就停下，也不要求用户重复验证。如果服务要求登录、出现站点限制或该协议版本不支持所需方法，记录实际原因，不启动付费自动化或无限轮换入口。

该协议随宿主版本演进；不支持时先核对本机 `codex app-server generate-json-schema` 的契约与[官方 App Server 文档](https://learn.chatgpt.com/docs/app-server)，不从认证存储提取 token，不使用已移除的 codex mcp-server。上面代码只作为指令中的临时协议客户端，不属于归档产物或常驻服务。
