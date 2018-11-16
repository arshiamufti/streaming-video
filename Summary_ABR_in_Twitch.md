# DASH in Twitch: Adaptive Bitrate Streaming in Live Game Streaming Platforms

[link to paper](https://dl.acm.org/citation.cfm?id=2676657)

## In This Paper

In this paper, the authors focus on implementing adaptive bitrate (ABR) streaming in a large online live-streaming platform (Twitch).
* They motivate the need for ABR streaming in live streaming.
* Design two strategies for implementing ABR
* Discuss the costs and benefits of ABR in Twitch

## Background on Twitch
The broadcasters are gamers, who control one channel each. When a channel is "online", that means that the broadcaster is broadcasting
live content and a session is live. When it is offline, the broadcaster is either disconnected or choosing not to livestream. While a
session is online, the number of its viewers can fluctuate as viewers choose to start watching a livestream, stop watching it, or get 
disconnected due to some error or resource limit issue.

During a session, a broadcaster captures a video, encodes it, and uploads it to a streaming server. This video stream is the source or
the raw video stream. The streaming server can then transcode the video into one or more multiple video representations which are then
delivered to the end-users (a.k.a. stream viewers).
						
State of Twitch streaming in 2014: ABR streaming is only offered to some premium broadcasters (such as those with a large audience).
This means that a small subset of popular channels is transcoded into multiple representations. Viewers can then take advantage of this 
(they are not limited to downloaded fixed bitrates). For non-premium broadcasters, the raw video stream is forwarded from the streaming
server to the viewers without transcoding. This means that many viewers of non-premium broadcasters are potentially unable to watch the
stream because they are unable to download content fast enough and the content is not available in lower bitrates.

*Note: it appears that as of 2018, ABR is available to all users.*

## Workload at Twitch

*(A) Total bandwidth needed to deliver video to users*

bandwidth is usually over 1.5 Tbps and under 2 Tbps (this doesn't include the transmission of raw video from the broadcasters to Twitch)

*(B) Concurrent Channels*

According to collected data, there were between 4000 and 8000 concurrent channels on a given day. This is a useful proxy for the computing needs of the Twitch data centers. This indicates to us that if we were to, say, implement ABR for all sessions, we would have to pay a high computational cost due to the relatively large # of concurrent sessions.

*(C) Channel popularity*	
					
Broadcaster popularity in Twitch follows a Zipf distribution with a high alpha value (1.3-1.5, over 3 months), signifying a long tail of unpopular channels.

*(D) Quality & Popularity of videos*

The quality and popularity of videos are positively correlated. The authors guess that this is explained by the fact that as a channel gets more and more popular, the channel broadcasters are incentivized to upload content in higher quality (by investing in better equipment, etc).

Naturally, there is also a relationship between quality (resolution) and bitrates. For example, only half the sessions at 720 or 1080p have a bitrate less than 2 Mbps. However, nearly 90% of sessions at 480p have a bitrate under 2 Mbps.

From these observations, we can draw three conclusions:

1. ABR streaming is useful in order to cope with (a) high delivery cost (b) the inaccessibility of channels to viewers with poor download capacities
2. Implementing ABR for all channels is likely not an optimal strategy due to the large amount of computing power needed for 4000-8000 concurrent sessions.
3. We need not treat all channels equally. Since channel popularity in this dataset fits a Zipf distribution with a large alpha value, only a few channels are extremely popular and thus worth serving over ABR streaming.

## Problem Statement
For which channels should the raw video stream be transcoded into multiple representations and then delivered over ABR streaming, given the tradeoff between the cost of transcoding and the benefits of ABR?

## Strategies
This problem is addressed using two different (and separate) strategies:

**Strategy #1: on the fly, "threshold-j" strategy**

The decision of whether an online channel should be transcoded or not can be taken at any time during an session. The selection criteria for transcoding is based on choosing the most popular channels at an instant of time.

1. Only broadcasters with a raw live video at resolutions 480p, 720p, and 1080p are considered as candidates for ABR streaming.
2. Define a value j, which is the threshold on the number of viewers. Every five minutes, all candidate broadcasters with more than j viewers are selected to be transcoded.

**Strategy #2: at-startup, "top-k" strategy**

The decision of whether an online channel should be transcoded or not can be taken only when the session starts and then applies for the entirety of that session. The selection criteria for transcoding is based on choosing the most popular channels historically

- A list of the k most popular channels is maintained by surveying the channels periodically.
- When a new session starts, if the broadcaster is in the list of candidates and if the raw live video has a resolution 480p, 720p, or 1080p, it is selected for transcoding.

## Evaluation

The top-k and threshold-j strategies are evaluated against two simple and extreme strategies:

1. the "none" strategy: transcode zero channels (which is what Twitch does for non-premium broadcasters as of 2014)
2. the "all" strategy: transcode all channels.

The authors also choose k = 50 and j = 100.

The metrics used for evaluation are:

a) average daily bandwidth needed to deliver video channels
b) QoE, approximated by the fraction of degraded viewers which can access a download stream which is lower than their download capacity
c) time taken to transcode (which also approximates the commercial cost of transcoding)

Now, we explore the performance of these metrics in detail:

a) average daily bandwidth needed to deliver video channels

the top-k and threshold-j produce a substantial reduction in the bandwidth needed to deliver the video that is very close to that provided by the "all" strategy. This allows us to conclude two things: that ABR reduces bandwidth requirements, and that it is strategic to focus on transcoding only popular channels

b) QoE

Here, the "all" strategy provides an upper bound. Just like the previous section, the top-k and threshold-j strategies perform nearly as well as the as the "all" strategy: around 80% of degraded viewers are able to find suitable video streams that are lower than their download capacities.

c) time taken to transcode

The transcoding hours of the "all" strategy is one full order of magnitude larger than for the top and threshold strategies.

---

Main takeaway: the top-k and threshold-j provide benefits that are nearly on par with the all strategy. However, the "all" strategy is dramatically more expensive computationally than the first two. By tweaking the values of k and j, it can be possible to strategically pick channels for transcoding such that the benefits (in terms of bandwidth and QoE) are maximized and the computational/commercial cost of this operation is minimized.

Worth discussing: the on the fly strategy and the startup strategies have a very similar performance - why?

Additional resource used in reading: thttps://dl.acm.org/citation.cfm?id=2713177 (by the same authors)
