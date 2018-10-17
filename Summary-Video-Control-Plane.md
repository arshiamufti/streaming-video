[link to original paper](https://dl.acm.org/citation.cfm?id=2342431)

# Background

As an introduction, the paper introduces two fundamental ideas:

1. The metrics needed to accurately capture and represent the performance of a streaming video application are different than applications with more traditional workloads.
2. Although streaming video applications are increasingly preferring HTTP chunk-based protocols over more specialized streaming protocols, the architecture of the underlying infrastructure (at the CDN and ISP level) does not leverage the streaming nature of these applications.

**Idea #1**

Strategies to optimize applications with traditional workloads focus typically on latency or throughput. For example, one would want to optimize the latency of web transfers or user sessions and increase the throughput large file transfers. However, optimizing these metrics don't necessarily translate to a pleasant user experience for streaming applications. Reducing latency in streaming video applications is less critical because application data units are large enough to amortize latency effects. Reducing overall completion time also doesn't necessarily reflect the quality of the viewing session because data such as the rate of interruptions due to re-buffering, frequency of rate adaptation, slope of rate adaptation etc aren't captured in that metric.

Thus, there is a need to introduce new metrics to capture the nature of a viewing session. Besides this, there is a need to **redefine the way we analyze these metrics**. Thus, not only do new metrics have to be introduced at the user and network level in order to address this shortcoming, but their quality also needs to be sustained over extended periods of time (in the order of minutes). The second requirement follows from observed user preferences and demands of video applications. For example, users are sensitive to buffering—a 1% increase in buffering can lead to more than a 3 minutes reduction in expected viewing time.

**Idea #2**

Streaming video applications are increasingly deploying HTTP chunk based protocols, instead of the more traditional, specialized streaming protocols (like RTMP). The ease of deploying these applications on already existing HTTP infrastructure (such as HTTP based CDNs) has in fact led to a rapid increase in the creation of these applications and platforms. Furthermore, a diversity of clients can be allowed to use these systems since HTTP across clients is also ubiquitous.

However, there is a mismatch between the requirements of these applications and the provisions of these HTTP-based infrastructures at the ISP and the CDN level. The authors discover via client-side measurements of over 200 mil client viewing sessions video re-buffering rates, start up times, etc are quite high and average bitrates at which video is streamed is also typically pretty low.

- 20% of sessions experience a re-buffering ratio of ≥ 10%
- 14% of users have to wait more than 10 seconds for video to start up
- more than 28% of sessions have an average bitrate less than 500Kbps, and 10% of users fail to see any video at all.

From this, it is evident that the underlying video delivery infrastructure is not sufficient as it is. Recall that the **overarching goal**—to improve the video viewing experience, given an unreliable delivery network.

The three questions the authors pose are:

1. What parameters can we adapt? [eg, bitrate, parameters of the CDN]
2. When should these parameters be optimized? [at video startup? mid stream?]
3. Who sets these parameters? [the client? the server?]

# Current Infrastructure

## Overview of the current infrastructure

As a first step, the authors examine the performance of today’s delivery infrastructure and highlight potential sources of inefficiencies.

**Data collection**

- data is based on one week of client-side measurements from 200+ million viewing sessions (successful and failed) from 50+ million viewers across
91 popular video content providers around the world.
- The content served includes both live and video-on-demand video, but since similar results were observed from both types of traffic, only aggregated data is shown.
- Client side statistics regarding the current network conditions (e.g., estimated bandwidth to the chosen CDN server) and the observed video quality (e.g., re-buffering ratio, chosen bitrate) were collected.
- Sessions that lasted < 1 minute were not included in the analysis since they usually came from users are not interested in the video.

**Metrics**

As explained in the introduction, metrics that have an accurate impact on user engagement and satisfaction must be captured. As such, the authors focus on the following:

- Average bit rate: The average bitrate experienced by a view over its lifetime.
- Re-buffering ratio: Total buffering time divided by buffering plus playing time [excluding paused or stopped time and buffering time before video start]
- Startup time: The buffering time elapsed before a video starts to play.
- Failure rate: The % of sessions that failed to start playing **and** experienced a fatal error during the process.
    - *Note*: these errors are typically related to the CDN (i.e., edge servers are missing data or are overloaded and therefore rejecting new connections)
- Exits before video start: The % of sessions that failed to play the video **without** experiencing a fatal error.
    - *Note*: these "errors" are usually user related — they occur because users are not interested in the con- tent or because they waited too long for the video to load and lost patience.

**Observations**

- 40% of the views experience at least 1% re-buffering ratio. 20% experience at least 10% re-buffering ratio.
- 23% of the views wait more than 5 seconds before video starts. 14% wait more than 10 seconds.
- 28% of the views have average bitrate less than 500Kbps, and 74.1% have average bitrate less than 1Mbps.

These metrics can have a huge impact on user experience and engagement. Even a 1% increase in the re-buffering ratio can reduce total play time by several minutes. It has also been observed that playing videos at a consistently high bitrate is strongly correlated with longer viewing sessions.

## Explaining the current infrastructure

In this section, the authors examine three potential issues that could result in the poor video quality observed.

**1) Client-side variability**

