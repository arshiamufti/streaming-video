# Summary: On Crowdsourced Interactive Live Streaming: A Twitch TV-Based Measurement Study

[link to paper](https://arxiv.org/pdf/1502.04666.pdf)

## Twitch as a platform: how is it different from other streaming services?

1) Twitch does not provide the sources of live streaming by themselves. Twitch instead **bridges sources and viewers**. These sources are not professional content creators (they can be gamers, hobbyists, etc) and often have limited computational power and inconsistent network resources. Due to this, broadcasters can start and stop streaming at will or could crash (stop broadcasting) anytime. This makes the live streaming relatively **non traditional** and poses additional challenges to live streaming.

2) Twitch also supports and encourages viewers to interact with broadcasters (contrast this with viewers watching a livestream of an awards show or football game: there is no interactivity). Therefore, in addition to managing the broadcast latency of the video content, the latency of the delivery of these messages must also be managed and synchronized with video delivery.

## In this paper

This paper considers Twitch as fairly representative of modern crowdsources live streaming systems. Using a combination of **crawled data** and **captured traffic** of broadcasters/viewers, the authors aim to investigate

1. the architecture of the Twitch system
2. the behavior and experience of individual content broadcasters and viewers

## Inside Twitch's Architecture

Since Twitch is proprietary, large parts of its system are unknown to users.

### **Crawler**

Using the Twitch API, the authors crawled the access data of livestreams over 2 months (October & November 2014). The crawled data included the following information

- the number of Twitch streams, the number of Twitch views, and the meta-data of live streams every ten minutes.

The resulting dataset dataset included just under 3000 active broadcasters (i.e., sources of live video) who broadcast 100k+ live streams, with around 18 million views. 

### **Findings (Architecture)**

- **Devices:** Broadcast sources are heterogeneous (PCs, laptops, PS4/XBox game consoles). Furthermore, and multiple sources can be involved in a single broadcast event (live stream). PCs/laptops are the most popular devices, followed by game consoles.
- **Streaming protocol:** Twitch uses RTMP (over an HTTP tunnel) streaming servers to compensate for the weaknesses of streaming sources (network or bandwidth fluctuation). The content will then be streamed from the streaming servers to viewers over HTTP Live Streaming (using a CDN).
- **Adaptive online transcoding:** To accommodate heterogeneous viewers, Twitch provides adaptive online transcoding streamers who pay for their premium service. This feature allows the viewers watching from a web browser manually select a proper quality from one of "Source", "High", "Medium", "Low", and "Mobile". Mobile users by default have the "Auto" option (aka adaptive streaming).
- **Interactive messaging:** As described before, Twitch allows stream viewers to interact with broadcasters. A set of interactive messaging servers receive the viewer’s live messages, and then *dispatch* the messages to the corresponding live broadcaster and other viewers. These servers for interaction are only deployed in North America using the IRC (Internet Relay Chat) protocol.

### **Analyzing Behaviour & Experience of Broadcasters & Viewers (Setup)**

- The authors set up 3 source-end PCs (one commentator and two game players) and 5 viewers over the Twitch platform
- Each player has installed a web camera that captures the video in realtime, encodes it locally, and transmits it to the Twitch platform through RTMP.
- To monitor incoming and outgoing traffic, they used network tools such as **Wireshark**, **tcpdump**, and **ISP lookup**.
- To quantify the viewer-side latencies and understand the impact of changes in the network on the QoE, they authors uses the commentator’s laptop as an NTP server and synchronized other devices to improve the accuracy of measurement results.

## View Statistics and Patterns

### Popularity & Duration (of views)

