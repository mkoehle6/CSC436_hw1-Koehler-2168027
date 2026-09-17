# CSC 436 - Homework 1 - task 7
# Diagnosing Network Failures

## Failure 01 - refused from the LAN

### Observation
Priya can access her CampusPulse web application from her computer; her friend, who is on the same LAN, cannot access it. When her friend tries to access the application, he receive a "refused" error message. Close examiniation of the startup log shows the application opened a socket on the localhost interface only at port 8471.
```
[2026-08-15T21:12:58.883Z] campuspulse v0.3.1 starting
[2026-08-15T21:12:58.901Z] listening on localhost:8471
[2026-08-15T21:12:58.901Z] ready
```
I infer that the application uses port 8471 to listen for incoming HTTP requests.  The netstat command filtered for port 8471 confirms that the application is only listening on the localhost interface.
```
C:\> netstat -ano | findstr :8471

  TCP    127.0.0.1:8471         0.0.0.0:0              LISTENING       41768
  ```
### Mechanism
The application is bound to the localhost interface, and will only accept connections that arrive on the loopback interface. When Priya's friend attempted to connect, the SYN packet arrived at her machine, and the operating system checked for a socket listening on port 8477 on the destination IP address. Because no application was listening on that port and interface, the operating system responded with a RST, ACK packet, indicating that the connection was refused.

### Root Cause
The application was configured to listen on the localhost interface only, which prevents it from accepting connections from other devices on the LAN.

### Cause Ruled Out
The host being down or unreachable was ruled out since the error message indicated conection refused, which means the host was reachable but no application was listening on the requested port.

### Command to Confirm
```
C:\> Get-NetTCPConnection -State Listen -LocalPort 8471
```

 ### Priya's Hypotheses
 None of Priya's hypotheses were correct. Line 12 of file 04-curl-from-dev.txt is the connection refused message.  If port 8471 was blocked, the packet would have been dropped. Additionally, WiFi client isolation prevents client computers from talking to each other by dropping packets.  Both would have resulted in timed out errors.
 Lastly, line 35 of 02-listening-sockets.txt shows the firewall is disabled.  It is not blocking port 8471. 

 A failed ping does not prove that the host is down or unreachable.  The ping command uses ICMP, which is a different protocol than TCP.  The host may be reachable but not responding to ICMP packets.
 The error message from line 12 of file 04-curl-from-dev.txt indicates that the host was reachable, but no application was listening on the requested port.  The operating system responded with a RST, ACK packet, indicating that the connection was refused.
 ```
 curl: (7) Failed to connect to 192.168.1.47 port 8471 after 3 ms: Connection refused
```
### The Fix
To fix the issue, Priya needs to configure the CampusPulse application to listen on all network interfaces.
Netstat command filtered for port 8471 after the fix would show that the application is now listening on all interfaces.
```
 $ netstat -ano | findstr :8471

      TCP    0.0.0.0:8471           0.0.0.0:0              LISTENING       41768
 ```
 Moving the port from 8471 to 80 would change nothing.  The socket would still be bound to the localhost interface and would not receive IP traffic.


 ## Failure 02 - Timeout from outside

 ### Observation
 Https requests from the public internet timeout with no response.  Https requests from the loopback and private address succeed.

 ### Mechanism
 SYN packets from the public internet do not reach graders machine. The packets are dropped somewhere between the public internet and the grader's machine.  The packets do not reach the grader's machine, so the operating system does not respond with a RST, ACK packet.  The client waits for a response until it times out. A connection refused would indicate the machine is at least reachable from the public internet.

 ### Root Cause
 Line 34 of file 05-security-group.json shows that rule 'allow-https` (Tcp/443) was removed on 8/13/2026 causing the security group to activly block incoming HTTPS traffic from the public internet.  

 ### Proof It Is Not Failure 01
 Line 20 from 02-curl-on-the-box.txt proves the application is listening on a non-loopback interface.
 ```
ubuntu@campuspulse-demo:~$ curl -sS --max-time 6 -o /dev/null \
    -w "%{http_code} in %{time_total}s\n" http://10.42.7.19:8080/healthz
200 in 0.004s
```
### Marcus's Hypotheses
The deprecated config key has nothing to do with the problem.  Deprecated means it will no longer be supported in future versions.  Line 4 of 04-container-log.txt clear states the config
key will be removed in the next version of the software, not that it has already been removed.
```
2026-08-16T03:00:12.004Z  WARN  config key "trustProxyHeaders" is deprecated and will be
2026-08-16T03:00:12.004Z  WARN  removed in v0.5. Use "proxy.trustForwardedFor" instead.
```
Additionally, the suggestion of binding the network interface to 0.0.0.0 is wrong as well. It is already bound to that address as Marcus states and is noted
at line 5 of 04-container-log.txt.

```
2026-08-16T03:00:12.019Z  INFO  campuspulse v0.4.2 listening on 0.0.0.0:8080
```
Lastly, DNS is not the problem either.  Line 8 from 01-curl-from-outside.txt shows that dns resolution is working correctly. 
```
* Host status.campuspulse-demo.example:443 was resolved.
```

### The Fix
The fix would be to undo the change to the security group and re-add the rule to allow incoming HTTPS traffic on port 443.
If the following curl command returns with something other that a timeout, it would prove that the machine is reachable from the internet.
```
$ curl -sS --max-time 30 -v https://status.campuspulse-demo.example/healthz
```