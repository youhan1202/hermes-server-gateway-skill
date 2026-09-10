---
name: hermes-server-gateway
description: 在服务器（国内网络）上部署 Hermes gateway 并接入消息平台（微信官方 iLink Bot、Telegram 等）的完整流程与运维。含国内镜像安装、systemd 服务化、微信扫码登录、多 profile 多账号、消息精简配置、provider 切换、双实例清理。用户要求"服务器上装 Hermes / 微信接入 / gateway 配置 / 多微信"时加载。
tags: [hermes, gateway, weixin, wechat, ilink, systemd, profile, server]
---

# Hermes Server Gateway 部署与运维

在服务器上部署 Hermes Agent 作为常驻消息 gateway（微信/Telegram 等平台），
以及日常运维（服务管理、模型切换、消息精简、多账号）。

## 触发条件
- 在服务器/远程机器上安装 Hermes（国内网络环境）
- 配置 gateway 接入微信/其他消息平台
- 多微信/多平台账号（profile 机制）
- gateway 服务化（systemd）、消息推送粒度调整、模型切换

## 一、国内服务器安装 Hermes（github/pypi 不通的环境）

服务器无法直连 github.com / pypi.org 是国内常态，官方 `curl install.sh | bash` 会
在 git clone 阶段失败。解法：

1. **测连通**：`curl -sI https://hermes-agent.nousresearch.com/install.sh`（官方源通常可达）；
   `curl -sI https://github.com` / `https://pypi.org`（通常超时）
2. **测镜像（按云厂商测多个源，实测差异大）**：ghfast.top（阿里云 ECS 实测 200，**腾讯云实测 403**）、
   gh-proxy.com / gh.ddlc.top（腾讯云实测 200）、ghproxy.com 等常挂；阿里云 pypi `https://mirrors.aliyun.com/pypi/simple/`（两云均 200）。
   安装前先 `curl -sI -L --max-time 10 <加速源>/https://github.com/NousResearch/hermes-agent/archive/refs/heads/main.tar.gz`
   批量测 6-7 个源挑可用的，**不要默认 ghfast.top 通用**
3. **定制安装器**：
   ```bash
   curl -sSL https://hermes-agent.nousresearch.com/install.sh -o /tmp/hermes_install.sh
   sed -i 's#https://github.com/NousResearch/hermes-agent.git#https://ghfast.top/https://github.com/NousResearch/hermes-agent.git#' /tmp/hermes_install.sh
   export UV_INDEX_URL=https://mirrors.aliyun.com/pypi/simple
   export UV_DEFAULT_INDEX=https://mirrors.aliyun.com/pypi/simple
   export PIP_INDEX_URL=https://mirrors.aliyun.com/pypi/simple
   export PIP_TRUSTED_HOST=mirrors.aliyun.com
   setsid bash /tmp/hermes_install.sh > /tmp/hermes-install.log 2>&1 &
   ```
4. 完成后 `hermes --version` 验证；venv python 在 `/usr/local/lib/hermes-agent/venv/bin/python`
   （FHS root 布局）；HERMES_HOME=/root/.hermes

5. ⚠️ **npm 浏览器依赖失败 ≠ 安装失败**（2026-08 腾讯云实测）：install.sh 尾部 `npm install`（playwright chromium，~114MB）在部分国内网络超时/失败，日志 `✗ npm install failed or timed out; Node.js dependencies were not installed`，**browser 工具 unavailable 但 chat/gateway/微信/cron 全部正常**，可后补。
   但**hermes 命令可能没注册**（launcher 步骤被跳过，`command -v hermes` 为空）——手动补 wrapper：
   ```bash
   rm -f /usr/local/bin/hermes      # ⚠️ 必须先删，见下方坑
   cat > /usr/local/bin/hermes << 'EOF'
   #!/usr/bin/env bash
   unset PYTHONPATH
   unset PYTHONHOME
   exec "/usr/local/lib/hermes-agent/venv/bin/python" "/usr/local/lib/hermes-agent/hermes" "$@"
   EOF
   chmod +x /usr/local/bin/hermes
   ```
   ⚠️ **大坑：先 `ln -sf <源> /usr/local/bin/hermes` 再 `cat > /usr/local/bin/hermes` 会跟随符号链接把 hermes 源文件覆盖成 bash 内容**（`ModuleNotFoundError` → `SyntaxError: invalid syntax`）。修复：`cd /usr/local/lib/hermes-agent && git checkout -- hermes` 恢复源文件，再按上面先 `rm -f` 再写独立 wrapper。

