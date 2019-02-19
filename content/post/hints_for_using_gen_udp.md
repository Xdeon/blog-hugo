---
title: "Notes for using gen_udp"
date: 2017-06-29T22:04:44-06:00
draft: false
description: For my own notes
tags: ["erlang", "learning"]
---

When I was using [`gen_udp`](http://erlang.org/doc/man/gen_udp.html) implementing an application protocol, I struggled a lot to improve its performance in terms of throughput (requests processed per second) and latency. 

I was trying to build a server application where many concurrent clients send huge amount of requests via UDP while each of them may not send a lot. Regarding socket options, I noticed that people saying `{active, true}` gives best performance at the cost of potentially flooding a process. Since I did not have any security consideration under a test environment, I used to believe `{active, true}` fits better.

However after some googling I realized that, in an application where a single process holds the socket and dispatches incoming messages based on some identity of the worker processes, like the one I tried to implement, there's more to consider. I took this structure simply because I was using the connectionless UDP and erlang only allows the controlling process of a socket to receive packets if the socket was configured to use active mode, which gives me no other obvious choice if I want to handle packets in other processes (no concurrent acceptor processes like TCP). I could not use multiple UDP sockets either because only one open socket on server side is allowed in this case (reuseport has different behavior on different platforms thus also not considered for simplicity reason, well, if anyone has any idea of an alternative solution then help really appreciated). With that structure in mind, the socket process is obviously the bottleneck as it is essentially sequential and therefore should do as little work as possible. By using `{active, true}`, the socket process may get penalty if it can not process messages as fast as messages come into its mailbox, hence degrade performance a lot. 

What's the solution? As far as I know, simply use `{active, N}` instead of `{active, true}` where N is somthing like 100 and have an extra `handle_info/2` clause to reset the socket (`inet:setopts(Socket, [{active, N}]`). This may lead to a first impression that more messages need to be handled by the socket process, however, it actually improves the whole throughput and does not impact latency much. One may want to adjust the value of N based on the network traffic status. From my preliminary test a larger N may help throughput under high load but increase the latency between a request and its response.

UPDATE:

It is observed that combining `{active, N}` and `{read_packets, Integer}` could further improve throughput. According to official documentation, `{read_packets, Integer}` "sets the maximum number of UDP packets to read without intervention from the socket when data is available. When this many packets have been read and delivered to the destination process, new packets are not read until a new notification of available data has arrived." There is [a discussion of this option](http://erlang.org/pipermail/erlang-questions/2017-January/091290.html) in the mailing list that described it clearly. In my test, set `{active, 100}` and `{read_packets, 1000}` gives me a good enough result.