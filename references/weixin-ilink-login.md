# 微信 iLink Bot 扫码登录（Hermes gateway）详细流程

2026-08 实测。Hermes v0.20 的 `gateway/platforms/weixin.py` 走腾讯官方
iLink Bot API（`ILINK_BASE_URL = https://ilinkai.weixin.qq.com`），扫码登录，
低风控（微信官方 ClawBot 同源渠道）。

## 端点
- `ilink/bot/get_bot_qrcode?bot_type=3` → 返回 `qrcode`（hex token）+ `qrcode_img_content`（**完整可扫 liteapp URL**）
- `ilink/bot/get_qrcode_status?qrcode=<token>` → status: wait / scaned / scaned_but_redirect / expired
- 其他：getupdates（长轮询收消息）、sendmessage、sendtyping、getconfig、getuploadurl

## 扫码脚本（参数化，支持多 profile）

```python
# weixin_qr.py —— 用法: python weixin_qr.py <hermes_home>
import asyncio, sys, json, io
sys.stdout = io.TextIOWrapper(sys.stdout.buffer, encoding="utf-8")
sys.path.insert(0, "/usr/local/lib/hermes-agent")   # 服务器安装路径
from gateway.platforms.weixin import qr_login

home = sys.argv[1] if len(sys.argv) > 1 else "/root/.hermes"
creds = asyncio.run(qr_login(home, timeout_seconds=480))
print("\n===RESULT===")
print(json.dumps(creds, ensure_ascii=False) if creds else "LOGIN_FAILED")
```

服务器执行（用 hermes 的 venv python）：
```bash
setsid /usr/local/lib/hermes-agent/venv/bin/python /tmp/weixin_qr.py /root/.hermes > /tmp/weixin_qr.log 2>&1 &
```

## 流程要点
1. 日志输出 `请使用微信扫描以下二维码：` + liteapp URL（`https://liteapp.weixin.qq.com/q/<id>?qrcode=<hex>&bot_type=3`）
   —— **把 URL 给用户**，微信里打开/扫码即可（qrcode 库未装时终端 ASCII 码渲染失败，不影响）
2. 轮询 480 秒，过期自动刷新（最多 3 次）
3. 成功输出：
   ```
   微信连接成功，account_id=<account_id>@im.bot
   {"account_id": "...@im.bot", "token": "...@im.bot:<hex>", "base_url": "https://ilinkai.weixin.qq.com", "user_id": "...@im.wechat"}
   ```

## 凭据配置（.env）
```bash
cat >> /root/.hermes/.env << 'EOF'
WEIXIN_TOKEN=<account_id>@im.bot:<hex>
WEIXIN_ACCOUNT_ID=<account_id>@im.bot
WEIXIN_ALLOWED_USERS=<user_id>@im.wechat    # 关键！不配则拒收所有消息
EOF
```
- 重启服务生效（systemctl restart hermes-gateway）
- 多 profile：凭据写各自 `profiles/<name>/.env`，allowlist 各自配

## 验证
- `hermes send -l` → "no channels discovered yet" = 通道未建立（需用户先给 bot 发消息）
- 用户微信发消息 → bot 回复 = 端到端通
- 之后 `hermes send -t 'weixin:<chat_id>' '消息'` 可主动推送

## 注意
- 日志含凭据（token），不要完整贴进对话/文档；凭据存 `D:\hermes猎头\阿里云服务器信息.txt` 类本地文件
- token 类似会话凭据，丢失需重新扫码
- 微信扫码提示 "scaned" 后需用户在手机确认才算完成
