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

**What this does not prove:** It does not prove that either address is reachable or that the server will answer.

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
**Display filter:** tls.handshake.type == 1
```
6	0.021536200	192.168.80.204	104.20.23.154	TLSv1.3	516	Client Hello (SNI=www.example.com)
```
The record decoded:
```
Handshake Protocol: Client Hello
    Handshake Type: Client Hello (1)
    Length: 453
    Version: TLS 1.2 (0x0303)
    Random: ce1cea1e4f060f91f228080f88ddfa57dec95b07ec2cccb449de97157b5a4d93
    Session ID Length: 32
    Session ID: 170cc8454fbf571a24867de91092e9a4c874672d8e118019da6083a8d2f35f44
    Cipher Suites Length: 40
    Cipher Suites (20 suites)
    Compression Methods Length: 1
    Compression Methods (1 method)
    Extensions Length: 340
    Extension: server_name (len=20) name=www.example.com
    Extension: status_request (len=5)
    Extension: supported_versions (len=5) TLS 1.3, TLS 1.2
    Extension: signature_algorithms (len=26)
    Extension: session_ticket (len=0)
    Extension: supported_groups (len=8)
    Extension: ec_point_formats (len=2)
    Extension: application_layer_protocol_negotiation (len=11)
    Extension: key_share (len=208) x25519, secp256r1, secp384r1
    Extension: post_handshake_auth (len=0)
    Extension: extended_master_secret (len=0)
    Extension: renegotiation_info (len=1)
    Extension: psk_key_exchange_modes (len=2)
    [JA4: t13d2013h1_2b729b4bf6f3_e24568c0d440]
    [JA4_r: t13d2013h1_002f,0035,003c,003d,009c,009d,1301,1302,c009,c00a,c013,c014,c023,c024,c027,c028,c02b,c02c,c02f,c030_0005,000a,000b,000d,0017,0023,002b,002d,0031,0033,ff01_0804,0805,0806,0401,0501,0201,0403,0503,0203,0202,0601,0603]
    [JA3 Fullstring: 771,4866-4865-49196-49195-49200-49199-49188-49187-49192-49191-49162-49161-49172-49171-157-156-61-60-53-47,0-5-43-13-35-10-11-16-51-49-23-65281-45,29-23-24,0]
    [JA3: fae0e5d973c96ae1888b99538efa0363]
```
**What this proves:** The client is initiating a TLS handshake. This is the first part of
establishing a secure connection.
**What this does not prove:** The connection is secure. Only the client side has initiated
the handshake; the server still needs to respond and complete the negotiation.
**How I would independently verify:**
```
PS C:\Users\mlkoe> curl.exe -sS -o NUL -w "%{ssl_verify_result} %{http_version}\n" https://www.example.com
0 1.1
```
According to the curl manual, `ssl_verify_result = 0` means that certificate verification was successful.


## Stage 4 - http
**Display filter:** http.request

There are no packets matched with this display filter.

**What this proves:** http packets are being encrypted by TLS.

**What this does not prove:** The application layer is functioning properly.
