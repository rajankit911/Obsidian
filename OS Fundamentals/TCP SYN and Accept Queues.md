---
tags:
  - os
  - networking
  - tcp
  - performance-tuning
date_created: 2026-06-13
last_modified: 2026-06-13
---
# Overview

Establishing a TCP connection requires a three-way handshake:

1. The client sends a **SYN** (synchronize) packet to the server, indicating a desire to establish a connection.
2. The server responds with a **SYN-ACK** (synchronize-acknowledge) packet, indicating a willingness to establish a connection.
3. The client replies with an **ACK** (Acknowledge) packet, indicating that it has received the SYN-ACK message and the connection has been established.

> [!INFO] Queue Roles
> Due to network latency, the kernel maintains a **SYN queue** to store connection information (TCP 4-tuple, MSS, window scale) while waiting for the client's final ACK.
> Once the ACK is received, the connection moves to the **Accept queue**, decoupling the network layer from the application layer until `accept()` is called.

![[tcp_handshake_queues.svg]]

# Queue Overflows

If the client continuously sends SYN packets without completing the handshake (e.g., SYN flood) or if the server application delays calling `accept()`, these queues can overflow.

## SYN Queue Overflow

The server's behavior when the SYN queue overflows is mainly determined by the `net.ipv4.tcp_syncookies` option.

- `net.ipv4.tcp_syncookies = 0`: Discard newly arrived SYN messages when the queue is full.
- `net.ipv4.tcp_syncookies = 1`: Formally enable the SYN Cookie mechanism only when the SYN queue is full.
- `net.ipv4.tcp_syncookies = 2`: Unconditionally enable the SYN Cookie mechanism.

> [!TIP] Best Practice
> It is generally suggested to only enable SYN Cookies when the SYN queue is full (`net.ipv4.tcp_syncookies=1`) because SYN Cookies drop TCP options like Window Scale (unless TCP Timestamps are used) and increase server CPU load due to Hash calculations.

### Checking SYN Queue Size and Overflow

Connections in the SYN queue are in the `SYN-RECEIVED` status. You can count them using:

```bash
sudo netstat -antp | grep SYN_RECV | wc -l
```

When SYN packets are dropped due to a full queue:

```bash
$ netstat -s | grep "LISTEN"
 <Number> SYNs to LISTEN sockets dropped
```

> [!NOTE] CentOS Behavior
> In CentOS, if `SYN Cookie` is set to 1, the dropped count will still increase if the SYN queue is full, helping evaluate if the queue needs expanding. If set to 2, the count has no meaning.

## Accept Queue Overflow

The server's behavior is determined by `net.ipv4.tcp_abort_on_overflow`.

- `0` (Default): The server directly discards the ACK packet of the third handshake and retransmits the SYN-ACK (up to `net.ipv4.tcp_synack_retries`). PSH packets carrying application data will also be dropped and retransmitted by the client (up to `net.ipv4.tcp_retries2`).
- `1`: The server directly sends an RST packet to the client to close the connection.

> [!TIP] Best Practice
> We generally recommend setting `net.ipv4.tcp_abort_on_overflow` to `0` unless you are certain the server cannot recover from heavy traffic quickly and wish to notify the client as soon as possible. Setting it to `0` is more advantageous for burst traffic.

### Checking Accept Queue Size and Overflow

Use the `ss` command to check the queue status for listening sockets (`Recv-Q` is the current size, `Send-Q` is the max size):

```bash
$ ss -lnt
LISTEN   0    1024   *:1883         *:*
```

When packets are dropped due to an Accept queue overflow:

```bash
$ netstat -s | grep "overflowed"
    <Number> times the listen queue of a socket overflowed
```

# Increasing the Size of the Queues

## Accept Queue

The maximum size is the smaller of the `backlog` parameter in the `listen()` function and the Linux kernel parameter `net.core.somaxconn`.

In EMQX, set the `backlog` for each listener:
```hocon
listeners.tcp.default {
  tcp_options {
    backlog = 1024
  }
}
```

## SYN Queue

The size is affected by parameters like `net.ipv4.tcp_max_syn_backlog` and `net.core.somaxconn`, varying by OS.

**CentOS Calculation Formula:**

```
Max SYN Queue Size = roundup_pow_of_two(
  max(min(somaxconn, backlog, sysctl_max_syn_backlog), 8) + 1)
```

**Ubuntu Calculation Formula:**

```
Max SYN Queue Size = min(min(somaxconn, backlog), 0.75 * tcp_max_syn_backlog + 1)
```

> [!WARNING] Making Changes Permanent
> `sysctl -w` changes are temporary. Write them to `/etc/sysctl.conf` and run `sysctl -p` to make them permanent.
> ```ini
> net.core.somaxconn = 4096
> net.ipv4.tcp_max_syn_backlog = 4096
> ```

# Verify the Maximum Size of the SYN Queue

You can use a Python script and `hping3` to verify the SYN queue capacity.

**Server Script (`server.py`):**

```python
import socket
import time 

def start_server(host, port, backlog):
    server_socket = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    server_socket.bind((host, port))
    server_socket.listen(backlog)
    print(f'Server is listening on {host}:{port}')
    while True:
        time.sleep(3)

if __name__ == '__main__':
    start_server('0.0.0.0', 12345, 256)
```

**Server Preparation (Ubuntu example):
**
```bash
echo 0 > /proc/sys/net/ipv4/tcp_syncookies
echo 64 > /proc/sys/net/core/somaxconn
echo 512 > /proc/sys/net/ipv4/tcp_max_syn_backlog
sudo tc qdisc add dev eth0 root netem delay 200ms
python3 ./server.py
```

**Client Test (`hping3`):**

```bash
apt-get install hping3
# -S, send SYN packet
# -p <Port>, specify the port
# -c <Count>, the number of packets sent
# -i u100, send packets at intervals of 100us
hping3 -S -p 12345 -c 65 -i u100 <Your Hostname>
```

> [!INFO] Conclusion
> A very large accept queue is not always good. Setting reasonable size limits is a protection for the server during flood attacks or sudden traffic spikes.

## References
- [EMQX Performance Tuning: TCP SYN Queue and Accept Queue](https://www.emqx.com/en/blog/emqx-performance-tuning-tcp-syn-queue-and-accept-queue)
- ![[EMQX Performance Tuning_ TCP SYN Queue and Accept Queue _ EMQ.pdf]]
