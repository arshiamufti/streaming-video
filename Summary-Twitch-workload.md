# Summary: On Crowdsourced Interactive Live Streaming: A Twitch TV-Based Measurement Study

# Twitch as a platform: how is it different from other streaming services?

1) Twitch does not provide the sources of live streaming by themselves. Twitch instead **bridges sources and viewers**. These sources are not professional content creators (they can be gamers, hobbyists, etc) and often have limited computational power and inconsistent network resources. Due to this, broadcasters can start and stop streaming at will or could crash (stop broadcasting) anytime. This makes the live streaming relatively **non traditional** and poses additional challenges to live streaming.

2) Twitch also supports and encourages viewers to interact with broadcasters (contrast this with viewers watching a livestream of an awards show or football game: there is no interactivity). Therefore, in addition to managing the broadcast latency of the video content, the latency of the delivery of these messages must also be managed and synchronized with video delivery.

# In this paper

This paper considers Twitch as fairly representative of modern crowdsources live streaming systems. Using a combination of **crawled data** and **captured traffic** of broadcasters/viewers, the authors aim to investigate

1. the architecture of the Twitch system
2. the behavior and experience of individual content broadcasters and viewers

# Inside Twitch's Architecture

Since Twitch is proprietary, large parts of its system are unknown to users.

## **Crawler**

Using the Twitch API, the authors crawled the access data of livestreams over 2 months (October & November 2014). The crawled data included the following information

- the number of Twitch streams, the number of Twitch views, and the meta-data of live streams every ten minutes.

The resulting dataset dataset included just under 3000 active broadcasters (i.e., sources of live video) who broadcast 100k+ live streams, with around 18 million views. 

## **Findings (Architecture)**

- **Devices:** Broadcast sources are heterogeneous (PCs, laptops, PS4/XBox game consoles). Furthermore, and multiple sources can be involved in a single broadcast event (live stream). PCs/laptops are the most popular devices, followed by game consoles.
- **Streaming protocol:** Twitch uses RTMP (over an HTTP tunnel) streaming servers to compensate for the weaknesses of streaming sources (network or bandwidth fluctuation). The content will then be streamed from the streaming servers to viewers over HTTP Live Streaming (using a CDN).
- **Adaptive online transcoding:** To accommodate heterogeneous viewers, Twitch provides adaptive online transcoding streamers who pay for their premium service. This feature allows the viewers watching from a web browser manually select a proper quality from one of "Source", "High", "Medium", "Low", and "Mobile". Mobile users by default have the "Auto" option (aka adaptive streaming).
- **Interactive messaging:** As described before, Twitch allows stream viewers to interact with broadcasters. A set of interactive messaging servers receive the viewer’s live messages, and then *dispatch* the messages to the corresponding live broadcaster and other viewers. These servers for interaction are only deployed in North America using the IRC (Internet Relay Chat) protocol.

## **Analyzing Behaviour & Experience of Broadcasters & Viewers (Setup)**

- The authors set up 3 source-end PCs (one commentator and two game players) and 5 viewers over the Twitch platform
- Each player has installed a web camera that captures the video in realtime, encodes it locally, and transmits it to the Twitch platform through RTMP.
- To monitor incoming and outgoing traffic, they used network tools such as **Wireshark**, **tcpdump**, and **ISP lookup**.
- To quantify the viewer-side latencies and understand the impact of changes in the network on the QoE, they authors uses the commentator’s laptop as an NTP server and synchronized other devices to improve the accuracy of measurement results.

# View Statistics and Patterns

## Popularity & Duration (of views)

1. For the global view dataset (100k + streams), the plot of the number of views as a function of the rank of the video streams’ has a long tail (on the linear scale). As such, it does not fit a classical Zipf distribution.
2. In general, top of 0.5% broadcasters contribute to more than **70% of the total views**. In some extreme cases, the top-0.4% broadcasters account for more than 90% of total views. From this, the authors concluded that the distribution of the views in Twitch **exhibits extreme skewness** (stronger than that measured in conventional streaming systems like YouTube)
3. Streaming durations are also highly diverse. About 30% of live streams have a duration of 1-2 hours. Another 30% have a duration of over four hours — this is significantly longer than the duration in other user generated video streaming platforms (like YouTube, for which it is rarely over 3 hours). The duration of a Twitch broadcast is also **hard to predict** since it depends on its popularity with the viewers, the level of their engagement, and so on. (Again, note how this differs from traditional live video broadcast, which doesnt change in duration depending on viewer activity). This unpredictability poses an additional challenge on the **allocation of computational power & bandwidth for real-time transcoding and broadcasting**.

## Event & Source Driven Views
