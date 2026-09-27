# Vultr Ubuntu 部署 Shadowsocks-Python（Python 3.14）

测试环境：

```text
VPS：Vultr
系统：Ubuntu
Python：3.14
Shadowsocks-Python：3.0.0
客户端：Shadowrocket
示例端口：14449
加密方式：aes-256-gcm
```

> Shadowsocks-Python 3.0.0 是较老的实现，在 Python 3.14 环境需要进行兼容性修改。
>
> 以下所有 `YOUR_SERVER_IP`、`YOUR_STRONG_PASSWORD` 均需要替换为自己的信息，不要把真实密码上传到公开 GitHub 仓库。

---

# 一、SSH 登录 VPS

在本地电脑执行：

```bash
ssh root@YOUR_SERVER_IP
```

登录成功后应看到：

```text
root@vultr:~#
```

---

# 二、更新系统并安装依赖

```bash
apt update
apt install -y wget gcc g++ autoconf automake make build-essential tcpdump netcat-openbsd
```

---

# 三、下载 Shadowsocks 安装脚本

```bash
cd /root
wget https://raw.githubusercontent.com/sucong426/VPN/main/ss.sh
```

检查：

```bash
ls -l ss.sh
```

赋予权限：

```bash
chmod +x ss.sh
```

---

# 四、运行安装脚本

使用：

```bash
bash ss.sh
```

不要使用：

```bash
sh ss.sh
```

否则 Ubuntu 的 `dash` 可能报：

```text
Syntax error: "(" unexpected
```

---

# 五、安装过程中填写参数

脚本会依次要求设置密码、端口和加密方式。

推荐：

```text
Password:
YOUR_STRONG_PASSWORD
```

端口：

```text
14449
```

加密方式：

```text
aes-256-gcm
```

出现：

```text
Press any key to start...or Press Ctrl+C to cancel
```

按 Enter 开始安装。

---

# 六、检查 Python 版本

```bash
python3 --version
```

例如：

```text
Python 3.14.x
```

---

# 七、检查 Shadowsocks

```bash
ssserver --version
```

如果直接显示：

```text
Shadowsocks 3.0.0
```

可以跳到第九步。

如果出现：

```text
AttributeError: module 'collections' has no attribute 'MutableMapping'
```

需要执行下面的 Python 3.14 兼容性修复。

---

# 八、修复 Python 3.14 兼容问题

先自动找到 `lru_cache.py`：

```bash
LRCACHE=$(find /usr/local/lib -path '*shadowsocks*' -name 'lru_cache.py' | head -1)
```

确认路径：

```bash
echo "$LRCACHE"
```

正常情况下类似：

```text
/usr/local/lib/python3.14/dist-packages/shadowsocks-3.0.0-py3.14.egg/shadowsocks/lru_cache.py
```

替换旧 API：

```bash
sed -i 's/collections.MutableMapping/collections.abc.MutableMapping/g' "$LRCACHE"
```

添加：

```python
import collections.abc
```

执行：

```bash
grep -q '^import collections\.abc' "$LRCACHE" || \
sed -i '/^import collections$/a import collections.abc' "$LRCACHE"
```

再次检查：

```bash
ssserver --version
```

正常应该显示：

```text
Shadowsocks 3.0.0
```

---

# 九、查看 Shadowsocks 配置

```bash
cat /etc/shadowsocks.json
```

配置一般类似：

```json
{
    "server":"0.0.0.0",
    "server_port":14449,
    "local_address":"127.0.0.1",
    "local_port":1080,
    "password":"YOUR_STRONG_PASSWORD",
    "timeout":300,
    "method":"aes-256-gcm",
    "fast_open":false
}
```

重点确认：

```text
server_port = 14449
password = 自己设置的密码
method = aes-256-gcm
```

---

# 十、启动 Shadowsocks

重启服务：

```bash
/etc/init.d/shadowsocks restart
```

正常应该看到：

```text
Starting Shadowsocks success
```

查看状态：

