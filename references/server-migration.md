# 服务器迁移：云助手 base64 分块传输（无 SSH 密码 / workbench 挂时）

2026-08 阿里云 → 腾讯云实测。适用场景：目标机有 SSH 密钥可直连，但**源机**无 SSH 凭据、
workbench upload/download 又失效（`UserBehavior.SessionManagerDisabled`）——只剩云助手 RunCommand 通道。

## 原理
- 云助手 RunCommand 可以执行任意 shell（含 `echo <base64> | base64 -d > file`）
- 命令输出上限有限（保守 12KB base64/块），大文件拆块循环拉取
- 核心配置包（排除 node/bin/lsp 运行时）实测仅 78KB → 9 块，1 分钟内拉完

## 步骤

### 1. 源机打包（只打核心配置）
```bash
cd /root/.hermes
tar czf /tmp/hermes-core.tar.gz \
  config.yaml .env cron kanban.db weixin scripts \
  skills/bt-site-ops skills/windows-cn-web-search \
  profiles/wechat2/config.yaml profiles/wechat2/.env profiles/wechat2/cron \
  profiles/wechat2/skills/bt-site-ops profiles/wechat2/skills/windows-cn-web-search
base64 -w0 /tmp/hermes-core.tar.gz | wc -c   # 估算总大小，定块数
```

### 2. 源机切块
```bash
base64 -w0 /tmp/hermes-core.tar.gz | split -b 12000 -d -a 2 - /tmp/hc_
ls /tmp/hc_* | wc -l   # 块数
```

### 3. 本机循环拉取（python，直调云助手 API 拿 Output 字段）
```python
# 复用 ecs-run.py 的 ecs_call/sign（importlib 加载，文件名带横线不能直接 import）
import importlib.util, base64, time
spec = importlib.util.spec_from_file_location("ecs_run", r"D:\hermes猎头\scripts\ecs-run.py")
m = importlib.util.module_from_spec(spec); spec.loader.exec_module(m)

def run_capture(cmd):
    # RunCommand + 轮询 + 取 res[0]["Output"]（云助手返回的 Output 本身是 base64）
    # 参考 ecs-run.py 的 run()，但直接返回解码后的纯输出文本
    ...

chunks = [run_capture("cat /tmp/hc_%02d" % i) for i in range(n)]
data = base64.b64decode("".join(chunks))
open(r"D:\hermes猎头\temp\hermes-core.tar.gz", "wb").write(data)
```

### 4. 校验 + 传目标机
```bash
# 本机校验（Windows 用 python tarfile，GNU tar 会把 D:\ 当远程主机）
python -c "import tarfile; t=tarfile.open(r'...tar.gz'); print(len(t.getnames()))"
scp -i <pem> hermes-core.tar.gz root@<目标IP>:/tmp/
```

## 要点
- 块大小 12KB base64（~16KB 字符输出）在云助手输出上限内；若某块被截断（拼装校验失败），
  把 split -b 降到 8000 重来
- Windows PowerShell 里 `gh api ... > file` 会 Unicode 污染二进制——下载二进制用 python urllib
- 源机脚本上传也用同一模式：`echo <b64> | base64 -d > /tmp/x.sh && bash /tmp/x.sh`
- 迁移完成后**务必先停源机 gateway**（iLink 长轮询双抢消息），再验证新机
