# Summary: Characterizing the Workload of a Netflix Streaming Video Server

# Part 1: Introduction and Background

## The Work

In this paper, the authors aim to investigate and describe the workload of a single production Netflix server called a *Catalog server* which contains a portion of the Netflix catalog. They do this by analyzing the HTTP requests from anonymized log files observed at the server from Netflix client devices.

## The Motivation

Netflix is an extremely popular online streaming platform and comprises a significant of downstream internet traffic in the US during peak hours. Along with Netflix, there are other platforms that provide a similar service (Hulu, Amazon Prime, etc) and are likely designed similarly. Together, the video streamed from these services combined accounts for nearly 50% of Internet traffic. As consumer demand for these services grows, and these platforms aim to scale up in order to meet this demand. To do so in an efficient and sustainable manner, it is important to understand the nature of the production workloads these servers as a first step.

## Background on Netflix

**The CDN**

- The Netflix website itself is hosted on AWS servers and served to users from geographically strategic regions.
- However, the audio and video content itself is not hosted on AWS. It is hosted on clusters of high capacity web servers that are placed in internet exchange (IX) sites around the world. Some ISPs also contain servers supplied by Netflix inside their data center in order to reduce inter-ISP traffic. Together, these two types of deployments comprise the Netflix CDN as it is today.
- In a traditional pull-based CDN, when a client requests content, it "pulls" it down from the the nearest edge server in the CDN. If the content isn't available, the CDN does the work to locate the data from an origin server. The cached data in the CDN typically stays around for a while. This mechanism makes the first request for the data very slow (since it hasn't been cached in the CDN yet).
- The Netflix CDN behaves differently. During off-peak hours when traffic is low, the Netflix control plane is able to make predictions on which titles are most likely to be accessed in specific areas. Based on these predictions, it directs the content servers to add and remove content accordingly.
- When a user requests playback of title, the client device acquires a file called a "manifest" which contains important information such as the addresses of the content servers which should be used to play the content and the bitrate-specific URLs of the content.

**The Streaming Mechanism**

- Content requested by clients is hosted on content servers and made available in several different audio and video bitrates.
- For the most part, clients strive to play content in the highest available quality (both audio & video) supported by the network and the available content servers. They do this by implementing some variant of a DASH (dynamic adaptive streaming over HTTP) algorithm, which attempts to playback content at a high quality, but at the same time aims to keep its playback buffer size at a constant size by adjusting the requested bitrate of the downloaded content in reaction to changes in network or server conditions.
- Clients select a primary server for playback and for the most part, continue to use that unless the server is not performing well (it has low throughput, times out frequently, etc)

## How data was collected

- Data from a single *Catalog* server (which contains less popular content) was collected over a 24 hour period in March 2014
- The server is capable of handling substantially more than the peak throughput of 5.5 Gbps (which is slightly less than double the avg throughput of 2.9 Gbps), so it wasn't overloaded at any point in the data collection period.
- Data from the last hour of the log period is discarded. This is because the end of a log file is always missing some requests that were in flight but not logged yet (they are only logged after they are completely handled) and discarding the tail end of the data simplifies data processing.

# Part 2: Understanding the characteristics of client requests

i. *segments*

Netflix content is divided into 2 second intervals called segments. Segments can be thought of as atomic units — a full encoded segment needs to be downloaded for decoding and playback. So clients would switch bitrates at segment boundaries.

Since segments don't have a fixed size, the beginning of each content file contains a table of offsets for all segments. When a playback session starts, the client downloads segments offsets for multiple files that encode the same title in different bitrates. It then selects a starting bitrate to play content, and then switches bit rate in response to server or network conditions.

ii. *pacing*

Clients perform an action called *pacing* as well. Pacing is the action of limiting the number of segments downloaded ahead of the playback point in order to avoid waste when a user event (pausing, stopping, scrubbing to a different point) occurs. Pacing typically occurs only when a client has achieved a steady state (i.e., enough content has been downloaded such that the client can afford to slow down).

iii. *chunk vs open ended requests*

Netflix clients issue HTTP GET requests in two different ways.

- Chunk requests are those in which the data is requested by providing the offsets of the first and last bytes
- Open ended requests are those in which clients only specify the offset of the first byte. The server handles this by sending data until the client terminates the TCP connection or the file ends.

Clients do not necessarily use only one of these request formats; often different request formats are used for audio and video content.

As mentioned before, clients use a DASH algorithm to download content. However, different clients have different implementations and requirements, so the methods they employ for downloading content can substantially differ. For example, they could differ in when they decide to begin pacing, or their threshold for switching to a lower or higher bitrate, and so on. The paper demonstrates this by examining some two different sessions.

## Key Results

The following are some key results obtained on performing an analysis of all the requests made in the collected data. Note that content can be downloaded as only audio, only video, or a combination of the two.