```bash
/etc/init.d/shadowsocks status
```

正常：

```text
Shadowsocks (pid XXXXX) is running...
```

---

# 十一、检查 14449 端口

```bash
ss -lntup | grep 14449
```

正常应该同时看到：

```text
udp   UNCONN ... 0.0.0.0:14449 ...
tcp   LISTEN ... 0.0.0.0:14449 ...
```

---

# 十二、配置 Ubuntu UFW 防火墙

查看防火墙：

```bash
ufw status verbose
```

如果显示：

```text
Status: active
Default: deny (incoming)
```

需要放行 Shadowsocks 端口。

TCP：

```bash
ufw allow 14449/tcp
```

UDP：

```bash
ufw allow 14449/udp
```

重新加载：

```bash
ufw reload
```

检查：

```bash
ufw status
```

正常应出现：

```text
22/tcp       ALLOW IN    Anywhere
14449/tcp    ALLOW IN    Anywhere
14449/udp    ALLOW IN    Anywhere
```

不要删除：

```text
22/tcp
```

否则可能无法继续 SSH 登录 VPS。

---

# 十三、VPS 本机测试端口

执行：

```bash
nc -vz 127.0.0.1 14449
```

正常：

```text
Connection to 127.0.0.1 14449 port [tcp/*] succeeded!
```

如果这里失败：

```bash
/etc/init.d/shadowsocks status
```

以及：

```bash
ss -lntup | grep 14449
```

---

# 十四、测试服务器公网端口

退出 SSH：

```bash
exit
```

然后在自己的 Mac/Linux 电脑执行：

```bash
nc -vz YOUR_SERVER_IP 14449
```

正常：

```text
Connection to YOUR_SERVER_IP port 14449 [tcp/*] succeeded!
```

---

# 十五、Shadowrocket 配置

新建：

```text
类型：
Shadowsocks
```

服务器：

```text
YOUR_SERVER_IP
```

端口：

```text
14449
```

密码：

```text
YOUR_STRONG_PASSWORD
```

算法：

```text
aes-256-gcm
```

插件：

```text
none
```

混淆：

```text
none
```

注意：

```text
aes-256-gcm
```

不能错误选择为：

```text
aes-256-cfb
```

服务器与客户端加密方式必须完全一致。

---

# 十六、如果 Shadowrocket 显示超时

首先确认服务：

```bash
/etc/init.d/shadowsocks status
```

然后：

```bash
ss -lntup | grep 14449
```

然后：

```bash
ufw status verbose
```

服务器本机测试：

```bash
nc -vz 127.0.0.1 14449
```

本地电脑测试：

```bash
nc -vz YOUR_SERVER_IP 14449
```

---

# 十七、使用 tcpdump 抓包

在 VPS 执行：

```bash
tcpdump -ni any port 14449
```

然后在 Shadowrocket 中测试一次节点。

如果服务器出现：

```text
IP CLIENT_IP.xxxxx > SERVER_IP.14449
```

说明客户端流量已经到达服务器。

结束抓包：

```text
Ctrl + C
```

---

# 十八、TCP 能连但 CONNECT 不通

先停止后台 Shadowsocks：

```bash
/etc/init.d/shadowsocks stop
```

前台运行：

```bash
ssserver -c /etc/shadowsocks.json
```

然后用 Shadowrocket 测试。

这样如果出现：

```text
Python Traceback
OpenSSL Error
libcrypto Error
```

可以直接看到错误。

停止前台服务：

```text
Ctrl + C
```

重新后台启动：

```bash
/etc/init.d/shadowsocks start
```

---

# 十九、常用管理命令

启动：

```bash
/etc/init.d/shadowsocks start
```

停止：

```bash
/etc/init.d/shadowsocks stop
```

重启：

```bash
/etc/init.d/shadowsocks restart
```

状态：

```bash
/etc/init.d/shadowsocks status
```

查看配置：

```bash
cat /etc/shadowsocks.json
```

查看监听：

```bash
ss -lntup | grep 14449
```

