# Watching Video Over The Web (Pt 1)

- Cable and IPTV services run over **managed networks** for distribution because these services (1) use **multi-cast transport** and (2) require certain quality-of-service (QoS) features.
- In contrast, conventional streaming technologies such as Microsoft Windows Media, Apple QuickTime, and Adobe Flash, as well as the emerging adaptive streaming technologies such as Microsoft’s Smooth Streaming, Apple’s HTTP Live Streaming, and Adobe’s HTTP Dynamic Streaming, run over mostly **unmanaged networks**.
    - These streaming technologies send the content to the viewer over a **unicast connection** (from a server or content delivery network [CDN]) through either a proprietary streaming protocol running on top of an existing transport protocol, mostly TCP and occasionally UDP, or the standard HTTP protocol over TCP.

## Progressive Download

- Progressive download: uses HTTP/TCP
- has been quite popular for online content viewing due to its simplicity
- the playout can start as soon as enough necessary data is retrieved and buffered (used by youtube)
- **cons**
    - progressive download doesn’t offer the flexibility and rich features of streaming. Before the download starts, the viewer must choose the most appropriate version if the content is available in more than one resolution
    - if there isn’t enough bandwidth for the selected version, the viewer might experience frequent freezes and rebuffering.
    - these limitations will inhibit the growth of large volume movie distribution on the internet

## Adaptive Streaming

- aims to address the cons above while preserving the simplicity of progressive download
- hybrid: progressive download + streaming
- it is a pull based mechanism (just like progressive download) — the client sends HTTP request messaged to get particular segments of the content form a sever and then renders the media while the content is being transferred
- but these segments are **short** — this enables the client to download only what is necessary and use "trick" modes efficiently, which gives the impression that the client is streaming.
    - short duration segments are also available at multiple bitrates (each corresponding to different resolutions and quality of content), meaning that the client can **switch between bitrates** with each request (this is not the case with progressive downloads)
    - the client strives to always retrieve the next best segment on the basis of a variety of available factors
        - (network related) available **bandwidth**, state of the **tcp connections**
        - (device related) display resolution, CPU available
        - (streaming conditions) playback buffer size
- goal: to provide the best quality of experience by
    - playing the highest achievable quality
    - starting up faster
    - enabling quicker seeking
- since it uses HTTP, it is highly portable because HTTP connectivity is very ubiquitous.

# Push/Pull Based Protocols

When content needs to be transmitted across a network, two fundamental factors that affect the ways in which we can accomplish this are

1. the type of content being transferred
2. the underlying network conditions

Examples:

- simple file transfer over a lossy network: the emphasis is on reliable delivery, so a protocol that retransmits lost packets would be useful here
- audio/media delivery with **real time** viewing requirements: the emphasis is on low latency and efficient transmission (so occasional losses might be tolerated)

A media streaming protocol is defined by: **(1) the structure of the packets (2) algorithms used to transmit the media on a network**

## Push-Based Media Streaming Protocols

- once a server & client establish a connection, the server streams packets to the client until the client stops or interrupts the session
- the server maintains a session state with the client and listens for client commands requesting state change
- example of this: real time streaming protocol (RTSP) [RFC 2336]
- push based streaming protocols typically use RTP (real-time transport protocol) [RFC 3550] for data transmission. RTP usually runs on UDP.
    - Recall that UDP does not have any inherent rate control mechanisms. This allows the server to push packers to the client at a bit rate that depends on application-level constraints **rather than the underlying transport protocol**.

**bitrates**

- the server transmits content at a bit rate that matches the client's media consumption rate. Under normal circumstances, this ensures that the client's buffer levels remain **stable** over time [because rate of server→client transmission ~= rate of client consumption]
- the server also optimizes the use of the network — the client cannot consume at a rate above the encoding bitrate, so transmitting over that rate would unnecessarily overload the network
- if packet loss or transmission delays occur (in other words, the receive bandwidth decreases), the client's buffer might get drained, resulting in a buffer underflow which interrupts playback.