完整细节：`references/cn-server-hermes-install.md`

## 二、模型 provider 配置与切换

- 密钥放 `.env`（`DEEPSEEK_API_KEY` / `XIAOMI_API_KEY` / `TAVILY_API_KEY` 等），config 不写明文
- 密钥传递**不经对话**：本机脚本读 key → 生成追加文件 → workbench upload → `cat >> .env`
  （工具输出会被 secret redaction 脱敏，对话里看不到完整 key 是正常现象）
- provider 配置：`hermes config set providers.xiaomi.base_url https://api.xiaomimimo.com/v1` 等逐项设
- 切换模型：`hermes config set model.provider <name>` + `model.default` + `model.base_url`
  （DeepSeek ↔ MiMo 实测可来回切，旧 provider 配置保留）
- 验证：`hermes chat -q '只回复模型名称'`（模型会自报）

## 三、gateway 服务化（systemd 开机自启）

```bash
# root 环境必须加 --run-as-user root，否则报 "Refusing to install as root"
hermes gateway install --system --run-as-user root --start-now --start-on-login
# 多 profile：每个 profile 独立服务（单元名带 profile 后缀）
hermes -p wechat2 gateway install --system --run-as-user root --start-now --start-on-login
```
- 单元：`hermes-gateway.service` / `hermes-gateway-wechat2.service`
- 管理：`systemctl status/restart hermes-gateway`，日志 `journalctl -u hermes-gateway -f`
- ⚠️ **双实例陷阱**：先 setsid 跑过 gateway 再装服务时，旧进程不会自动退出 →
  同一 profile 双实例并存（消息重复/cron 双触发）。`ps aux | grep "gateway run"` 检查，
  精准 `kill <旧PID>` 清理，只留 systemd 服务实例
- ⚠️ 含 `systemctl restart hermes-gateway` 的命令可能被 Hermes approval 拦
  （"cannot restart gateway from inside the gateway process"）——拆成两条命令：
  先做配置/文件变更，再单独跑重启

## 四、微信接入（腾讯官方 iLink Bot API，低风控）

Hermes v0.20 weixin 适配器走 **ilinkai.weixin.qq.com**（腾讯官方 iLink Bot API）——
**不是第三方协议**，扫码登录、低封号风险（微信官方 ClawBot 同源渠道）。

- 配置为环境变量：`WEIXIN_TOKEN` / `WEIXIN_ACCOUNT_ID`（必填）、
  `WEIXIN_ALLOWED_USERS`（白名单，只允许指定微信触发）、可选 `WEIXIN_DM_POLICY` 等
- **不配 allowlist 会拒绝所有消息**（日志：`Unauthorized user: <id> on weixin`）——
  把绑定微信的 user_id 加入 `WEIXIN_ALLOWED_USERS`
- 扫码登录流程：调 `qr_login`（gateway.platforms.weixin 模块）→ 打印 liteapp URL →
  用户微信扫码确认 → 返回凭据 dict（account_id/token/user_id）
- 端到端验证：用户先在微信里给 bot 发消息建立 channel（`hermes send -l` 无 channel 时
  不可主动发送），bot 回复即通

完整流程与参数化扫码脚本：`references/weixin-ilink-login.md`

## 五、多微信 = 多 profile

```bash
hermes profile create wechat2 --clone-from default   # 克隆模型等配置
# 新 profile 需重新复制 API key（clone 不复制 .env）：
grep '^DEEPSEEK_API_KEY=' /root/.hermes/.env >> /root/.hermes/profiles/wechat2/.env
# 各自扫码绑微信（qr_login 的 hermes_home 传 profile 路径）
# 各自装服务（见第三节），凭据写各自 profile 的 .env
```
- 每个 profile 独立 config/.env/会话/记忆/skills（技能要复制到各 profile 的 skills/ 目录）
- 检查任务：`hermes cron list` / `hermes -p wechat2 cron list`（每个 profile 独立 cron）

## 六、消息推送粒度精简（用户偏好：只同步关键动作和结果）

gateway 默认把工具过程/中间说明全推到微信，用户嫌繁杂。配置（两个 profile 都要设）：
```bash
hermes config set display.tool_progress off                    # 工具步骤不推
hermes config set display.interim_assistant_messages false     # 动作说明不推
hermes config set display.busy_ack_detail false                # 忙碌确认不推
hermes config set display.background_process_notifications errors  # 后台任务只报错
```
改完重启服务生效。效果：微信里一条指令 = 一条答案。

## 六点五、会话自动重置（session_reset，防旧上下文污染）

