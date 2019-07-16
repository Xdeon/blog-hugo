---
title: "DTLS issues in OTP21"
date: 2019-05-23T16:41:28+08:00
draft: false
summary: ""
tags: ["erlang"]
comments: true
---

I was planning to continue with the implemenation of the Erlang CoAP server (as a hobby project maybe) and noticed some issues while exploring the DTLS features. 

<!--more--> 

First, Erlang (OTP21 for the time of testing) does not provide ciphers required by the DTLS-secured CoAP ([RFC7252 section 9.1](https://tools.ietf.org/html/rfc7252#section-9.1)). Forturnately, this issue has been addressed in OTP22 (See [this](https://bugs.erlang.org/browse/ERL-745)).

Second, once a DTLS server has been started, that is to say, some process owns the "listen" socket, then termination of that process will not bring down the socket as one may expect. This leads to interesting behaviour such as when you restart your DTLS server you noticed it failed with reason similar to `{error, {shutdown, closed}}`. I have done some preliminary research on this, and it seems the `dtls_listener_sup` supervisor does not shut down its `dtls_packet_demux` child properly when it should do so. To me the only fix  for now is to restart the `ssl` application every time before restarting the DTLS server, which apperently does not work if what you want is just to rescue the listener process from a crash. I filed a [bug report](https://bugs.erlang.org/browse/ERL-917) to the OTP team and hope they would dig into this problem with more details.