查看防火墙：

```bash
ufw status verbose
```

本机端口测试：

```bash
nc -vz 127.0.0.1 14449
```

公网端口测试：

```bash
nc -vz YOUR_SERVER_IP 14449
```

抓包：

```bash
tcpdump -ni any port 14449
```

查看 CPU：

```bash
top
```

退出：

```text
q
```

查看高 CPU 进程：

```bash
ps aux --sort=-%cpu | head -15
```

---

# 二十、最简完整命令流程

如果已经知道整个过程，可以按下面顺序操作。

## 1. 安装依赖

```bash
apt update
apt install -y wget gcc g++ autoconf automake make build-essential tcpdump netcat-openbsd
```

## 2. 下载脚本

```bash
cd /root
wget https://raw.githubusercontent.com/sucong426/VPN/main/ss.sh
chmod +x ss.sh
```

## 3. 安装

```bash
bash ss.sh
```

安装时设置：

```text
Port：14449
Method：aes-256-gcm
Password：自己设置
```

## 4. Python 3.14 修复

```bash
LRCACHE=$(find /usr/local/lib -path '*shadowsocks*' -name 'lru_cache.py' | head -1)
```

```bash
sed -i 's/collections.MutableMapping/collections.abc.MutableMapping/g' "$LRCACHE"
```

```bash
grep -q '^import collections\.abc' "$LRCACHE" || \
sed -i '/^import collections$/a import collections.abc' "$LRCACHE"
```

## 5. 验证版本

```bash
python3 --version
ssserver --version
```

## 6. 启动服务

```bash
/etc/init.d/shadowsocks restart
```

## 7. 放行防火墙

```bash
ufw allow 14449/tcp
ufw allow 14449/udp
ufw reload
```

## 8. 检查状态

```bash
/etc/init.d/shadowsocks status
```

```bash
ss -lntup | grep 14449
```

```bash
ufw status
```

## 9. 本机测试

```bash
nc -vz 127.0.0.1 14449
```

## 10. 外部测试

在自己的电脑：

```bash
nc -vz YOUR_SERVER_IP 14449
```

---

# 二十一、最终检查清单

```text
[ ] Ubuntu 系统更新完成
[ ] Python 3.14 正常
[ ] Shadowsocks-Python 3.0.0 安装完成
[ ] MutableMapping 兼容问题已修复
[ ] /etc/shadowsocks.json 配置正确
[ ] Shadowsocks 服务正在运行
[ ] TCP 14449 正在监听
[ ] UDP 14449 正在监听
[ ] UFW 已放行 TCP 14449
[ ] UFW 已放行 UDP 14449
[ ] Vultr Cloud Firewall 已放行（如果启用）
[ ] VPS 本机 nc 测试成功
[ ] 外部 nc 测试成功
[ ] Shadowrocket 地址正确
[ ] Shadowrocket 端口正确
[ ] Shadowrocket 密码正确
[ ] Shadowrocket 算法为 aes-256-gcm
[ ] 插件为 none
[ ] 混淆为 none
```

---


# 二十二、版本说明

下面这个路径：

```text
/usr/local/lib/python3.14/dist-packages/shadowsocks-3.0.0-py3.14.egg/
```

其中：

```text
python3.14
```

代表 Python 版本。

```text
shadowsocks-3.0.0
```

代表 Shadowsocks-Python 软件版本。

```text
py3.14
```

表示该软件安装在 Python 3.14 环境中。

因此：

```text
Python 3.14
```

和：

```text
Shadowsocks 3.0.0
```

不是同一个版本号。

---

## 说明

Shadowsocks-Python 3.0.0 属于较老的软件实现，在现代系统中还可能遇到：

```text
Python API 兼容问题
OpenSSL 3 兼容问题
旧依赖失效
init 脚本兼容问题
```

已有环境可以继续使用；如果重新部署长期运行的服务器，更建议考虑维护活跃的 Shadowsocks 实现，例如：

```text
shadowsocks-rust
```
