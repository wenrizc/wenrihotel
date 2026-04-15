## 1. 文件与目录（定位、复制、清理）

- 查看与定位：`pwd`、`ls -lh`、`tree`
- 目录操作：`cd`、`mkdir -p`、`rmdir`
- 复制移动：`cp -r`、`mv`
- 删除：`rm -rf`（谨慎）

```bash
ls -lh
mkdir -p /data/app/logs
du -sh /data/app
df -h
```

## 2. 查找与权限（线上排查高频）

- 查找文件：`find`、`locate`
- 权限与所有者：`chmod`、`chown`

```bash
find . -name "*.log" -mtime -1
chmod 644 app.log
chown -R app:app /data/app
```

## 3. 文本与日志（看日志的正确姿势）

- 查看：`cat`（小文件）、`less`（大文件）、`head`、`tail -f`
- 过滤：`grep -n`、`grep -r`
- 文本处理：`awk`、`sed`、`sort`、`uniq`、`xargs`

```bash
tail -f app.log
grep -n "ERROR" app.log | head
awk '{print $1}' access.log | sort | uniq -c | sort -nr | head
```

## 4. 进程与资源（CPU/内存/磁盘）

- 进程：`ps aux`、`top/htop`、`kill -9`
- 资源：`free -h`、`vmstat`、`iostat`（需工具包）、`uptime`

```bash
ps aux | grep java
top -Hp <pid>
free -h
```

## 5. 网络与诊断（连通性与端口排查）

- 连通性：`ping`、`traceroute`
- HTTP 调试：`curl -v`、`wget`
- 端口与连接：`ss -lntp`、`lsof -i`

```bash
curl -v https://example.com
ss -lntp | grep 8080
lsof -i :8080
```

## 6. 服务管理与系统日志

- `systemctl`：启动/停止/重启/查看状态
- `journalctl`：查看 systemd 管理的日志

```bash
systemctl status nginx
journalctl -u nginx --since "1 hour ago"
```

## 7. 调试与抓包

当“看日志看不出来、但就是慢/卡/连不上”时，常用的进阶工具有：

- `strace`：跟踪系统调用，判断卡在网络、磁盘还是锁上。
- `tcpdump`：抓包看三次握手、重传、RST、TLS 握手等。
- `dig`/`nslookup`：排查 DNS 解析与缓存问题。

```bash
# 跟踪某个进程的系统调用（谨慎使用，可能有开销）
strace -p <pid> -f -tt -T

# 抓 8080 端口的 TCP 流量（示例）
tcpdump -i any tcp port 8080 -nn -vv

# DNS 查询
dig example.com
```
