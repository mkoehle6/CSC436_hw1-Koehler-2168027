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
### Output That Would Prove Me wrong
```
 $ netstat -ano | findstr :8471

      TCP    0.0.0.0:8471           0.0.0.0:0              LISTENING       41768
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


