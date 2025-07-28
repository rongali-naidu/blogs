# From SQL Logs to Packet Captures: My First Network Debugging to Trace Lost DB Connections

## Introduction

### Context

Recently, I transitioned into a Database Engineer role, and one of my first challenges was debugging a tricky issue in the **integration layer**—the component responsible for fetching data from **SQL Server** and feeding it to the **frontend UI applications**.

The logs revealed **500 errors** and **timeout errors**, but the root cause wasn’t immediately clear. A timeout could mean:

* An **SQL timeout error**
* A **Lambda timeout** (if serverless functions are involved)
* Or something else entirely—**any component between the frontend and the database**, including **network infrastructure**, could be timing out.

### It wasn’t easy.

I had to connect **multiple concepts**:

* How do clients actually connect to the database?
* How can we **track incoming connections** to SQL Server?
* How can we determine if a **query is reaching the database at all**?

### In our case...

With the help of **SQL Server’s system tables and utilities**, we confirmed that **no client-related SQL queries** were even reaching the database during the error periods.

This shifted our focus: the **problem was not inside SQL Server**, but **somewhere between the client and the database**.

---

### **The Real Challenge? Understanding What’s Actually Happening on the Network.**

To solve this, we had to go **beyond the application and database layers** and start exploring the **network level**.

That’s where **packet tracing and analysis tools** come in—they allow you to **see the actual data traffic** between systems and understand whether:

* Connections are being established.
* Requests are being sent.
* Responses are being received.
* Or packets are being dropped, delayed, or blocked.
* came to know about tcpdump, pktmon, and Wireshark tools for this analysi



##  **What Are tcpdump, pktmon, and Wireshark?**

| Tool          | Description                                                                                                                                      | Interface |
| ------------- | ------------------------------------------------------------------------------------------------------------------------------------------------ | --------- |
| **tcpdump**   | A **command-line tool** to capture and display raw network packets in real time. Lightweight and used commonly on **Linux/Unix systems**.        | CLI       |
| **pktmon**    | A **Windows built-in tool** for capturing and monitoring network traffic and packet flow, especially useful for **Windows environments**.        | CLI       |
| **Wireshark** | A **graphical tool** that visually shows network traffic and allows deep **protocol inspection**. Excellent for learning and detailed debugging. | GUI       |


---

## Understanding TCP Connection Handshake

A successful TCP connection involves a three-step handshake:

1. **SYN** — Client requests to start communication.
2. **SYN-ACK** — Server acknowledges and responds.
3. **ACK** — Client confirms and connection is established.

For secure connections (e.g., to a DB over TLS):

* **ClientHello** is sent post-handshake.
* **ServerHello + Certificate** follows.
* Final **Finished** signals mark encrypted session establishment.

If **SYN or SYN-ACK fails**, the handshake collapses, causing dial timeouts.

---

## Visual: TCP and TLS Flow Diagram

```
Client                   Server
  | ----- SYN ------>     |
  | <---- SYN-ACK ----    |
  | ----- ACK ------>     |   <-- TCP Connection Established
  |                       |
  | -- ClientHello --->   |
  | <- ServerHello + Cert |
  | -- KeyExchange -->    |
  | -- Finished ------>   |
  | <- Finished --------  |   <-- TLS Secure Session Established
  |                       |
  | ----> Data Packets    |
  | <---- Data Packets    |
  | ---- FIN, ACK ---->   |
```

---

## Tools We Used for Diagnosis

### Linux: tcpdump

```bash
sudo tcpdump -i eth0 -w output_file.pcap
```

* Captures packets on `eth0` until interrupted.
* Output can be analyzed using **Wireshark**.

### Windows: pktmon

```cmd
pktmon start -c
pktmon stop
pktmon etl2pcap PktMon.etl -o testpcap.pcap
```

* Lightweight packet capture tool built into Windows.
* Converts to `.pcap` for Wireshark analysis.

---

## What We Found

Using packet captures, we compared **successful** and **failed** connection attempts to the database.

### Successful Request Flow:

1. **SYN → SYN-ACK → ACK** — TCP connection established.
2. **TLS Handshake** — ClientHello, ServerHello, Certificate.
3. **Query Execution** — Application data flows (PSH flag).
4. **Connection Closure** — FIN, ACK.

### Failed Request Flow:

* **SYN sent** but **no SYN-ACK reply** from DB.
* TLS handshake never started.
* Client retried, still no connection.
* Result: **Dial timeout** and service returned **500 Error**.

---

## Root Cause Analysis

Two main contributors to the dial timeouts:

