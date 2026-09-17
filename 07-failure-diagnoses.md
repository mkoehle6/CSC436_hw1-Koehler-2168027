# CSC 436 - Homework 1 - task 6
# DevTools HAR and waterfall

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


