# CSC 436 - Homework 1 - Task 4
# Annotated Capture

 
|  |  |
| --- | :---: |
| File: | 04-capture.pcapng |
| Packets: | 21 |
| Bytes: | 7664 |
| Client: | 192.168.80.204 (RFC 1918 - this machine is behind NAT) |
| Target: | www.example.com |
| Streams: |  |


**Redaction:** None was needed. There were no cookies, `Authorization` headers, credentials, or public client addresses. The client address is private, and the destination is a public documentation host.


## Stage 1 - DNS

**Display filter:** dns

```
1	0.000000000	192.168.80.204	192.168.80.1	DNS	75	Standard query 0xf254 A www.example.com
2	0.000640100	192.168.80.1	192.168.80.204	DNS	107	Standard query response 0xf254 A www.example.com A 104.20.23.154 A 172.66.147.243

```
What the bytes say:
Packet 1 - the query
```
Transaction ID: 0xf254       <--- used by DNS to match the response to the query
Flags: 0x0100 Standard query
Queries: www.example.com: type A, class IN
```

Packet 2 - the response
```
Domain Name System (response)
    Transaction ID: 0xf254        <--- matches packet 1
    Flags: 0x8180 Standard query response, No error
    Questions: 1
    Answer RRs: 2    <-- www.example.com has two separate IP addresses.
    Queries
        www.example.com: type A, class IN
            Name: www.example.com
            [Name Length: 15]
            [Label Count: 3]
            Type: A (1) (Host Address)
            Class: IN (0x0001)
    Answers
        www.example.com: type A, class IN, addr 104.20.23.154  <-- address 1
            Name: www.example.com
            Type: A (1) (Host Address)
            Class: IN (0x0001)
            Time to live: 78 (1 minute, 18 seconds)
            Data length: 4
            Address: 104.20.23.154  
        www.example.com: type A, class IN, addr 172.66.147.243  <-- address 2
            Name: www.example.com
            Type: A (1) (Host Address)
            Class: IN (0x0001)
            Time to live: 78 (1 minute, 18 seconds)
            Data length: 4
            Address: 172.66.147.243
```
**What this proves:** DNS resolved www.example.com to two addresses. The response shows a
time to live of 78 seconds. RFC 1035 specifies TTL as an unsigned 32-bit integer, but
the TTL value is controlled by the DNS operator and may change over time.

**What this does not prove:** It does not prove that either address is reachable, or that the server will answer.

**How I would independently verify:** I used Google Admin Toolbox Dig, available at
[https://toolbox.googleapps.com/apps/dig/#A/](https://toolbox.googleapps.com/apps/dig/#A/)
```
id 52163
opcode QUERY
rcode NOERROR
flags QR RD RA
;QUESTION
www.example.com. IN A
;ANSWER
www.example.com. 300 IN A 104.20.23.154
www.example.com. 300 IN A 172.66.147.243
;AUTHORITY
;ADDITIONAL
```

## Stage 2 - TCP handshake
```
3	0.003279400	192.168.80.204	104.20.23.154	TCP	66	10604 → 443 [SYN] Seq=0 Win=65535 Len=0 MSS=1460 WS=256 SACK_PERM
4	0.018901500	104.20.23.154	192.168.80.204	TCP	66	443 → 10604 [SYN, ACK] Seq=0 Ack=1 Win=65535 Len=0 MSS=1400 SACK_PERM WS=8192
5	0.018958800	192.168.80.204	104.20.23.154	TCP	54	10604 → 443 [ACK] Seq=1 Ack=1 Win=65280 Len=0
```
**What this proves:** The three-way handshake was successfully completed between my machine
and the remote server. My machine initiated the connection by sending a SYN packet from
the operating-system-assigned port 10604. This is a dynamic port on my machine, as verified
with the command:
```
PS C:\Users\mlkoe> netsh.exe
netsh>int
netsh interface>ipv4
netsh interface ipv4>show dynamicport tcp

Protocol tcp Dynamic Port Range
---------------------------------
Start Port      : 1024
Number of Ports : 64511
```
The server responded with the SYN-ACK packet, and the connection is now identified by the four-tuple
192.168.80.204:10604 <-> 104.20.23.154:443.

When the connection is closed, the operating system frees port 10604 for use by another client. It is no longer
associated with this connection.

**What this does not prove:** There is no evidence that anything above the transport layer
is working. HTTP is an application-layer protocol. We can prove only that there is a connection
to the standard HTTPS port; we do not know whether the process bound to the socket is actually
a web server.

**How I would independently verify:**
```
PS C:\Users\mlkoe> curl.exe -sS -v https://example.com/ -o NUL
* Host example.com:443 was resolved.
* IPv6: (none)
* IPv4: 104.20.23.154, 172.66.147.243
*   Trying 104.20.23.154:443...
* schannel: disabled automatic use of client certificate
* ALPN: curl offers http/1.1
* ALPN: server accepted http/1.1
* Established connection to example.com (104.20.23.154 port 443) from 192.168.80.204 port 64619
* using HTTP/1.x
> GET / HTTP/1.1
> Host: example.com
> User-Agent: curl/8.21.0
> Accept: */*
>
* Request completely sent off
* schannel: remote party requests renegotiation
* schannel: renegotiating SSL/TLS connection
* schannel: SSL/TLS connection renegotiated
< HTTP/1.1 200 OK
< Date: Tue, 15 Sep 2026 23:29:15 GMT
< Content-Type: text/html
< Transfer-Encoding: chunked
< Connection: keep-alive
< Server: cloudflare
< last-modified: Tue, 15 Sep 2026 16:59:31 GMT
< allow: GET, HEAD
< Accept-Ranges: bytes
< Age: 2588
< cf-cache-status: HIT
< CF-RAY: a3bb6995e84ecb6b-EWR
<
{ [571 bytes data]
* Connection #0 to host example.com:443 left intact
```

## Stage 3 - TLS ClientHello
```