1. For the global view dataset (100k + streams), the plot of the number of views as a function of the rank of the video streams’ has a long tail (on the linear scale). As such, it does not fit a classical Zipf distribution.
2. In general, top of 0.5% broadcasters contribute to more than **70% of the total views**. In some extreme cases, the top-0.4% broadcasters account for more than 90% of total views. From this, the authors concluded that the distribution of the views in Twitch **exhibits extreme skewness** (stronger than that measured in conventional streaming systems like YouTube)
3. Streaming durations are also highly diverse. About 30% of live streams have a duration of 1-2 hours. Another 30% have a duration of over four hours — this is significantly longer than the duration in other user generated video streaming platforms (like YouTube, for which it is rarely over 3 hours). The duration of a Twitch broadcast is also **hard to predict** since it depends on its popularity with the viewers, the level of their engagement, and so on. (Again, note how this differs from traditional live video broadcast, which doesnt change in duration depending on viewer activity). This unpredictability poses an additional challenge on the **allocation of computational power & bandwidth for real-time transcoding and broadcasting**.

### Event & Source Driven Views

As discussed before, since Twitch itself does not create content, the experience of viewing is highly dependent on the broadcasters. From the crawled data, the authors observed that it was not unusual to for live streaming sessions to disconnect and then reconnect quickly. Accordingly, viewership also decreased (and then recovered). Conclusion: Twitch itself is highly reliable with over-provisioned resources but **it cannot guarantee the source video quality as well as traditional platforms.**

## Latencies

The authors measure and analyze 3 types of latencies intricately tied to user QoE: live messaging latency, broadcast latency, and switching latency.

Important note: these viewer-side latencies includes not just the processing time within the Twitch server and the time from the server to the viewer (as in traditional streaming platforms), but also the latency from the source (broadcaster) to the streaming server.

### **Live Messaging Latency**

- this is the time elapsed between sending & receiving a message
- this depends on both network conditions and the used device: the authors observe that
    - two desktop devices witness the almost same latency between the message sending and receiving operations
    - the receiving latency of mobile devices is lower than the sending latency
- overall, the latency is quite low (< 400 ms), which enables responsive interaction between broadcasters and viewers

### **Broadcast Latency**

This is the time taken to deliver streamed video to a viewer — the higher this is, the more limited interaction is and the less "real-time" the system is

**Across Devices**

- Broadcast delay is constantly higher for mobile devices (over PCs) [important note: these measurements were made after ensuring that the downloading bandwidth was significantly higher than the streaming bitrate, to eliminate the effects of a bottleneck in the network]
- Twitch adopts a **device-dependent broadcast latency strategy** to gather the crowdsourced content for processing and to ensure smoothed live streaming
    - For **desktop devices**, there is an unavoidable latency: Twitch streaming servers receive content over RTMP and convert them to HTTP Live Streaming chunks (4 seconds long)
    - For **mobile devices**, Twitch sends three more chunks than desktop devices.
    - Thus, even in an ideal network, mobile devices suffer from an extra 12 seconds of latency

**Across networks**

The authors measured the broadcast latencies of mobile devices across both WiFi and 3G and observed that over WiFi, broadcast latencies are relatively constant as streaming bitrates increase. Over 3G, they increase due to extra network delays.

### Source Switching Latency

A Twitch viewer has the option to stream from different platforms for the same broadcast event.

- Switching latency is typically related to downloading bandwidth (the more a device has, the lower the latency).
- This relationship isn't a strictly proportional one. For example, the devices in a mobile network generally have lower switching latencies than those in a wired network, although the mobile bandwidths are indeed much lower. This is probably evidence of a device based strategy used in Twitch.

## Impact of Broadcaster Behaviour

As described before, broadcasters on Twitch have diverse network conditions and computing power — both in terms of **stability** and **capacity**.

By varying the maximum uploading bandwidth of a broadcaster, the authors observe that:

- with a decrease in bandwidth, there is a corresponding increase in the number of dropped frames
- there is also a corresponding increase in the observed broadcast latency (viewer-side)

Still Twitch attempts to maintain a stable broadcast latency by using the following strategy: if the uploader's capacity recovered (it drops less frames than before), Twitch does not correspondingly decrease the broadcast latency. The broadcast latency is decreased only slightly - perhaps in response to the network variation on the broadcaster's end - in order to provide a smoother viewing experience to the viewer.

Overall, it is evident that Twitch has less control over a smooth live streaming experience due to potential variation both **on the broadcaster's end and the viewer's end.**