1. **Listener IP Failover Behavior**

   * Our DB cluster’s **listener DNS** resolved to multiple IPs.
   * During failovers, some IPs remained in DNS even though their respective DB nodes were unavailable.
   * Clients trying these inactive IPs hit dial timeouts.

2. **No TCP Response from Target IP**

   * Packet captures confirmed SYN was sent, but **no SYN-ACK** was returned from certain IPs.
   * This led to connection timeouts after retries.

---

## Why TCP Dial Timeouts Happen – Broader Causes

* **Firewall/Security Groups** blocking inbound SYN packets.
* **Port not listening** on the target server.
* **Network latency/congestion** causing packet loss.
* **Client-side port exhaustion** or network stack issues.
* **DNS resolution delays or stale entries**.

---

## Types of Timeouts: Clarifying Dial Timeout vs Command Timeout

| Timeout Type        | Cause                                                                                                       |
| ------------------- | ----------------------------------------------------------------------------------------------------------- |
| **Dial Timeout**    | Client cannot establish a **TCP connection** (SYN/SYN-ACK failure).                                         |
| **Command Timeout** | TCP connection established, but **query execution delayed** (e.g., long-running queries, blocked sessions). |

**Key Insight**: In our issue, **no TCP connection** was made — it failed **before the handshake**, hence a **dial timeout**.

---

## Client Timeout Settings

| Client         | Setting              | Default Value | Applies To                   |
| -------------- | -------------------- | ------------- | ---------------------------- |
| ADO.NET / .NET | `Connection Timeout` | 15 seconds    | TCP handshake (dial timeout) |
| ADO.NET / .NET | `Command Timeout`    | 30 seconds    | Query execution              |
| JDBC           | `connectTimeout`     | Varies        | TCP connection time          |
| JDBC           | `socketTimeout`      | Varies        | Query execution + response   |

---

## Network Timeout Diagnosis Checklist

| Step                         | Tool/Method                  | Interpretation                             |
| ---------------------------- | ---------------------------- | ------------------------------------------ |
| Check DNS Resolution         | `dig`, `nslookup`            | Slow/stale DNS could delay connection      |
| Monitor SYN/SYN-ACK exchange | `tcpdump`, `Wireshark`       | No SYN-ACK = Target not reachable          |
| Confirm Port Listening       | `ss -tulnp`, `netstat -an`   | Target port open? Firewall blocking?       |
| Monitor Query Execution Time | SQL Logs, Profiler           | Helps isolate command timeouts             |
| Retry Patterns / Failures    | Client logs, Packet captures | Repeated failures = failover or load issue |

---

## Best Practices for Connection Resiliency

* **Connection Pooling**: Reuse existing connections.
* **Exponential Backoff**: Implement smarter retry strategies.
* **DNS TTL Management**: Set appropriate TTLs for failover-sensitive services.
* **Health Checks**: Remove unhealthy IPs from DNS rotation dynamically.
* **Packet Monitoring**: Detect SYN failures early via network tools.


## Command Cheatsheet

| Tool             | Platform  | Command                                | Purpose                     |
| ---------------- | --------- | -------------------------------------- | --------------------------- |
| `tcpdump`        | Linux     | `sudo tcpdump -i eth0 -w capture.pcap` | Capture packets             |
| `pktmon`         | Windows   | `pktmon start -c`, `etl2pcap`          | Packet logging              |
| `wireshark`      | All       | Open `.pcap` file                      | Visual analysis             |
| `ss`             | Linux     | `ss -tulnp`                            | List open ports and sockets |
| `netstat`        | Win/Linux | `netstat -an`                          | Show connection states      |
| `dig`/`nslookup` | All       | `dig yourdbhost.com`                   | Check DNS resolution        |

---

## Resources for Deeper Learning

* [SQL Server: Timeout expired errors](https://learn.microsoft.com/en-us/troubleshoot/sql/database-engine/connect/timeout-expired-error)
* [Connection Pooling (ADO.NET)](https://learn.microsoft.com/en-us/dotnet/framework/data/adonet/sql-server-connection-pooling)
* [SQL Server Network Configuration](https://learn.microsoft.com/en-us/sql/database-engine/configure-windows/configure-sql-server-network-configuration)
* [TCP/IP Illustrated](https://www.amazon.com/TCP-IP-Illustrated-Volume-Addison-Wesley/dp/0201633469)
* [Wireshark 101 – YouTube](https://www.youtube.com/watch?v=TkCSr30UojM)
* [RedHat tcpdump Guide](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/8/html/monitoring_and_automation/tcpdump_monitoring-and-automation)
* [TLS Handshake Overview](https://www.cloudflare.com/learning/ssl/what-happens-in-a-tls-handshake/)



