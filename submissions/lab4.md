# Lab 4 submission

## Task 1: Trace a Request End-to-End

I used WSL2 Ubuntu on Windows for the Linux networking tools.

The packet capture is included in:

- `submissions/lab4-trace.pcap`
- `submissions/lab4-trace.txt`

### TCP handshake

The connection starts with the normal TCP three-way handshake.

```text
127.0.0.1.46700 > 127.0.0.1.8080: Flags [S]
127.0.0.1.8080 > 127.0.0.1.46700: Flags [S.]
127.0.0.1.46700 > 127.0.0.1.8080: Flags [.]
```

The client sends SYN, the server answers with SYN/ACK, and the client confirms with ACK.

### HTTP request

```text
POST /notes HTTP/1.1
Host: 127.0.0.1:8080
User-Agent: curl/8.5.0
Accept: */*
Content-Type: application/json
Content-Length: 39

{"title":"trace me","body":"in flight"}
```

### HTTP response

```text
HTTP/1.1 201 Created
Content-Type: application/json
Content-Length: 93

{"id":6,"title":"trace me","body":"in flight","created_at":"2026-09-18T08:36:28.371985059Z"}
```

The server returned `201 Created`, so the POST completed successfully.

### Connection close

```text
127.0.0.1.46700 > 127.0.0.1.8080: Flags [F.]
127.0.0.1.8080 > 127.0.0.1.46700: Flags [F.]
127.0.0.1.46700 > 127.0.0.1.8080: Flags [.]
```

Both sides closed the TCP connection normally with FIN packets.

## Five debugging commands

### 1. Listening socket

Command:

```text
ss -tlnp | grep :8080
```

Output:

```text
LISTEN 0 4096 *:8080 *:* users:(("quicknotes",pid=2812,fd=3))
```

This shows that QuickNotes is listening on TCP port 8080.

### 2. Routes

Command:

```text
ip route show
```

Output:

```text
default via 172.22.144.1 dev eth0 proto kernel
172.22.144.0/20 dev eth0 proto kernel scope link src 172.22.146.20
```

WSL has a default route through its virtual network interface.

### 3. Reachability

Command:

```text
mtr -rwc 5 localhost
```

Output:

```text
HOST: DESKTOP-4QJC5AV Loss% Snt Last Avg Best Wrst StDev
1.|-- localhost          0.0%   5  0.1 0.1 0.0 0.1 0.0
```

There was no packet loss to localhost.

### 4. DNS

Command:

```text
dig +short example.com @1.1.1.1
```

Output:

```text
172.66.147.243
104.20.23.154
```

DNS resolution through 1.1.1.1 worked.

### 5. Logs

Command:

```text
journalctl --user -u quicknotes -n 20
```

Output:

```text
-- No entries --
```

There are no systemd user service entries because QuickNotes was started directly with `go run .`.

## What I would check first for a 502

If QuickNotes returned 502 through a proxy, I would first check whether the application is actually listening on the expected port with `ss -tlnp`. Then I would call `/health` directly on port 8080 to bypass the proxy. If the direct request failed, I would check the process and application logs. If the direct request worked, I would check the proxy upstream address, port, DNS, and firewall rules between the proxy and QuickNotes.

## Task 2: Outside-In Debugging

### Broken deployment

I started two QuickNotes instances with the same address:

```text
ADDR=:8080 go run .
ADDR=:8080 go run .
```

The second process failed with:

```text
2026/09/18 11:31:55 quicknotes listening on :8080 (notes loaded: 5)
2026/09/18 11:31:55 listen: listen tcp :8080: bind: address already in use
exit status 1
```

The root cause was that TCP port 8080 was already owned by the first QuickNotes instance.

### 1. Process check

Command:

```text
ps -ef | grep "go run" | grep -v grep
ps -ef | grep quicknotes | grep -v grep
```

Output:

```text
root 2837 274 3 11:31 pts/0 00:00:00 go run .
root 2958 2837 0 11:31 pts/0 00:00:00 quicknotes
```

Decision: one QuickNotes instance was still running.

### 2. Listening socket

Command:

```text
ss -tlnp | grep 8080
```

Output:

```text
LISTEN 0 4096 *:8080 *:* users:(("quicknotes",pid=2958,fd=3))
```

Decision: port 8080 was already occupied.

### 3. HTTP reachability

Command:

```text
curl -s -o /dev/null -w "%{http_code}\n" http://localhost:8080/health
```

Output:

```text
200
```

Decision: the first instance was healthy and reachable.

### 4. Firewall

Command:

```text
iptables -L -n -v
```

Output:

```text
Chain INPUT (policy ACCEPT)
Chain FORWARD (policy ACCEPT)
Chain OUTPUT (policy ACCEPT)
```

Decision: the firewall was not blocking the local request.

### 5. DNS

Command:

```text
dig +short localhost
```

Output:

```text
127.0.0.1
```

Decision: localhost resolved correctly.

## Repair

I stopped the conflicting processes and started one clean QuickNotes instance.

Command:

```text
ADDR=:8080 go run .
curl -s http://localhost:8080/health
```

Output:

```json
{"notes":5,"status":"ok"}
```

The service was healthy again and listening on port 8080.

## Mini-postmortem

The failure was caused by two application instances trying to own the same TCP port. The operating system behaved correctly by allowing the first listener and rejecting the second one. The systemic problem is that the deployment process did not check port ownership before starting another instance. A service manager or container orchestrator could prevent this by controlling the number of processes and their ports. Startup health checks and clear bind-error monitoring would also make the problem visible immediately. A deployment should verify that the new process is healthy before replacing or restarting an existing healthy instance.

## Bonus

Not attempted.