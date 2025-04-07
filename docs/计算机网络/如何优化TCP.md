
#### 一、背景与目标
- **TCP 特性**：面向连接、可靠、双向传输的协议，性能优化依赖操作系统内核参数调整。
- **优化方向**：
  1. 三次握手：减少建立连接延迟。
  2. 四次挥手：加快连接关闭效率。
  3. 数据传输：提升吞吐量和资源利用率。
- **前提**：理解 TCP 状态变迁，针对问题对症下药。

---

#### 二、TCP 三次握手的性能提升
##### 1. 三次握手过程
- **目的**：同步序列号（SYN），确保可靠传输。
- **状态变迁**：
  - 客户端：`SYN_SENT` → `ESTABLISHED`。
  - 服务端：`SYN_RCV` → `ESTABLISHED`。
- **影响**：占 HTTP 请求 10% 以上时间，高并发或网络不佳时更显著。

##### 2. 客户端优化
- **SYN_SENT 优化**：
  - **问题**：未收到 `SYN+ACK`，需重传 `SYN`。
  - **参数**：`tcp_syn_retries`（默认 5 次，重传间隔 1、2、4、8、16 秒，总计 63 秒）。
  - **策略**：根据网络稳定性调整（如内网调低至 2-3 次）。
  - **调整**：
    ```bash
    sysctl -w net.ipv4.tcp_syn_retries=3
    ```

##### 3. 服务端优化
- **SYN_RCV 优化**：
  - **半连接队列**：存储未完成握手的连接。
    - **问题**：队列满（如 SYN 攻击）导致新连接丢弃。
    - **查看**：`netstat -s | grep "SYNs to LISTEN sockets dropped"`。
    - **调整**：
      - `tcp_max_syn_backlog`：半连接队列大小（默认 256）。
      - `somaxconn`：accept 队列上限（默认 128）。
      - `backlog`：`listen()` 参数。
      - 示例：
        ```bash
        sysctl -w net.ipv4.tcp_max_syn_backlog=1024
        sysctl -w net.core.somaxconn=1024
        # Nginx 配置：listen 80 backlog=1024;
        ```
    - **syncookies**：队列满时仍建立连接。
      - **原理**：用 Cookie 验证，避免队列依赖。
      - **参数**：`tcp_syncookies`（0 关闭，1 队列满启用，2 常开）。
      - **建议**：设为 1 应对 SYN 攻击。
        ```bash
        sysctl -w net.ipv4.tcp_syncookies=1
        ```
  - **全连接队列**：存储已完成握手的连接。
    - **问题**：队列满（`min(somaxconn, backlog)`）丢弃连接。
    - **查看**：`ss -ltn`（Recv-Q 当前大小，Send-Q 最大长度）；`netstat -s | grep "times the listen queue of a socket overflowed"`。
    - **调整**：
      - `tcp_abort_on_overflow`：0（丢弃 ACK），1（发 RST）。
      - **建议**：设为 0 应对突发流量。
        ```bash
        sysctl -w net.ipv4.tcp_abort_on_overflow=0
        ```
    - **重传**：`tcp_synack_retries`（默认 5 次，63 秒）。
      ```bash
      sysctl -w net.ipv4.tcp_synack_retries=3
      ```

##### 4. 绕过三次握手
- **TCP Fast Open (TFO)**：
  - **原理**：首次握手缓存 Cookie，后续 SYN 携带数据。
  - **效果**：减少 1 RTT。
  - **条件**：Linux 3.7+，双方支持。
  - **参数**：`tcp_fastopen`（0 关闭，1 客户端，2 服务端，3 两者）。
    ```bash
    sysctl -w net.ipv4.tcp_fastopen=3
    ```

---

#### 三、TCP 四次挥手的性能提升
##### 1. 四次挥手过程
- **报文**：FIN（结束），ACK（确认）。
- **状态变迁**：
  - 主动方：`FIN_WAIT1` → `FIN_WAIT2` → `TIME_WAIT` → `CLOSED`。
  - 被动方：`CLOSE_WAIT` → `LAST_ACK` → `CLOSED`。
- **特点**：主动方有 `TIME_WAIT`（默认 60 秒）。

##### 2. 主动方优化
- **关闭方式**：
  - `close()`：完全关闭，生成孤儿连接。
  - `shutdown()`：半关闭（`SHUT_WR` 发送 FIN）。
