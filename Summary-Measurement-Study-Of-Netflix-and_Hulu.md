# Summary: Measurement Study of Netflix, Hulu, and a Tale of Three CDNs

[link to original paper]([https://ieeexplore.ieee.org/abstract/document/6915771](https://ieeexplore.ieee.org/abstract/document/6915771)

Netflix and Hulu are large video streaming platforms in the US and Canada, with downstream traffic from their platforms accounting for a large fraction of overall internet traffic during peak hours. In this paper, the authors aim to understand the architecture of these platforms, especially their content delivery strategies. They discover that for Netflix, CDN selection is primarily based on users, whereas for Hulu it is independent of most factors (and so, likely done based on business agreements).

Due to this, Netflix and Hulu do not necessarily provide the optimal quality of experience (QoE) to end users — there is a key trade-off in CDN selection: business constraints vs. end user QoE. The authors also propose new video delivery strategies that can improve the user QoE while still conforming to the business-related and financial constraints of these corporations. However, this summary only focuses on the individual architectures of these systems.

# The Netflix Platform

## High Level Overview

- `[www.netflix.com](http://www.netflix.com)` is hosted by Netflix, but most other Netflix servers are hosted on AWS.
- Netflix uses various AWS services such as S3, EC2, etc. Key functions, such as content ingestion, log recording/analysis, DRM, CDN routing, user sign-in, and mobile device support, are all done in Amazon cloud.
- **CDNs:**  Netflix employs three different CDNs (Akamai, LimeLight, Level 3) to distribute video content to end users. The content is encoded and DRM protected are sourced from the Amazon cloud and copied over to servers in the CDNs.
- **Streaming algorithms:**  Netflix clients use the DASH (Dynamic Streaming over HTTP) protocol for streaming.
- The client uses the Silverlight application to download, decode and play Netflix movies on desktop web browsers.
- When playback is requested, the player fetches the manifest file for the content from the
control server at `agmoviecontrol.netflix.com`, based on which it starts to download content in chunks. Client reports are sent back to the control server periodically.
- Requests to download audio or video chunks are more frequent at the start of a playback session so as to fill up the playback buffer. In "Characterizing the Workload of a Netflix Streaming Video Server" by Brecht et al., this was called the **filling mode**. Once the buffer is sufficiently filled, downloads are more spaced out in time.

## A Quick Segue: DASH

In DASH, each video is encoded in at several different quality levels (bitrates) in separate files. Each file is divided into small units called segments which are a few seconds in length. Clients download the entire video file by requesting specific chunks at a certain bitrate. They also aim to keep their playback buffer of downloaded content at a relatively content size (in other words, they aim to pace the rate of filling up the buffer with the rate at which the application reads out of the buffer to play the video). To accomplish this, the clients dynamically adapt the requested bitrate of content in response to changes in the network or the server. This allows the client to switch between different quality levels at segment boundaries.

## Manifest File

***Contents of the manifest***

- The manifest file for a Netflix title (i.e., a video) provides the client player with metadata to perform the adaptive video streaming.
- The manifest files are **client-specific**, i.e., they are generated according to each client’s playback capability. For example, when a client requests a manifest file, it also specifies the formats and bitrates it has the capability to play and the manifest file is constructed and sent back accordingly.
- The manifest file is delivered to end user via an SSL connection (so it cannot be read over the wire using packet capture tools such as **tcpdump** or **wireshark**). The authors used the Firefox browser and the **Tamper Data** plug-in to extract the manifest files in XML format.
- The file contains:
    - the list of the CDNs
    - video/audio chunk URLs for multiple quality levels
    - parameters such as time-out interval, polling interval and so on
- The manifest files also show that Netflix uses three CDNs to serve the videos. Different **ranks** are assigned to the CDNs to indicate to the preferred CDN to the clients.

***Analyzing the manifest file***

Due to the wealth of information in the manifest file, the authors identify that it is important to analyze it well so as to shed some light on Netflix's content delivery strategies. Specifically, they aimed to understand how factors like geography, client capabilities, popularity of content, length of content, etc affects streaming parameters. To investigate this, they did the following.

- They used 6 different user accounts, 25 movies of varying popularity, age and type, four computers with Mac and Windows systems at four different locations for this
experiment.
- From each computer, they logged on to Netflix using each of the user accounts and played all of the movies for few minutes to collect the manifest files.

***CDN Ranking***

As mentioned previously, in manifest files, CDNs are ranked to indicate to clients which CDN they should prefer to download content from. To analyze this, the authors listed CDN rankings for each combination of user accounts, client computer, movie ID, and time of the day — and repeated this on several different days. The listing revealed that  CDN ranking is only dependent on the user ID.

- For a given user, the CDN ranking remains the same irrespective of other factors such as the types of movies played, computers used, time of the day, and so on.
- For the same movie, computer, location and time, two different users may see different CDN rankings.
- The CDN ranking for each user remains the same for at least several days.
- The CDN ranking is also mostly independent on network conditions and available bandwidth.

***CDN Selection Strategy***

Given that manifest files can specify URLs for multiple CDNs and multiple quality levels, as well as a preferred CDN, the authors aim to investigate Netflix's CDN selection strategy in response to changing bandwidth conditions. To simulate dynamic bandwidth, the authors play a single title from the beginning. Once the playback starts, they gradually throttle the available bandwidth of the preferred CDN in the manifest file. They also use a tool called *dummynet* to throttle the inbound bandwidth to the client.

The observed behavior is as follows:

- The client starts downloading video chunks from the first ranked, or preferred CDN. It starts out by downloading the content at a low bit rate and gradually improving it in a probing fashion.
- As bandwidth for the preferred CDN was lowered while keeping it intact for the other two, instead of switching to a non-throttled CDN, the client sticks with the preferred network and switches to a lower bitrate to adapt. It is only when it can no longer support even
the very low quality level, that it switches to the second ranked CDN.
- In general, the Netflix clients stay with the same CDN as long as possible even if it has to degrade the playback quality level.

# Hulu

## High-level overview

- The desktop client gets HTML for displaying webpages from Hulu’s front-end web server at `www.hulu.com`.
- It then contacts `s.hulu.com` to obtain a **manifest** file. This file describes the server location, available bit-rates and other details.
- The client the manifest file to contact a video content server to begin downloading and playing the video.
- **Streaming protocol:** Depending on the CDN, Hulu uses encrypted or raw RTMP (real time messaging protocol) to serve video for desktop clients and adaptive HTTP for mobile clients.
- **CDNs used:** Hulu uses the same 3 CDNs as Netflix to server video to users. Hulu clients are first assigned a preferred CDN hostname, and then uses DNS to select a server IP address.

## CDN selection strategy

- Via analysis of packet traces, it was found that a client uses only one CDN server throughout the duration of a video. **However,** it usually switches to a different CDN for the next video.
- Hulu clients adapt to changes in network bandwidth by adjusting the bit-rates of and continue to use the same CDN server as long as possible. When the current CDN server
is unable to serve the lowest possible bit-rate, they switch to a different CDN server —
 this strategy very similar to that of Netflix.
- The client also periodically reports it status to  `t.hulu.com`. Data that it reports includes:
    - the status of the client machine at that time (such as total amount of memory the client is using)
    - problems encountered in the recent past
    - the CDN servers for video content and advertisements
    - video bit-rate being downloaded and the current bandwidth at the client machine
    - current video playback position
    - number of buffer under-runs and number of dropped frames
    - reasons for why the requested bitrate changed (if a client adapts bitrates due to changes in the network)

**How do clients pick CDNs?**

To answer this question, the authors downloaded manifest files for 61 different videos of different genres, length and popularity on Hulu from 13 locations across the United States over multiple days. For each video, the manifest file (which contained the preferred CDN) was downloaded 100 times (with a 1 second interval between consecutive downloads). For each set of 100 downloads, the authors calculate the proportion of times each CDN is preferred.

- **Overall,** Level3 is the preferred CDN. It is preferred 47% of times, much more than the other two CDNs.
- **WRT location**, the CDNs have constant popularity — that is, their popularity is not dependent on client location.
- **WRT videos:**  CDN preference is also independent of which video is being served (there is not much variation in their popularity across the played videos).
- **WRT time:** The CDN preferences also stay relatively constant over time.

From this, the authors infer that **initial** CDN preference is most likely based on **pricing and business arrangements** and is not dependent on network conditions, instantaneous bandwidth, or the past performance of the CDNs.

**Note:** There is a striking similarity between the architectures of Netflix and Hulu architecture: both platforms use 3rd party commercial data centers and multiple CDNs to scale their services up and down in response to user demand.