**Hermes 默认 `session_reset: none`（2026-07 起会话永不自动重置）**——同一频道消息永远续在最早会话上，
几天后旧话题上下文污染新问题（症状：**"回复昨天的问题"**、api_calls=1 不搜索就乱答）。
修复（两个 profile 都配）：
```bash
hermes config set session_reset.mode both       # 每日定时 + 空闲双触发
hermes config set session_reset.at_hour 4       # 每天 4:00 重置
hermes config set session_reset.idle_minutes 1440  # 或空闲 24h
hermes config set session_reset.notify true     # 重置时通知用户
```
- 用户**立即清空旧会话**：在微信里发 `/new`（gateway 斜杠命令）
- 会话路由异常排查：`hermes sessions list` 看 Last Active 时间戳是否对不上（旧会话被续接）
- 消息"重发没回复"可能是 MessageDeduplicator 误杀（同内容短时重发）——提示用户改措辞或隔几分钟

## 六点七、cron 投递与内容质量

- **CLI 创建的任务 `--deliver origin` 无有效来源** → 任务每次执行（输出文件在 cron/output/ 生成）但从不投递。修复：`hermes cron edit <id> --deliver "weixin:<user_id>"`
- `hermes send -t "weixin:<user_id>"` 需 channel 已建立（用户先给 bot 发过消息）；返回 `sent` = 链路通
- **AI 日期/星期幻觉**：模型生成内容里的星期几不可靠（实测 MiMo 把 2026-08-31 周一写成"周日"）——cron prompt 里明确"标题只写日期，绝对不要写星期几"；涉及日期核实用 `python -c "import datetime; ..."` 程序验证，不轻信模型输出

## 八、跨服务器迁移（旧机停用前必读）

阿里云 → 腾讯云实测流程（2026-08），核心结论：

1. **只迁核心配置，不迁运行时**：`/root/.hermes` 大头是 node/bin/lsp（运行时组件，新机装完自动生成）。
   打包只需：`config.yaml .env cron kanban.db weixin scripts skills/<自定义> profiles/<名>/config.yaml profiles/<名>/.env profiles/<名>/cron profiles/<名>/scripts` **+ SOUL.md（服务器人格，打包清单极易漏，漏迁 = 人格丢失）**
   ——实测 tar.gz 仅 **78KB**（vs 全量 376M）。
   **⚠️ 必须包含各 profile 的 scripts/ 目录**（no-agent 定时任务脚本所在，如降雨提醒 weather_rain_alert_tai_an.py）——漏迁 = 任务 active 但静默空输出、永不投递（2026-08 实测踩坑，用户以为"天气定时正常"实为假象）。
2. **⚠️ 双实例抢消息（最重要的坑）**：微信 iLink 是**长轮询拉取**，**谁在轮询谁收消息**。
   迁移期间新旧两台服务器同时跑 gateway = 同一 bot 双抢消息，极不稳定。
   **必须先停旧机器 gateway**（`systemctl stop + disable` 两个服务，确认 `ps aux | grep "gateway run"` 为 0）再启用新机。
3. **微信 token 跨机器有效但可能过期**：iLink token 是账号级凭据，default 的 token 迁移后直接可用；
   但部分账号 token 会过期（日志 `Session expired; pausing for 10 minutes`）→ 重新扫码会生成**新 bot 账号**，
   必须同步更新 `.env` 的 `WEIXIN_TOKEN` / `WEIXIN_ACCOUNT_ID`（sed 替换，allowlist 的 user_id 不变）。
4. **无 SSH 密码/通道挂时的文件传输**：workbench 挂（SessionManagerDisabled）且不知 SSH 密码时，
   用云助手 RunCommand **base64 分块传输**：服务器 `base64 -w0 <包> | split -b 6000 -d -a 3 - /tmp/fc_`，
   本机 python 循环 `cat /tmp/fc_XXX` 拉每块（**6KB 块 + 逐块长度校验**，见下方坑）→ 拼装 → b64decode。
   ⚠️ **分块三坑（2026-08 实测）**：①`split -a 2` 后缀上限 99 块，>100 块截断只拉到 1/3（必须 `-a 3`）；
   ②12KB 块 × 300 块时云助手输出偶发污染 → 拼装后 `not a gzip file`（降到 6KB + 逐块 `len(out)==CHUNK` 校验重试）；
   ③拼装后必须 md5 对比服务器参考值 + 检查 gzip 魔数 `\x1f\x8b`。手抄 base64 必出错（插空格）。
   完整脚本模式见 `references/server-migration.md`。
