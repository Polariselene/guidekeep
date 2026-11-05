# B站空会话登录兼容与字幕验收

来源说明：下列兼容入口、调用器及合成测试由本项目开发过程中编写，调用已安装的外部工具接口，未复制或随项目分发上游库实现。上游问题依据见本页链接；外部库保留自身许可证，尤其 bilibili-api-python 17.4.2 的包元数据标识为 GPL-3.0-or-later，不能按本项目 MIT 重新授权或当作本项目文件打包。来源与分发边界见 [third-party-notices.md](third-party-notices.md)。

- [适用条件](#适用条件)
- [兼容入口](#兼容入口)
- [本机调用](#本机调用)
- [字幕验收](#字幕验收)

## 适用条件

仅适用于已实测的 Agent Reach 1.6.1、bilibili-cli 0.6.2、bilibili-api-python 17.4.2：用户本人确认二维码后，原生程序退出0但未取得可用会话。已有会话时直接读取字幕，不再次登录；其他版本先核对上游修复情况。

[上游说明](https://github.com/Panniantong/Agent-Reach/pull/866)确认1.6.1修正的是空会话误报成功，并未修复扫码会话获取。此入口仅在进程内保留原生SDK本次官方响应中的Cookie，不改安装文件、客户端、请求头或端点，不读取日常浏览器。回跳地址只作为响应数据解析，从不访问；有效响应会话优先于无需访问的回跳地址。

须已有用户登录授权，由用户本人扫码。先执行 --self-test 的16项离线检查；用户准备好后才生成约3分钟有效的二维码，并直接打开本机图片。不要只在可能不可见的工具输出里显示，也不让用户反复扫描旧图。收到官方412、验证、限流或未知状态立即停止，不换身份或后端规避。

## 兼容入口

以下代码通过标准输入或 -c 在已有原生B站工具的Python环境临时运行，不新增脚本文件。只输出状态，认证值只交给官方 save_credential 存入原有私有会话。

```python
import argparse, json, time, sys
from pathlib import Path
from urllib.parse import urlsplit, urlencode, unquote

GENERATE = "https://passport.bilibili.com/x/passport-login/web/qrcode/generate?source=main-fe-header"
POLL = "https://passport.bilibili.com/x/passport-login/web/qrcode/poll"
FIELDS = {"sessdata", "bili_jct", "ac_time_value", "buvid3", "buvid4", "dedeuserid"}

class LoginProblem(Exception):
    pass

def classify(payload, own_cookies):
    if payload.get("code") != 0:
        raise LoginProblem("OFFICIAL_API_REJECTED code=" + str(payload.get("code")))
    data = payload.get("data")
    if not isinstance(data, dict):
        raise LoginProblem("INVALID_OFFICIAL_RESPONSE")
    code = data.get("code")
    if code in (86101, 86090):
        return ("waiting" if code == 86101 else "confirm"), None
    if code == 86038:
        raise LoginProblem("QR_EXPIRED")
    if code != 0:
        raise LoginProblem("OFFICIAL_LOGIN_NOT_COMPLETED code=" + str(code))
    values = {}
    for name, value in own_cookies.items():
        key = name.lower().replace("__", "_")
        if key in FIELDS and isinstance(value, str) and value:
            values[key] = value
    if str(values.get("sessdata") or "").strip():
        values["ac_time_value"] = str(data.get("refresh_token") or "")
        return "done", values
    redirect = data.get("url") or ""
    if redirect:
        parsed = urlsplit(redirect)
        host = (parsed.hostname or "").lower()
        allowed_host = host == "bilibili.com" or host.endswith(".bilibili.com") or host == "passport.biligame.com"
        # The redirect is parsed as response data only; it is never requested.
        if parsed.scheme not in ("", "http", "https") or not allowed_host:
            raise LoginProblem("UNEXPECTED_LOGIN_REDIRECT")
        for part in parsed.query.split("&"):
            name, sep, value = part.partition("=")
            key = unquote(name).lower().replace("__", "_")
            if sep and key in FIELDS and value and key not in values:
                values[key] = value
    if not str(values.get("sessdata") or "").strip():
        raise LoginProblem("NO_USABLE_SESSION_IN_OFFICIAL_SUCCESS_RESPONSE")
    values["ac_time_value"] = str(data.get("refresh_token") or "")
    return "done", values

def self_test():
    passed = []
    def event(code=0, **data):
        return {"code": 0, "data": {"code": code, **data}}
    assert classify(event(86101), {}) == ("waiting", None)
    passed.append("unscanned_stays_waiting")
    assert classify(event(86090), {}) == ("confirm", None)
    passed.append("scanned_stays_pending_confirmation")
    for label, payload in [
        ("expired_stops", event(86038)),
        ("unknown_status_never_success", event(99999)),
        ("empty_session_never_success", event(url="https://passport.bilibili.com/callback?gourl=example")),
        ("api_rejection_stops", {"code": -412, "data": {}}),
        ("foreign_redirect_rejected", event(url="https://example.com/?SESSDATA=synthetic")),
    ]:
        try:
            classify(payload, {})
        except LoginProblem:
            passed.append(label)
        else:
            raise AssertionError(label)
    state, fields = classify(event(), {"SESSDATA": "synthetic-only", "bili_jct": "synthetic-csrf"})
    assert state == "done" and fields["sessdata"] == "synthetic-only"
    passed.append("official_response_cookie_session_accepted")
    state, fields = classify(event(url="https://passport.bilibili.com/callback?sessdata=synthetic%2Cencoded&bili_jct=synthetic"), {})
    assert state == "done" and fields["sessdata"] == "synthetic%2Cencoded"
    passed.append("redirect_query_session_and_encoding_preserved")
    for label, redirect in [
        ("response_session_not_discarded_for_unused_cross_domain_redirect", "https://passport.biligame.com/crossDomain?gourl=example"),
        ("response_session_not_discarded_for_relative_redirect", "//passport.bilibili.com/callback?gourl=example"),
    ]:
        state, fields = classify(event(url=redirect), {"SESSDATA": "synthetic-only"})
        assert state == "done" and fields["sessdata"] == "synthetic-only"
        passed.append(label)
    for label, redirect in [
        ("official_cross_domain_query_session_parsed_without_navigation", "https://passport.biligame.com/crossDomain?sessdata=synthetic%2Cencoded&bili_jct=synthetic"),
        ("legacy_http_redirect_data_parsed_without_http_request", "http://passport.bilibili.com/crossDomain?SESSDATA=synthetic%2Cencoded"),
    ]:
        state, fields = classify(event(url=redirect), {})
        assert state == "done" and fields["sessdata"] == "synthetic%2Cencoded"
        passed.append(label)
    print(json.dumps({"self_test": True, "passed": len(passed), "checks": passed}), flush=True)

def capture_response(response, request_url, sink):
    parsed = urlsplit(str(request_url))
    if parsed.hostname == "passport.bilibili.com" and parsed.path.startswith("/x/passport-login/web/qrcode/"):
        if response.code != 200:
            raise LoginProblem("OFFICIAL_HTTP_STATUS " + str(response.code))
        if parsed.path.endswith("/poll"):
            for key, value in (response.cookies or {}).items():
                if isinstance(value, str) and value:
                    sink[key] = value
    return response

async def native_login(qr_path, status_path):
    import asyncio, qrcode
    from bilibili_api import login_v2, Credential
    from bilibili_api.utils.network import get_client
    from bili_cli.auth import save_credential, get_credential
    client = get_client()
    original_request = client.request
    own_response_cookies = {}
    report = {"ok": False, "method": "native-sdk-response-cookie-preservation",
              "saved_session": False, "client": type(client).__name__,
              "request_identity_changed": False}
    async def observed_request(*args, **kwargs):
        response = await original_request(*args, **kwargs)
        request_url = kwargs.get("url") or (args[1] if len(args) > 1 else "")
        return capture_response(response, request_url, own_response_cookies)
    client.request = observed_request
    try:
        generate_api = login_v2.API["qrcode"]["web"]["get_qrcode_and_token"]
        poll_api = login_v2.API["qrcode"]["web"]["get_events"]
        generated = await login_v2.Api(credential=Credential(), **generate_api).result
        login_url = generated.get("url") or ""
        host = (urlsplit(login_url).hostname or "").lower()
        if urlsplit(login_url).scheme != "https" or not host.endswith(".bilibili.com"):
            raise LoginProblem("UNEXPECTED_QR_DESTINATION")
        key = generated.get("qrcode_key")
        if not key:
            raise LoginProblem("QR_KEY_MISSING")
        qrcode.make(login_url).save(qr_path)
        import os
        try:
            os.startfile(str(qr_path))
            print("QR_IMAGE_OPENED_ON_DESKTOP", flush=True)
        except (AttributeError, OSError):
            print("QR_IMAGE_FILE: " + str(qr_path), flush=True)
        print("QR_READY: scan the image with Bilibili and confirm on your phone.", flush=True)
        deadline = time.monotonic() + 180
        last_state = None
        while time.monotonic() < deadline:
            events = await login_v2.Api(credential=Credential(), **poll_api).update_params(qrcode_key=key).result
            if events.get("code") == 0:
                redirect = urlsplit(events.get("url") or "")
                print(json.dumps({"official_status": 0,
                    "response_session_present": any(str(k).lower()=="sessdata" and bool(v) for k,v in own_response_cookies.items()),
                    "redirect_scheme": redirect.scheme,
                    "redirect_host": redirect.hostname}), flush=True)
            state, fields = classify({"code": 0, "data": events}, own_response_cookies)
            if state != last_state:
                print("LOGIN_STATE: " + state, flush=True)
                last_state = state
            if state == "done":
                credential = Credential(**fields)
                if not credential.sessdata:
                    raise LoginProblem("SDK_SESSION_REJECTED")
                save_credential(credential)
                if get_credential(mode="optional") is None:
                    raise LoginProblem("SAVED_SESSION_NOT_LOADABLE")
                report.update(ok=True, saved_session=True, official_status=0)
                status_path.write_text(json.dumps(report, indent=2), encoding="utf-8")
                print(json.dumps(report), flush=True)
                return 0
            await asyncio.sleep(2)
        raise LoginProblem("LOGIN_WAIT_EXPIRED")
    except LoginProblem as error:
        report["error"] = str(error)
    except Exception as error:
        report["error"] = "NATIVE_SDK_ERROR " + type(error).__name__
        if hasattr(error, "code"):
            report["official_error_code"] = error.code
    finally:
        client.request = original_request
        try:
            qr_path.unlink(missing_ok=True)
        except OSError:
            pass
    status_path.write_text(json.dumps(report, indent=2), encoding="utf-8")
    print(json.dumps(report), flush=True)
    return 1

def run(qr_path, status_path):
    import asyncio
    return asyncio.run(native_login(qr_path, status_path))

def transport_test():
    from types import SimpleNamespace
    sink = {}
    response = SimpleNamespace(code=200, cookies={"SESSDATA": "synthetic-only"})
    result = capture_response(response, POLL, sink)
    assert result is response and sink["SESSDATA"] == "synthetic-only"
    other = {}
    capture_response(response, "https://example.com/", other)
    assert not other
    try:
        capture_response(SimpleNamespace(code=412, cookies={}), GENERATE, {})
    except LoginProblem:
        pass
    else:
        raise AssertionError("HTTP412 must stop")
    print(json.dumps({"native_transport_test": True, "passed": 3,
                      "checks": ["response_identity_preserved", "other_sites_not_captured", "http412_stops"]}), flush=True)

if __name__ == "__main__":
    parser = argparse.ArgumentParser()
    parser.add_argument("--self-test", action="store_true")
    parser.add_argument("--qr")
    parser.add_argument("--status")
    args = parser.parse_args()
    if args.self_test:
        self_test()
        transport_test()
    else:
        if not args.qr or not args.status:
            parser.error("--qr and --status required")
        sys.exit(run(Path(args.qr), Path(args.status)))
```

## 本机调用

用 uv tool dir 定位已有 Agent Reach 的工具环境，在其中的 Scripts/python.exe（Windows）或 bin/python 中通过标准输入执行下列调用器；首个参数是本参考文件路径。加 --self-test 只离线验证，不创建登录状态；真实登录去掉此参数。不要写死个人路径或自动安装依赖。

```python
import os, re, subprocess, sys, tempfile
from pathlib import Path
from agent_reach.runtime.tools import resolve
from agent_reach.runtime.sessions import session_env, has_login_state
from agent_reach.runtime import state

reference = Path(sys.argv[1])
self_test = "--self-test" in sys.argv[2:]
source = re.search(r"\x60\x60\x60python\n(.*?)\n\x60\x60\x60", reference.read_text(encoding="utf-8"), re.S).group(1)
exe = resolve("bili")
if not exe:
    raise SystemExit("Managed Bilibili tool missing; do not install automatically")
python = Path(exe).with_name("python.exe" if os.name == "nt" else "python")
args = [str(python), "-X", "utf8", "-c", source]
env = dict(os.environ)
env["PYTHONUTF8"] = "1"
env["PYTHONIOENCODING"] = "utf-8"
if self_test:
    args.append("--self-test")
else:
    if has_login_state("bili-cli"):
        raise SystemExit("Saved session already present; verify subtitle before logging in again")
    folder = Path(tempfile.mkdtemp(prefix="guidekeep-bilibili-login-"))
    env.update(session_env("bili-cli"))
    args += ["--qr", str(folder / "login.png"), "--status", str(folder / "status.json")]
code = subprocess.call(args, env=env)
if not self_test and code == 0 and has_login_state("bili-cli"):
    state.mutate(lambda data: data.setdefault("session_reuse", {}).__setitem__(
        "bilibili", {"source": "manual", "at": state.now()}))
    print("REACH_MANAGED_LOGIN_VERIFIED")
raise SystemExit(code)
```

## 字幕验收

1. 使用未修改的 agent-reach bilibili subtitle "BV号" --timeout 45，检查 ok 和 items[].text；普通 read 的简介不算字幕。
2. 1.6.1统一结果可能没有时间戳。通过同一原生工具的 video "BV号" --subtitle --json 获取 data.subtitle.items 的 from/to/content，原生命令使用 session_env("bili-cli")，不得手工读取或复制认证文件。
3. Windows 子进程仅为本次执行设置 PYTHONUTF8=1、PYTHONIOENCODING=utf-8，防止中文JSON误解码。核对统一文本与原生subtitle.text（或分段content拼接）一致，时间有序且不越过首P时长。
4. 只验收实际返回的首P；语言和自动/人工类型未返回时记未知，其他分P不声称完整。只有取得正文后才继续Guidekeep归档，不自动付费转录。

本机在用户指定 BV1fNVd6WEhY 已跑通：41段、423字符，时间0至68秒，首P时长69秒。上游安装文件未改动。实际过程和中间失败见 TESTING.md；不代表所有视频或新版本均可用。