*bitrate adaptation & bandwidth monitoring*

- to prevent buffer underflow, the server can **dynamically** switch to a stream at a lower bitrate → reduces the media consumption rate from the buffer and counteracts the effects of packet loss/transmission delays
- a sudden drop in encoding bitrate can result in a noticeable visual quality degradation, the reduction of bitrate should take place in intermediate steps **until the buffer consumption rate matches the receive bandwidth**
- when network conditions improve, the server switches to a higher bitrate stream (again, in multiple intermediate steps to prevent network overload)
- **where?** bandwidth monitoring is usually performed on the client side, which computes metrics such as RTT, packet loss, network jitter. Using this info, it can
    - determine when to switch to a lower bitrate stream
    - or communicate this information along with its buffer levels to the server and let the server decide [this information is usually transmitted via the RTP control protocol, RTCP]

## Pull-Based Media Streaming Protocols

- the **client** is the active entity that requests content from the media server
- when the server is **not** overloaded (it's idle, it's blocked on the client), the server response depends on the client's requests. So the bitrate at which the client receives content depends on the client and available network bandwidth
- HTTP is commonly used for pull based media delivery
- common example of pull based protocols: progressive download (discussed above)

**progressive download**

- the media client issues an http request to the server and starts pulling content from it as fast as possible
- once the buffer is filled up to the required level, streaming begins (while continuing to pull data in the the background)
- as long as the download rate is > playback rate, the client buffer is at a sufficient level to enable uninterrupted playback 👍
- if network conditions degrade, the download rate < playback rate ⇒ buffer underflow
- similar to push based protocols, pull based ones use bitrate adaptation to prevent buffer underflow

**bitrate adaptation**

- the media content is divided into short duration segments called "fragments", each of which is encoded at various bitrates and can be decoded independently. *playing the fragments, in sequence, back to back will allow the client to seamlessly construct the original media stream.*
- during download, the client will **dynamically** pick the fragment with a bitrate that is ≤ the available bandwidth and requests that fragment from the server

(further interesting reading, not noted here: fragment construction)

**examples of pull based adaptive streaming protocols, based on multi-bitrate fragments:** microsoft smooth streaming, apple HTTP live streaming

## Other ways of bitrate adaptation

**so far:** adaptive streaming that is based on the principle of switching between alternate encodings of content — either content as a WHOLE or between individual fragments of it encoded in multiple bitrates.

**Scalable Video Coding (SVC):** lets clients choose media streams appropriate for underlying network conditions and their decoding capabilities 

- In SVC, a video bitstream is made up of a hierarchical structure of layers.
- The base layer provides the **lowest level of quality** in terms of frame rate, resolution, and signal-to-noise ratio (SNR).
- Each layer on top of that is an "enhancement layer" - it provides an improvement for one or more of these parameters. Enhancement layers can be **independently stored** or sent over the network.
- **key idea:** we can modify the overall bitrate by selectively adding/removing enhancement layers to/from a stream.
- the "knobs" in SVC are: temporal, spatial, and SNR scalability
    - temporal: add/drop complete pictures from a stream
    - spatial: encodes video signal at multiple resolutions. The client can use lower resolution frames to predict and reconstruct higher resolution frames (using the enhancement layers)
    - SNR scalability: encodes the video signal at a single resolution but at different quality levels by **modifying the encoding precision** (each enhancement layer increases the precision of the lower layers)

**Analysis of SVC**

**(1) Efficiency of data layout/storage:** It has the ability to distribute information among layers with **minimal redundancy**. Traditionally, a stream that is encoded at different quality levels has a lot of redundancy between the encodings. But the layers in SVC have v little common data.

**(2) Graceful degradation of stream quality** when the network conditions hinder the delivery of enhancement layers. This can be done with out client/server intervention. In multi-bitrate encoding, we have to switch from one encoding to another via a client/server decision.

**(3) Complexity:** SVC streams are more complex to generate and impose codec restrictions (?) compared to multi-bitrate streams ⇒ adoption rate for SVC has been slow.