5. 新机装完后 `hermes --version`、`hermes cron list`（双 profile）、`hermes chat -q` 验证，
   再配 systemd 服务（第三节），最后让用户微信发消息端到端验证。
6. **静态站迁移优先 HTTP 直拉（比 base64 分块快一个数量级）**：旧服务器 80 端口还开放时，
   新服务器直接 `curl -o <file> http://<旧IP>/<path>` 逐个拉文件（实测阿里云→腾讯云 6 文件含 4.6M bgm 秒级完成）。
   只拉 index.html **当前引用**的哈希文件（assets/ 里旧版本 js/css 是构建冗余，index-*.js 每个 150KB × 20 个 ≈ 3MB，不拉）；
   拉完配 nginx server（`try_files $uri $uri/ =404`）→ 防火墙放行 http → 外网验证 200。
   纯静态站（无 PHP/API，数据在 localStorage）迁移零风险，无需迁数据库。

## 九、平台工具集

- `config.yaml` 的 `platform_toolsets` 里**没有 weixin 条目** = 微信平台用核心工具集
  （terminal/file/搜索等全可用）——这就是微信 AI 能直接操作服务器的基础
- 给微信 AI 配技能：复制 SKILL.md 到 `/root/.hermes/skills/<name>/`（及各 profile 的
  `profiles/<name>/skills/`），新会话自动加载
- 服务器管理类技能注意内置安全规则（破坏性操作先确认）

## 十、装 ClawHub/第三方技能（无 openclaw CLI 时）

用户给的 `openclaw skills install @owner/skill` 类命令：本机无 openclaw CLI、npm registry 也 404
（@tencent-adm/tencentcloud-api-skill 实测 npm 404）时，走 GitHub 镜像获取：

1. **找公开镜像**：`gh api "search/repositories?q=<skill名>" --jq '.items[] | {name: .full_name, desc: .description}'`，
   找描述含 "Automated public mirror of the official ClawHub ..." 的仓库（ClawHub 官方技能常有人做自动镜像）
2. **下载 tarball 用 python urllib，不要用 PowerShell `>` 重定向**（PS 5.1 重定向会污染二进制）：
   ```python
   req = urllib.request.Request('https://api.github.com/repos/<owner>/<repo>/tarball', headers={'User-Agent': 'hermes'})
   data = urllib.request.urlopen(req, timeout=60).read()
   open('skill.tar.gz','wb').write(data)  # tarfile 解压
   ```
3. 把 SKILL.md + references/ + scripts/ 复制到 Hermes skills 目录（本机 + 服务器各 profile 的 skills/）
4. 装完 `skill_view(name)` 验证 frontmatter/结构

实测：@tencent-adm/tencentcloud-api-skill → 镜像 Marco9442/tencentcloud-api-skill（tcapi 技能，腾讯云 API 助手，依赖 tccli：
`dnf install -y python3.11-pip && python3.11 -m pip install -i https://mirrors.aliyun.com/pypi/simple tccli`；凭证走 `tccli auth login` 浏览器 OAuth，技能安全红线是**严禁索要 SecretId/SecretKey**）。

## 十一、Windows 连服务器的 SSH pem 权限

Windows 下 ssh 用 pem 私钥报 "UNPROTECTED PRIVATE KEY FILE" 时收紧 ACL：
```bat
cmd /c 'icacls "D:\path\key.pem" /inheritance:r /grant:r "%USERNAME%:(R)"'
```
⚠️ 必须用 `cmd /c` 包裹——PowerShell 直接调 `icacls ... "$env:USERNAME:(R)"` 会把 `(R)` 拆成独立参数报
`Invalid parameter "(R)"`。密钥放工作目录（如 D:\hermes猎头\keys\）并纳入备份。

## 十一之二、服务器上启用浏览器工具（无头 Chrome，国内网络）

国内服务器装 Hermes 时 install.sh 尾部的 npm 步骤常失败 → 浏览器工具默认不可用。
补齐流程（2026-09 腾讯云 OpenCloudOS 9.6 实测跑通）：

1. **装 agent-browser**：`npm install -g agent-browser`（腾讯云 npm 镜像
   `mirrors.tencentyun.com/npm` 可用）
   - 0.37.x 的 npm 包**自带全部平台二进制**（`bin/agent-browser-linux-x64` 等），
     npm 新版跳过 postinstall（提示 allowScripts）**不影响**，不必 `--allow-scripts`
