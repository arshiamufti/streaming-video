# On the merits of SVC-based HTTP Adaptive Streaming

Summary of "On the merits of SVC-based HTTP Adaptive Streaming" by Famaey, Latre, et al.

full paper [here](https://www.researchgate.net/publication/244483544_On_the_merits_of_SVC-based_HTTP_Adaptive_Streaming) 

## HTTP Adaptive Streaming

- Content is temporally segmented and each segment is offered in different video qualities to the client. A video client can **dynamically** adapt the consumed video quality to match with the capabilities of the network and/or the client’s device.
- Thus, the use of HAS allows a service provider to offer video streaming over heterogeneous networks and to heterogeneous devices.
- Traditionally, H.264/AVC video codec is used to encode the HAS content (for each piece of content such as a movie, a separate AVC file is encoded)
- **benefits:** Using HTTP as a transport protocol allows HAS streams to easily traverse firewalls. Existing HTTP delivery infrastructure can be seamlessly adopted.
- **problem:** this leads to considerable storage redundancy since the same data has to be encoded in multiple ways. This also leads to increased storage and bandwidth requirements and reduced caching efficiency.
- Example of HAS protocols: MS smooth streaming, Apple Live streaming, etc. These have begun to substitute RTP.

---

Recently, there has been a SVC (scalable video codec) extension of AVC.

- Video is encoded in different layers and downloading each layer can improve video quality.
- The lowest quality level (base layer) can be decoded independently. The higher levels can only be decoded in combination with the previous layers.
- **benefits:** SVC reduces content dependency
- **problems:** introduced additional encoding overhead, which increased the bit rate of the content stream

## Existing implementations of HAS

MS Smooth Streaming, Apple's HTTP Live Streaming, and DASH all have the following in common:

- a standard web server, offering the segmented HAS content
- the transmission of the segments over standard HTTP connections
- an intelligent video client who uses some heuristic to determine at which quality to download future segments

**network based modifications**

- of HAS for 3GPP networks: the download and requests of HAS segments are parallelized [Liu et. al]
- an improved scheduling algorithm for HAS to improve the network utilization in CDNs [Pu et al]
- A modification of TCP for the last mile of content delivery (which is usually wireless) to increase avg throughput.

**server side optimizations**

- Traditionally AVC (adaptive video coding) is used for streaming video. The source video must be encoded into multiple independent AVC videos, each in a different quality —> leads to server side storage redundancy.
- The Scalable Video Codec is an **extension** of AVC which alleviates this.
- Previous studies of SVC: Huysegems et al, and Sanchez et al.
- *[Huysegems et al]* Theoretic advantages and disadvantages of using SVC instead of AVC.
    - **Advantages**: a theoretically better play-out during fluctuations (as SVC always downloads the lowest quality first), a reduction in storage and bandwidth requirements.
    - Challenges: the penalty in bitrate for encoding SVC, an increasing vulnerability to high round trip times.
- *[Sanchez et al]* discuss the benefits of using SVC for HAS delivery & propose an algorithm for live HAS delivery
    - **Advantages:** better web caching performance, saved uplink bandwidth and propose a scheduling algorithm for live HAS delivery. Additionally, an initial comparison of SVC and AVC is carried out but focused around the observed live latency.
- These two studies do not take into account an important feature of SVC: SVC-based client heuristics can decide to either download the next segment or increase the quality of previously downloaded segments.

# SVC over AVC

AVC: MS-SS (Microsoft Smooth Streaming)

SVC: MS-SS is adapted for SVC based selection

## AVC: MS-SS

- buffer’s state has 3 thresholds: a lower threshold L, an upper threshold U and a panic threshold P
- The goal of the heuristic is to (1) maintain a steady state of the play-out buffer and (2) download the highest quality possible.
- Every time a new segment is downloaded, the buffer’s state is evaluated to decide on the next segment.

***buffering state***

We start off in buffering mode — the heuristic will follow a more aggressive way of increasing the quality. It downloads the highest possible quality.

If the buffer is is decreasing or slowly increasing, the quality can only be increased or decreased with one level. This is to avoid oscillations in the decision of the heuristic (continuous fluctuations in the video quality can annoy the user).

The buffering state continues until the buffer is almost completely full (lines 7-8). In this case, the heuristic goes into a steady state.

***steady state***

- change in quality at this stage is more conservative — video quality is only increased or decreased with one level at a time
- to determine if the quality needs to be changed, the heuristic analyses (1) the buffer state (2) the download time of the prev segment and (3) the potential occurrence of the playout buffer getting starved.
- If starvation — next segment is downloaded in the lowest quality segment
- if download time was longer than expected — quality for next segment is decreased
- if the buffer state is below the panic threshold — we next segment downloaded in the lowest quality AND the buffer goes back to the buffering state

—

- if the buffer state is higher than P, the heuristic analyses the evolution of the buffer
- if the buffer is slowly changing:
    - if the buffer is below the lower threshold L — quality for next segment is decreased by 1
    - if the buffer is above the upper threshold U — quality is increased by 1 (if the bandwidth measurements indicate that this is possible.
- if the buffer is quickly changing:
    - if the buffer is below L — buffer switches to panic mode (next segment in the lowest quality) and switch to buffering state

## SVC: SVC-based MS-SS

**Naively porting MS-SS**

In the naive port, the MS-SS heuristic for quality selection (above) can be used for SVC sessions as well. Here, instead of decided to increase the bitrate of the segment being downloaded, the decision is translated into the # of layers being downloaded.

***disadvantage*:** This does not does not fully exploit the specific characteristics of SVC-based videos

**Sloping-based SVC heuristic**

An SVC-based client heuristic can allow more **granular** decisions than AVC-based clients.

- In AVC, a decision on the next quality to download is needed every s seconds, s = the duration of the segment. For SVC, the decision can be made more often because decisions can be re-evaluated after every download of a **layer** of the segment (and these layers smaller in size than the full AVC-based segment).
- The AVC-based decision can only download the **next** segment and needs to decide on the correct quality for that segment. In contrast, the SVC-based heuristic can, at any time, decide to download the base layer of a **new** segment **OR** increase the quality of a previously downloaded (not already played) segment by downloading an additional enhancement layer.

[the algorithm below was proposed by Andelin et al.]

***algorithm***

The heuristic recomputes the quality at which to download after the download of every layer.

- It can be configured to give priority to either prefetching (downloading for future segments) or backfilling (downloading for the current segments).
- This configuration is done by defining a slope in the heuristic: the steeper the slope, the more backfilling will be chosen over prefetching. Similarly, the flatter the slope, the more additional base layers of new segments will be downloaded.
- The configuration of the slope parameter of course significantly influences the behavior of the algorithm.
- [Andelin et al showed that] a flatter slope is needed when (1) the rate is low, (2) the rate is highly variable or (3) the rate has a high persistence but a chance of being low. In the next section, we vary the slope parameter in our comparison study of this heuristic with the two earlier described AVC- and SVC-based heuristics.

---

# Tests

Setup: a video server which provides a HAS-based streaming service to a set of N clients. For all experiments except the last, N = 1. The length of the playout buffer was varied for some experiments from 6 to 24 secs to investigate both small and large buffer movements. Each video client has a play-out buffer of P seconds, which is varied from 6 up to 24 secs.

## High RTT

- **RULE!** The throughput of a TCP connection is known to be inversely proportional to the RTT.
- In this test, buffer starvation and avg played video quality were measured under varying RTT. (Buffer starvation here is define as the time that the client has to wait for the segments to play. The client would have to wait when the buffer underflows, which would freeze the last frame and decrease QoE.)

***buffer starvation***

- Increasing RTT — increases the length of buffer starvations.
- **Why?** With an increase in RTT, there is a decrease in TCP throughput. So it is more likely that there will not be enough throughput to stream to the client and the buffer will go empty. The client will be idle longer, waiting for a response for a requested segment
- All 3 heuristics experience behave similarly under an increasing RTT.

***played video quality***

- SVC-based algorithms are much more vulnerable to high RTTs than AVC-based ones.
- **Why?** In SVC, the number of requests that need to be sent to the server is much higher, because each layer needs to be requested separately. This decreases the throughput even **further** because the client is idle for a longer period of time.
    - Note: The idle time between subsequent requests can be reduced or even removed by using **HTTP pipelining**. (in other words, dispatch the next HTTP request before receiving the previous response fully)

## Bandwidth fluctuations

Users often watch video on phones or tablets. In a mobile environments, the network is highly prone to fluctuations in bandwidth. In this test, the three algorithms were tested in a fluctuating bandwidth environment (the authors tested this on a moving tram)!

***small buffer sizes***

- AVC MSS algorithm is the least capable of gracefully adapting to highly variable bandwidth
    - **why?** thee periodic drops in available bandwidth cause it to continuously switch between different qualities
- SVC algorithms perform much better.
    - **why?** due to its ability to keep its buffer filled more easily.
    - The algorithm can first download multiple base layers and proceed to download higher quality layers. But in AVC, the entire segment needs to be downloaded — so when bandwidth is reduced, the buffer depletes more quickly.

***large buffer sizes***

With an increase in buffer size, all algorithms cope much better with bandwidth fluctuations. Eg, for a buffer size of 24 seconds, all algorithms are able to maintain the highest quality.

As depicted in Figure 5c, increasing the buffer size allows all algorithms to better cope with bandwidth fluctuations. When using a buffer of 24 seconds, all algorithms can maintain the highest quality, except when severe throughput problems occur between.

## Network Congestion

Previous experiments were done on a single client. But in reality there are multiple clients competing for shared bandwidth. In this test, the ability of the heuristic's ability to **fairly share bandwidth** among multiple clients.

*How do we evaluate the fairness of shared bandwidth?* By counting the number of clients, at every instant in time, at each of the three quality layers. Ideally, the clients should be split over at most two layers. When bandwidth is very low or very high, all clients should receive the minimum or maximum quality (respectively).

According to test results, all three algorithms show unbalanced behavior [note: discuss the graphs further with Tim).