- **FIN_WAIT1 优化**：
  - **问题**：未收到 ACK，重传 FIN。
  - **参数**：`tcp_orphan_retries`（默认 8 次）。
    ```bash
    sysctl -w net.ipv4.tcp_orphan_retries=4
    ```
  - **应对攻击**：`tcp_max_orphans`（默认 65536），超限发 RST。
    ```bash
    sysctl -w net.ipv4.tcp_max_orphans=131072
    ```
- **FIN_WAIT2 优化**：
  - **问题**：孤儿连接等待 FIN。
  - **参数**：`tcp_fin_timeout`（默认 60 秒）。
    ```bash
    sysctl -w net.ipv4.tcp_fin_timeout=30
    ```
- **TIME_WAIT 优化**：
  - **作用**：防止历史数据干扰，确保被动方关闭。
  - **时长**：2MSL（Linux 默认 MSL=30s，总计 60s）。
  - **策略**：
    1. **限制数量**：`tcp_max_tw_buckets`（默认 180000）。
       ```bash
       sysctl -w net.ipv4.tcp_max_tw_buckets=360000
       ```
    2. **复用**：`tcp_tw_reuse`（仅客户端，需 `tcp_timestamps=1`）。
       ```bash
       sysctl -w net.ipv4.tcp_tw_reuse=1
       sysctl -w net.ipv4.tcp_timestamps=1
       ```
    3. **跳过**：设置 `SO_LINGER`（`l_onoff=1, l_linger=0`），发 RST，仅限客户端。

##### 3. 被动方优化
- **CLOSE_WAIT 优化**：
  - **问题**：未调用 `close()`，需排查应用。
- **LAST_ACK 优化**：
  - **问题**：未收到 ACK，重传 FIN。
  - **参数**：`tcp_orphan_retries`。
- **特殊情况**：双方同时关闭，进入 `CLOSING` 状态。

---

#### 四、TCP 数据传输的性能提升
##### 1. 滑动窗口
- **作用**：控制发送速率，匹配接收方能力。
- **字段**：TCP 头部窗口字段（16位，最大 64KB）。
- **扩展**：`tcp_window_scaling=1`，窗口因子达 2^14，最大 1GB。
  ```bash
  sysctl -w net.ipv4.tcp_window_scaling=1
  ```

##### 2. 带宽时延积 (BDP)
- **定义**：网络中飞行报文最大字节数（带宽 × RTT）。
- **示例**：100MB/s × 10ms = 1MB。
- **关系**：
  - 缓冲区 > BDP：丢包。
  - 缓冲区 < BDP：带宽未充分利用。

##### 3. 缓冲区调整
- **发送缓冲区**：`tcp_wmem`（最小、默认、最大）。
  - 默认：`4096 16384 4194304`。
  - 调整：
    ```bash
    sysctl -w net.ipv4.tcp_wmem="4096 16384 8388608"
    ```
- **接收缓冲区**：`tcp_rmem`。
  - 默认：`4096 87380 6291456`。
  - 调整：
    ```bash
    sysctl -w net.ipv4.tcp_rmem="4096 87380 12582912"
    ```
  - 动态调节：`tcp_moderate_rcvbuf=1`。
    ```bash
    sysctl -w net.ipv4.tcp_moderate_rcvbuf=1
    ```
- **内存限制**：`tcp_mem`（页面数，低、高、最大）。
  - 默认根据系统内存计算。
  - 调整：
    ```bash
    sysctl -w net.ipv4.tcp_mem="88560 118080 177120"
    ```
- **注意**：避免在 socket 设置 `SO_SNDBUF`/`SO_RCVBUF`，关闭动态调整。

##### 4. 策略
- **高并发**：缓冲区上限 ≈ BDP，最小值保持 4K。
- **内存紧张**：调低默认值。
- **IO型服务器**：增大 `tcp_mem`。

---

#### 五、小结
| 阶段         | 关键参数                          | 优化目标                  |
|--------------|-----------------------------------|--------------------------|
| 三次握手     | `tcp_syn_retries`, `tcp_max_syn_backlog`, `tcp_syncookies`, `tcp_fastopen` | 减少延迟，抗攻击         |
| 四次挥手     | `tcp_orphan_retries`, `tcp_fin_timeout`, `tcp_max_tw_buckets`, `tcp_tw_reuse` | 加速关闭，减少资源占用   |
| 数据传输     | `tcp_wmem`, `tcp_rmem`, `tcp_window_scaling`, `tcp_mem` | 提升吞吐量，优化内存     |