2. **装 Chrome for Testing**（Google 官方源国内拉不动，实测重试 3 次全失败）→ 用 npmmirror：
   ```bash
   wget -O /root/chrome.zip \
     https://cdn.npmmirror.com/binaries/chrome-for-testing/<版本>/linux64/chrome-linux64.zip
   mkdir -p /opt/chrome-for-testing && cd /opt/chrome-for-testing && unzip -q /root/chrome.zip
   ln -sf /opt/chrome-for-testing/chrome-linux64/chrome /usr/local/bin/google-chrome
   ```
   - 版本号取 agent-browser install 日志打印的版本，或查
     `https://googlechromelabs.github.io/chrome-for-testing/last-known-good-versions-with-downloads.json`
   - 镜像 URL 是 `cdn.npmmirror.com/binaries/chrome-for-testing/...`（实测 200，187MB）
3. **补系统库**（RHEL 系/OpenCloudOS）：`dnf install -y mesa-libgbm alsa-lib`
   （其余 nss/gtk3/atk/libX*/cairo/pango 等系统自带；`ldd <chrome> | grep -c 'not found'` 应为 0）
4. **写环境变量**：各 profile 的 `.env` 追加
   `AGENT_BROWSER_EXECUTABLE_PATH=<chrome 绝对路径>`
5. **⚠️ 必须设 `browser.backend off`**：`hermes config set browser.backend off`（每个 profile 都要设）
   - 默认空值时，Hermes 一旦发现 `browser-use` CLI 或 `uvx` 可用就切到 **Browser Use 云模式**，
     给模型的工具变成 `browser_exec`（云端），首次调用要从 pypi 拉 browser-use 包 → 国内直接卡死
   - 症状：`hermes chat` 日志停在 `┊ 🌐 preparing browser_exec…` 数分钟无响应；
     `hermes doctor` 的工具集列表里 browser 显示 `⚠ system dependency not met`
   - 注意值要写成字符串 `'off'`（YAML 里裸 off 会变布尔 False，判断 `== "off"` 失效）；
     `hermes config set browser.backend off` 会自动加引号，验证时 grep config.yaml 确认
6. **验证**：`hermes doctor` → ✓ agent-browser / ✓ Playwright Chromium / ✓ browser；
   实测 `hermes chat -q '用 browser_navigate 工具打开 https://example.com ，用一句话报告 h1 标题'`
   ≈10s 返回 "Example Domain"（修复前卡 4 分钟）
7. 改完 `systemctl restart hermes-gateway hermes-gateway-wechat2` 生效；
   残留浏览器进程清理：`pkill -f agent-browser-chrome`

**原理**：`tools/browser_tool._chromium_installed()` 依次判定
① `AGENT_BROWSER_EXECUTABLE_PATH` ② PATH 中的 google-chrome/chromium/chrome ③ Playwright 缓存目录
——满足任一条浏览器工具才会对模型可见。第 5 步的 `browser.backend` 决定走
browser-use 云 CLI（browser_exec）还是内置本地工具（browser_navigate 等）。

## 陷阱汇总
- root 环境装 systemd 服务必须 `--run-as-user root`
- setsid 旧进程不会因服务安装自动退出 → 双实例，需手动 kill
- 微信无 allowlist = 拒收所有消息
- `.env` 修改 + `systemctl restart` 同命令组合易被 approval 拦 → 拆开执行
- **workbench upload 也可能整体失效**（`UserBehavior.SessionManagerDisabled`，控制台会话管理被关）——脚本上传改用**云助手 base64 直写文件**模式（`echo <b64> | base64 -d > /tmp/x.sh && bash /tmp/x.sh`，详见 alicloud-ecs-ops 技能 4.6 节）
- 服务器无代理：Tavily 搜索直连可用（api.tavily.com 405 是 HEAD 方法限制，POST 正常），
  降级链用搜狗/360 直连（见 windows-cn-web-search 技能）
- dnf 装的 python3.11 **默认无 pip**（`python3.11 -m pip` → No module named pip）——装 tccli 等工具前先
  `dnf install -y python3.11-pip`，pip 装包加 `-i https://mirrors.aliyun.com/pypi/simple`
- 体检时留意 mysqld：宝塔装的数据库服务默认 disabled（不自启），且可能无日志停掉（小内存 OOM），
  `systemctl is-enabled mysqld` + `ss -tlnp | grep 3306` 确认（详见 alicloud-ecs-ops 9.5 节）。
  ⚠️ **停着的服务可能是用户主动停用**（实测：MariaDB 无服务依赖被用户关闭省内存）——体检发现服务
  not running 时先问/查信息文件确认是否故意停用，不要直接当故障"修"（会违背用户意图），
  启动/自启改动前先确认