- **open ended/chunk splits**
    - Of all the data downloaded, pure audio content accounts for only 5% of the total.
    - Although only 1.3% of all the requests (over 65 million) requests are open-ended, those requests account for 1/3 of the requested data.
    - Most of the remaining 2/3 of the requested data (~58%) is downloaded via chunk requests for pure video.
- **request lengths**
    - for most audio and combined content, the requested amount of content amounts to more than 2 seconds of playback — this suggests that clients are requesting more than one segment at a time (recall that one segment is 2 seconds in length)
    - for most video content, the requested amount is, on average, between 0.4 and 1.9 seconds — this suggests that some clients divide a segment into multiple parts and download the parts in parallel
    - note: only chunk requests were examined for analyzing request lengths because of the high variability in the number of bytes downloaded
- **parallel downloading**
    - more than half the files were downloaded in parallel during a session
    - note: this is a useful metric to measure, because downloading in parallel can have a huge impact on the size of the requests (as an example, consider how a big a request for a 2 second video segment would be, versus the individual size of 4 requests for a .5 second fragment of that segment)

# Part 3: Understanding the spatial locality of server workload

The authors identify that it is important to estimate the degree of spatial locality of requests for content downloads. The nature of requests from clients is **not** strictly sequential, as one might visualize. Requests for segments can be issued out of order, especially when we take rate adaptation of the DASH algorithm into account. Having a measure of spatial locality would enable the designers of the system to design prefetching strategies for content. That is, if the requests do not have a high degree of spatial locality, then aggressive prefetching is unlikely to be useful and would in fact be expensive and wasteful. On the other hand, if the degree of locality is high, then prefetching strategies could provide a useful performance enhancement.

To study and measure spatial locality, the authors introduced an abstraction called **chains**.

- A chain represents a contiguous sequence of requests to the **same** file from the **same** client. Note that there is no constraint on the order of requests: a chain can include requests that were received out of order or on parallel TCP connections used by the same client.
- Requests that are **adjacent** to each other (that is, the end offset of one request is adjacent to the start offset of the other) are received within 40 secs of each other. The authors chose this limit by determining that the longest common time gap due to pacing is 32 seconds and therefore concluding that a gap of 40 seconds would include pacing gaps as well as gaps caused by client inactivity.

## Key Results

- Of the 65+ million requests, about 2.3 million chains were detected
- For video and combined content, the length of the chains (both in terms of number of bytes and nominal title time) were the same, regardless of whether the request was a chunk or open-ended request. However, for pure audio content, chains where significantly longer for open-ended requests.
- Most chains are relatively short — over 60% of the chains are 1 MB or lesser, only about 15% of chains are longer than 10 MB. However, most of the content is downloaded in long chains — over 90% of the total data is downloaded using the 15% of chains longer than 10 MB and only 2% of the data is downloaded using the common, short chains.

This allows us to conclude that **in spite of the sequentiality of data potentially being disrupted (due to bitrate adaptation, parallel TCP connections, and accesses to different bitrate files), there is high spatial locality in the requested data**.

*Zero-offset Chains*

- Around a quarter of the chains start at an offset of 0.
- Most chains of **open ended requests** are exactly 768 KB in size, and very few (0.4%) chains are shorter than that. (Note: the exact size of 768 KB is likely due to the size of the socket buffer in the server)
- Most chains of **chunk requests** are shorter than 768 KB (most of them are shorter than 128 KB)
- This allows us to conclude that most zero offset chains are much shorter than chains with non-zero starting offsets.

The authors also calculate that that is fairly likely (a 45% probability) that a chain that grows to a length of 1-60 MB.

These conclusions are later used in developing pre-fetching strategies based on server workloads.

# Part 4: Phases

The authors try to detect client access patterns in playback sessions and to understand the impact of these patterns on the sequentiality of the requests.

## The Three Phases

They first define three types of phases, in which clients issue requests with some distinguishing characteristics:

- Transient
    - The client issues requests for a multiple bit rate files in a short period of time.
    - This typically occurs at
        - the start of a session
        - when there is a change in network or server bandwidth, or
        - after the user changes to a different playback position
    - This pattern of requests for different files demonstrates the operation of the rate adaptation algorithm used by that client
- Stable
    - The client requests content sequentially from the same file (the bit rate being used is stable).
    - At some points in time (such as when a playback session just begins), the client downloads content as quickly as possible in order to fill up its playback buffer. This mode is called the **filling mode**. Once the playback buffer is filled up past a threshold, the download of further content is spaced out or **paced** (this mode is called the pacing mode).
    - Clients typically enter this mode after the transient phase.
- Inactive
    - The client stops downloading content from all files.
    - After this phase, when the client starts requesting content again, it typically enters the transient phase.

## Phases at the start of sessions

The authors analyze chunk requests for non-audio content issued at the start of playback sessions. They assume that clients start off in the transient phase, followed by the stable phase. Four measurements are made: number of concurrent connections, concurrent files (number of unique files requested), arrival interval, and request duration, measured for each second of elapsed time and averaged over the number of sessions that were running for that second.

