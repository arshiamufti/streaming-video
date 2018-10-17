## Background

As an introduction, the paper introduces two fundamental ideas:

1. The metrics needed to accurately capture and represent the performance of a streaming video application are different than applications with more traditional workloads.
2. Although streaming video applications are increasingly preferring HTTP chunk-based protocols over more specialized streaming protocols, the architecture of the underlying infrastructure (at the CDN and ISP level) does not leverage the streaming nature of these applications.

The these ideas are elaborated upon in this section

**Idea #1**

Strategies to optimize applications with traditional workloads focus typically on latency or throughput. For example, one would want to optimize the latency of web transfers or user sessions and increase the throughput large file transfers. However, optimizing these metrics don't necessarily translate to a pleasant user experience for streaming applications. Reducing latency in streaming video applications is less critical because application data units are large enough to amortize latency effects. Reducing overall completion time also doesn't necessarily reflect the quality of the viewing session because data such as the rate of interruptions due to re-buffering, frequency of rate adaptation, slope of rate adaptation etc aren't captured in that metric.

Thus, there is a need to introduce new metrics to capture the nature of a viewing session. Besides this, there is a need to **redefine the way we analyze these metrics**. Thus, not only do new metrics have to be introduced at the user and network level in order to address this shortcoming, but their quality also needs to be sustained over extended periods of time (in the order of minutes). The second requirement follows from observed user preferences and demands of video applications. For example, users are sensitive to buffering—a 1% increase in buffering can lead to more than a 3 minutes reduction in expected viewing time.

**Idea #2**

Streaming video applications are increasingly deploying HTTP chunk based protocols, instead of the more traditional, specialized streaming protocols (like RTMP). The ease of deploying these applications on already existing HTTP infrastructure (such as HTTP based CDNs) has in fact led to a rapid increase in the creation of these applications and platforms. Furthermore, a diversity of clients can be allowed to use these systems since HTTP across clients is also ubiquitous.

However, there is a mismatch between the requirements of these applications and the provisions of these HTTP-based infrastructures at the ISP and the CDN level. The authors discover via client-side measurements of over 200 mil client viewing sessions video rebuffering rates, start up times, etc are quite high and average bitrates at which video is streamed is also typically pretty low.

- 20% of sessions experience a rebuffering ratio of ≥ 10%
- 14% of users have to wait more than 10 seconds for video to start up
- more than 28% of sessions have an average bitrate less than 500Kbps, and 10% of users fail to see any video at all.

From this, it is evident that the underlying video delivery infrastructure is not sufficient as it is. Recall that the **overarching goal** is—to improve the video viewing experience, given an unreliable delivery network.

The three questions the authors pose are:

1. What parameters can we adapt? [eg, bitrate, parameters of the CDN]
2. When should these parameters be optimized? [at video startup? mid stream?]
3. Who sets these parameters? [the client? the server?]

## Overview of the current infrastructure

As a first step, the authors examine the performance of today’s delivery infrastructure and highlight potential sources of inefficiencies.

**Data collection**

- data is based on one week of client-side measurements from 200+ million viewing sessions (successful and failed) from 50+ million viewers across
91 popular video content providers around the world.
- The content served includes both live and video-on-demand video, but since similar results were observed from both types of traffic, only aggregated data is shown.
- Client side statistics regarding the current network conditions (e.g., estimated bandwidth to the chosen CDN server) and the observed video quality (e.g., re-buffering ratio, chosen bitrate) were collected.

**Metrics**

As explained in the introduction, metrics that have an accurate impact on user engagement and satisfaction must be captured. As such, the authors focus on the following:

- Average bit rate: The average bitrate experienced by a view over its lifetime.
- Re-buffering ratio: Total buffering time divided by buffering plus playing time [excluding paused or stopped time and buffering time before video start]
- Startup time: The buffering time elapsed before a video starts to play.
- Failure rate: The % of sessions that failed to start playing **and** experienced a fatal error during the process.
    - *Note*: these errors are typically related to the CDN (i.e., edge servers are missing data or are overloaded and therefore rejecting new connections)
- Exits before video start: The % of sessions that failed to play the video **without** experiencing a fatal error.
    - *Note*: these "errors" are usually user related — they occur because users are not interested in the con- tent or because they waited too long for the video to load and lost patience.
