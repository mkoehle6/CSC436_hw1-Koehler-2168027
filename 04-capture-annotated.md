# CSC 436 - Homework 1 - Task 4
# Annotated Capture

 
|  |  |
| --- | :---: |
| File: | 04-capture.pcapng |
| Packets: |  |
| Bytes: |  |
| Client: | 192.168.80.204 (RFC 1918 - this machine is behind NAT |
| Target: | www.bmw.com |
| Streams: |  |


**Redaction:** none was needed. No cookies, no `Authorization` header, no credentials, no public client address. The client address is private and the destination is a public documentation host. 


## Stage 1 - DNS

**Display filter:** dns

```
1	0.000000000	192.168.80.204	192.168.80.1	DNS	75	Standard query 0xf254 A www.example.com
2	0.000640100	192.168.80.1	192.168.80.204	DNS	107	Standard query response 0xf254 A www.example.com A 104.20.23.154 A 172.66.147.243

```