The measured client-side inter- and inter-session bandwidth is quite variable. That is, picking the same starting bitrate for every session probably wont work. Keeping the bitrate constant over entire viewing session won't provide good user engagement either. Thus, there is a need for intelligent bitrate selection and switching. Specifically:

- we need to choose a bitrate at the start of each session to account for inter-session variability
- we need to dynamically adapt the bitrate midstream to account for intra-session variability

**2) CDN variability across space and time**

The performance of CDN infrastructure for delivering video can vary significantly both across space (across ISPs or geographical regions) and time. This could be caused by a variety of reasons: changes in load or misconfiguration (content missing from edge servers) or other temporal network conditions.

According to the authors' observations:
- The performance of different CDNs can vary within a given city
- For each metric, no single CDN is optimal across all cities.
- CDNs may differ in their performance across metrics. For example, wrt video startup time, one CDN may perform but wrt failure rate, it may perform the worst.
- No CDN has the best performance all the time. Every CDN experiences some performance issues over time
- The re-buffering ratio and failure rate of a CDN may experience high fluctuations over time.

From these observations, we can infer the need for providers to have multiple CDNs to optimize delivery across different geographical regions and over time. It also suggests that dynamically choosing a CDN can potentially improve the overall video quality.

**3) AS under stress**

When ISPs and ASes are overloaded, they can experience quality issues under heavy load. The authors observe that an increase in the load of the AS (i.e., # of sessions increase) is also accompanied by an increase in the re-buffering ratio.

From this, we can see that it ideal for the video delivery infrastructure to be aware of such hotspots. An increase in load can lead to the entire AS getting congested without some mechanism in place to handle the congestion. Note that if the entire AS is overloaded, the load increases on all CDNs, so switching to a different CDN would not help. Therefore, some optimization has to be performed by the content provider as well, such as reducing the bitrate for all views during overload.

# Designing a Control Plane

## Design Parameters

**What parameters can we control?**

- choice of bitrate
- choice of CDN/server to serve the content.

**When can we choose these parameters?**

- select them at startup time when the video player is launched
    - *Note:* this is not a robust approach as both changes in **client bandwidth** and **network conditions (such as the condition of the CDN)** can impact the ability of the client to download and buffer user experience.
- dynamically adapt these midstream in response to changing network conditions
    - *Note:* this is the current de-facto approach performed **client side**.

**Who decides the values for these parameters?**

- purely client-side mechanisms
- server-driven configuration
- alternative "control plane" that either maintains or is aware of global state and selects these parameters

## The Control Plane

**Main Idea:** use a centralized control plane to optimize content delivery.
The three three key components in the video control plane:

1. a measurement component responsible for actively monitoring the video quality of clients
2. an oracle that uses historical and current data to predict the potential performance a user will receive for a particular combination of CDN and bitrate at the current time
3. the global optimization engine that uses the first two components to assign the CDN and bitrate for each user
