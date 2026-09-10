# 国内服务器安装 Hermes（github/pypi 不通）—— 2026-08 实测

服务器（阿里云 ECS，Alibaba Cloud Linux 3）网络实测：
- ✅ hermes-agent.nousresearch.com（官方安装源）直连 200
- ❌ github.com / pypi.org 超时（HTTP 000）
- ✅ ghfast.top（GitHub 加速）200
- ✅ mirrors.aliyun.com/pypi（有 hermes-agent 包）200

## 安装步骤

```bash
# 1. python3.11（Alibaba Cloud Linux 3 默认 python3.6 太老）
dnf install -y python3.11

# 2. 下载官方安装器（服务器直连可达）
curl -sSL --max-time 60 https://hermes-agent.nousresearch.com/install.sh -o /tmp/hermes_install.sh

# 3. 替换 GitHub 仓库地址为加速镜像（install.sh 用 git clone 安装源码）
sed -i 's#https://github.com/NousResearch/hermes-agent.git#https://ghfast.top/https://github.com/NousResearch/hermes-agent.git#' /tmp/hermes_install.sh

# 4. pypi 镜像环境变量
export UV_INDEX_URL=https://mirrors.aliyun.com/pypi/simple
export UV_DEFAULT_INDEX=https://mirrors.aliyun.com/pypi/simple
export PIP_INDEX_URL=https://mirrors.aliyun.com/pypi/simple
export PIP_TRUSTED_HOST=mirrors.aliyun.com

# 5. setsid 脱离式安装（防云助手 RunCommand 超时杀进程）
setsid bash /tmp/hermes_install.sh > /tmp/hermes-install.log 2>&1 &
```

## 安装器行为（v0.20 实测）
- 用 uv 管理：装 uv → node v22（浏览器工具）→ git clone 仓库到
  `/usr/local/lib/hermes-agent`（FHS root 布局）→ venv 装依赖 → Chrome Headless Shell
- 完成后：`hermes` 命令在 `/usr/local/bin/hermes`，venv python 在
  `/usr/local/lib/hermes-agent/venv/bin/python`，HERMES_HOME=/root/.hermes
- 安装器自动生成完整 config.yaml（~92KB，provider 默认 anthropic）与 .env（默认模板）
  —— 无需手动创建，直接 `hermes config set` 改

## 验证
- `hermes --version`
- `hermes chat -q '回复：服务器就绪'`（首次 2 分钟，后续 10-20 秒）
- 注意首次可能有 `Auxiliary title generation failed: HTTP 400`（DeepSeek 不支持某
  response_format），不影响主对话

## 密钥/凭据传递（不经对话）
本机工具输出会被 secret redaction 脱敏（key 显示为 `sk-4f4...b6c8`）——
这是正常现象，不是出错。传递方式：
1. 本机 python 读 key → 写追加文件（不 print key）
2. workbench upload 追加文件到 /tmp
3. `cat /tmp/append_xxx >> /root/.hermes/.env`（及各 profile 的 .env）

## 安装后快速配置
```bash
hermes config set model.provider deepseek
hermes config set model.default deepseek-v4-flash
hermes config set model.base_url https://api.deepseek.com
```