- The number of concurrent files peaks at after the first 3 seconds of playback.
- The number of concurrent connections follows a similar pattern.

This indicates that in the first few seconds of playback, clients open many parallel TCP connections to download content from files at different bitrates.

- The request duration stays relatively stable over elapsed time (from which we can infer that the request size is largely independent of phase).
- The arrival interval (which is the amount of time between requests, regardless of whether the requests are made over parallel TCP connections or not) increases between 10 and 200 seconds of elapsed time. This can be explained by the different modes of the client. Initially (between 0 & 10 seconds), the client is in the filling mode and issues requests for content relatively faster. Once the playback buffer is filled, it enters pacing mode and spaces out its requests in time, so the average arrival interval decrease and then remains relatively constant.

## Distinguishing Phases

The authors define short chains as those with a duration of ≤ 40 seconds, and long chains have a duration longer than 40 seconds. They then detect clusters of short chains that have a < 10 second gap between the end of one chain and the start of another and define the discrete clusters as representing transient phases. The values of 10 and 40 were experimentally determined values.

- The long chains represent stable phases and are ~8.5 mins in length.
- Nearly every session starts with a transient phase.

## Impact on Sequentiality

- Across all sessions, 5.2% of the time is spend in the transient phases account for 5.2% of the time and around 79% of the time in the stable phases (there is a small amount of overlap between the transient & stable phases for some clients).
- 7.6% of the total number of bytes are downloaded in transient phase chains and 92.4% in stable phase chains. (Note: the proportion of bytes downloaded in the transient phase is higher than the proportion of time because clients are in the filling mode during the transient phase)
- These findings explain the distribution of chain lengths. In the transient phase, there are many short chains since data is downloaded from multiple files. But since not much content is read in total, the chains are short and large in number. The stable phase is the phase in which is where clients spend most of their time, which explains why the chains are long and few in number.

# Part 5: Designing pre-fetching strategies

## First-Grow Algorithm

There are two viable algorithms for picking the amount of data to be pre-fetched. The first is a simple algorithm that does not require utilize any of the data gathered above about server workload. It simply prefetches a fixed amount of data. The second algorithm does utilize workload information in two ways and is called the **first-grow algorithm**.

1. It uses one of 3 specifically-determined prefetch sizes for the first prefetch in a chain. This size depends on on the type of chain and its starting offset in the file.
2. It grows the size of subsequent prefetches by a certain multiplier. Its value is based on a power-law relationship derived from the chain lengths until the prefetch size reaches a maximum.

**Prefetch Size**

To determine an optimal prefetch size, chains are categorized into 3 categories, each with significantly different chain lengths: chains of open-ended requests that start at 0, chains of chunk requests that start at 0, and the remaining chains. We have to pick an **efficient** prefetch size, such that

- enough data is prefetched that the clients can easily avail it when they request it with the costs
- but not prefetch too much (if the prefetch size is longer than the chain length, some prefetched data will never be requested)
- and not prefetch too little (otherwise we will have to repeatedly perform seeks)

Keeping these constraints in mind, prefetch sizes of 1025, 128, and 2867 KB were calculated for the 3 categories respectively.

**Multiplier**

This algorithm also uses a multiplier to increase the prefetch size to provide a 45% survival rate. This value of 45% was determined empirically since it provided the lowest disk utilization for the workload.

We perform a separate cost-benefit analysis for each category and determine the best first prefetch sizes. For this analysis, the benefit is the amount of data that is requested by clients at the start of each chain, up to the prefetch size. The cost has two components: prefetching data that will not be requested because the chain is shorter than the prefetch size, and performing seeks for the chains that are longer than the prefetch size. The prefetch sizes that provide the lowest cost-benefit ratios are listed in Table V and they vary widely, reflecting the different chain length distributions.

## Evaluation of first-grow

To evaluate this algorithm, the usage of two important resources was tracked: the hard drives that store content, and the system memory that holds prefetched content.

[Question for Tim: which category of chains are represented in the graph?]

Using the fixed algorithm, a prefetch size of 7 MB is optimal, with a maximum hard drive utilization of 57% and a maximum memory usage of 47 GB. By setting a 7 MB as the **maximum prefetch size** in first-grow, both maximum hard drive utilization and maximum memory usage is lower. This is because 7MB is only maximum size of prefetch, so the average size of the prefetched data is smaller. Two other alternative ways of evaluating first-grow against fixed are:

- using the same 47 GB of memory usage as the fixed algorithm, the first-grow algorithm uses 8% less hard drive utilization
- at 57% disk utilization, the first-grow algorithm uses 30% less system memory.

These experiments and simulations demonstrate that leveraging information about server workloads in prefetching strategy can be significantly more performant than using a simple, fixed prefetched size.