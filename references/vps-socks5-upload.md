# VPS 部署 — SOCKS5 上传模式

当 VPS 的 SSH 端口 22 被墙，但 HTTP/HTTPS 端口可通时，通过 Clash Verge + SOCKS5 代理上传文件。

## 前置条件

- Windows: Clash Verge Rev 已安装，有可用的代理节点
- VPS: 已知 root 密码

## 步骤

### 1. 启动 mihomo 内核（如果 Clash GUI 未运行）

```bash
# 启动内核（后台常驻）
"/c/Program Files/Clash Verge/verge-mihomo.exe" \
  -d "/c/Users/<user>/AppData/Roaming/io.github.clash-verge-rev.clash-verge-rev" \
  -f "/c/Users/<user>/AppData/Roaming/io.github.clash-verge-rev.clash-verge-rev/clash-verge.yaml"
```

Clash Verge 的 `mixed-port` 同时支持 HTTP 和 SOCKS5 代理。默认端口在配置文件中（通常是 7897）。

### 2. 验证 SOCKS5 可用

```python
import socket
sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
sock.settimeout(5)
sock.connect(("127.0.0.1", 7897))
sock.send(b"\x05\x01\x00")  # SOCKS5 greeting
resp = sock.recv(2)
print("SOCKS5 OK" if resp == b"\x05\x00" else "FAIL")
```

### 3. 通过 SOCKS5 上传文件（Python + paramiko）

```python
import socket, os, paramiko

def socks5_connect(host, port, proxy="127.0.0.1", proxy_port=7897):
    sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    sock.settimeout(15)
    sock.connect((proxy, proxy_port))
    sock.send(b"\x05\x01\x00")
    sock.recv(2)
    hb = host.encode()
    sock.send(b"\x05\x01\x00\x03" + bytes([len(hb)]) + hb + port.to_bytes(2, 'big'))
    sock.recv(10)
    return sock

sock = socks5_connect("151.242.125.117", 22)
ssh = paramiko.SSHClient()
ssh.set_missing_host_key_policy(paramiko.AutoAddPolicy())
ssh.connect("151.242.125.117", username="root", password="<password>", sock=sock)
sftp = ssh.open_sftp()

# 上传整个目录
count = 0
local_root = r"C:\path\to\local\web"
remote_root = "/var/www/html/site"
for root, dirs, files in os.walk(local_root):
    rel = os.path.relpath(root, local_root)
    remote = remote_root if rel == "." else f"{remote_root}/{rel.replace(chr(92), '/')}"
    try: sftp.stat(remote)
    except:
        try: sftp.mkdir(remote)
        except: pass
    for f in files:
        sftp.put(f"{root}/{f}", f"{remote}/{f}")
        count += 1
sftp.close()
ssh.close()
```

## 踩坑

- `connect` 工具（mingw64）的 HTTP CONNECT 方式对 Hysteria2 代理无效
- SSH 的 `ProxyCommand` 也不稳定
- **唯一可靠方式**：Python socket 直连 SOCKS5 + paramiko
- paramiko 需要通过 pip 安装：`python -m pip install paramiko`
- sandbox (execute_code) 的 Python 可能没有 paramiko，需用系统 Python
