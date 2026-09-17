# CSC 436 - Homework 1 - Task 9
# Reject an AI claim


Question asked to ChatGPT-5.6 Luna

```
will curl -v show me the negotiated TLS version?
```

The answer I received.
```
Yes. `curl -v` will normally show the negotiated TLS version in its verbose output.

 For example:

* SSL connection using TLSv1.3 / TLS_AES_256_GCM_SHA384


 Here, **TLS 1.3** is the negotiated TLS version, and `TLS_AES_256_GCM_SHA384` is the cipher suite.

 You can run:

curl -v https://example.com

 and look for a line containing `SSL connection using` or `TLS`.

 ```

 I am rejecting this claim because the output of `curl -v` does not always show the negotiated TLS version.  Lines 17-21 of
 artifact 05-curl-trace.txt from Task 5 show that the negotiated TLS version is not displayed in the verbose output of `curl -v`.
 ```
17 * schannel: renegotiating SSL/TLS connection
18 * schannel: SSL/TLS connection renegotiated
19 * schannel: remote party requests renegotiation
20 * schannel: renegotiating SSL/TLS connection
21 * schannel: SSL/TLS connection renegotiated
```

When I told the model it was wrong it corrected itself.
```
curl -v does not identify tls version.  it just says ssl/tls connection renegotiated.
```
```
You're right. With curl 8.21.0 built against Schannel, curl -v does not necessarily print the negotiated TLS version. My previous answer was too broad.

The message:

* schannel: SSL/TLS connection renegotiated

does not tell you whether the connection is TLS 1.2 or TLS 1.3. It indicates a TLS renegotiation event.
```
