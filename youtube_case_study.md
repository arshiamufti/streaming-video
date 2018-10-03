# youtube as a case study

(published in 2007)

[arxiv link](https://arxiv.org/pdf/0707.3670.pdf)

## Current stats about YouTube

* 40 minutes, up 50% year-over-year
* total \# of videos: 5+ billion
* total \# of daily active users: ~2 billion
* the default limit for video length is 15 mins, though users can request an increase. Avg length of videos is around 5 minutes, and very popular videos rarely exceed 10 mins in length.
* for a 480p video encoded in the standard frame rate, youtube recommends a bitrate of **2.5 mbps** [compare this with the most popular bitrate in 2007 being 330 kpbs?]
* Youtube accounts of ~19% of downstream internet traffic during peak hours

## Introduction

- YouTube is growing fast (aided by the the social networking component of the platform) and is suffering from scalability constraints.
- In order to efficiently engineer network traffic and sustainably scale the service up, it is essential to have an understanding of the **features of Youtube** and its **usage patterns**.
- Testing methodology:
    - crawl YouTube for a 3 months (in early 2007)
    - authors have obtained 27 datasets totaling over 2.6 million videos. At the time, this was a significant portion of the entire YouTube video repository.
    - most of these videos were accessible from the YouTube homepage <5 clicks, so they were all active videos and could be considered to be fairly representative of measuring the repository.
- *goal*: to design YouTube for scalability and improved user experience taking into consideration the relationship between videos, the relationship between users, and users and videos, and the features of the videos themselves.

# Background

*Traditional video sharing*

Video sharing before Youtube did not have a social media aspect to it. Content reviews and rating did not exist and videos that were uploaded, transferred, and watched using traditional media servers were like standalone units of content - they were largely independent of each other. Due to this, there was no unifying relationship between videos that could be leveraged in order to optimize or enhance content delivery. However, the features of YouTube introduce this potential - videos can be rated, reviewed, commented upon, and shared. This exposes data and relationships between videos that was not previously available.

*Workload measurement*

There has been a previous research effort into understanding the workloads of traditional media servers, looking at: video popularity, access locality, characteristics of live and stored video streams,etc. Many statistics of traditional media servers are differ from YouTube (especially w.r.t. video length and lifespan).

# Measurements:

Each video contains the following metadata: user who uploaded it, date when it was uploaded, category, length, number of views, number of ratings, number of comments, and a list of “related videos”. The related videos are links to other videos that have a similar title, description, or tags, all of which are chosen by the uploader. (the authors have scraped only the top 20 videos since a web page shows at most 20)

*Crawler*

- If a video **b** is in the related video list of video **a**, then there is a directed edge from **a** to **b**. Our crawler uses a breadth-first search to find videos in the graph. An initial set of 0 depth videos is defined and the crawler processes them in a queue and went down to a depth of 4 or 5.
- note: the crawler is single threaded to avoid being flagged as a network attack
- The initial set of videos the crawler started with was from lists such as “Recently Featured”, “Most Viewed”, “Top Rated” and “Most Discussed”, for “Today”, “This Week”, “This Month” and ”All Time” (a total of ~190 videos)
- The crawler was run every few days with the intitial video set being obtained from these lists. If a previously found video was encountered again, its statistics were updated (this was helpful in tracking popularity).
- Other information that was collected
    - file size and bit rate information (this had to be retrieved separately)
    - information about YouTube users: number videos uploaded, information about their friends, etc (over 1 million users were crawled)

## Characteristics of YouTube Video

**Category:** music the the post popular

**Video length**

- traditional servers contain typically 0.5-2 hour movies
- YouTube is mostly comprised of short clips (in 2007 there was a 10 min limit on YouTube videos)

**File size**

- 98.8% of the crawled videos were less than 30 mb
- avg file size: 8.4 MB [357 TB]

**Bit rates**

- most videos have a bit rate of ~330 kbps (two other common ones are 200 and 285 kbps)

**User access patterns:**

- video accesses on a media server does **not follow a zipf distribution** - this is true for both traditional and YouTube video servers
- when plotting rank vs # of views on a log-log scale, we observe that the curve is **initially** linear, but the tail decreases beyond a rank of around 10^3. [note: the videos used for these measurements were from a single dataset of 100k videos collected in one day, with the set collected as specified in the last section]
- a gamma distribution fits better (so does weibull, refer to pg 5)
- discuss with tim: what does this tail mean? and why does it exist?
- a plot of ratings against the rank als has a similar shape

**Growth trend of the # of views and life span**

- some YouTube videos are very popular (their number of views grows very fast), while others are not
- after a certain period of time, some videos are almost never watched.
- how was this measured: the # of views statistic for relatively recent videos was updated every week for 7 weeks. The resulting dataset only contains videos for which there are 7 full data points, meaning that removed videos or videos for which stats were not available were not counted in the dataset [around 43k videos]
- *modelling the growth trend:* the growth trend depends on a growth factor **p** which is used as the exponent in the power law equation (pg 5). A video's growth trend:
    - is increasing if p > 1
    - is growing mostly constantly if p ~= 1
    - if slowing down in growth if p < 1
- Over 70% of the 43k videos have a growth trend which is < 1 — meaning that as time passes, videos grow in popularity more slowly.
- What is the lifespan of a video: videos will persist on YouTube until the creator takes them down or YouTube takes it down for a violation. We can define the lifespan to end as the time at which the growth of a video slows beyond a certain point [in other words, when the growth, measured by the # of views, increased by a factor of less than **t = 10 %**].
- **most videos have a short active life span**
    - **implications —>** videos are typically watched frequently in a short span of time. After the video’s active life span is complete, fewer and fewer people will access them over time. We can use this behavior when **designing web caching and server storage** by (1) predicting the active lifespan of a video (2) using this predicted value to determine when to cache a video or drop a video from a cache.

## Social Networking in YouTube

The main difference between YouTube and other video sharing platforms: videos are no longer independent from each other. They are inherently tied to their creators and viewers, which also have social relationships of their own.

*Small World Networks*

The **small world concept** is that people are linked to all others by short chains of acquaintances (six degrees of separation). "Social networks that are neither completely random nor completely regular, but possess characteristics of both". The clustering coefficient of a graph is defined as how "cliquish" it is (recall that a clique is a subset of vertices in a graph s.t. the induced subgraph is complete). A **small world graph** is defined as one in which the the clustering coeff. is large (a feature of regular graphs) but the **avg distance between the nodes** (also known as hte characteristic path length) is small (a feature of small graphs).

The clustering coeff. for a **node** in a graph is the the proportion of all the possible edges between neighbors of the node that actually exist in the graph. The clustering coeff. of the **graph** itself is simply the average of the clustering coeffs. of all nodes in the graph.

The characteristic path length of a **node** is defined as: the avg. of the min. # of hops it takes to reach all other nodes in the graph from that node. The CPL of the **graph** is then the avg of this value over all nodes.

*Small work networks in YouTube*

The graph formed by related videos in YouTube has the features of a small world network.

- The clustering coefficient is very large compared to a similar sized random graph
- the CPL of the larger datasets are similar to the short path lengths measured in the random graphs.

This finding follows from the fact that YouTube uses tags, video titles and descriptions to find related videos and create this graph.

## Improving YouTube Scalability

**Proxy Caching/Storage Management**

- caching frequently accessed data at proxies located close to clients is a good way of saving bandwidth, reducing server load, and improving user experience (reduced delays)
- there are 3 features of youtube videos that can help in the design of these proxy caches
    1. the number of youtube videos: this number is very very high compared to other video platforms
    2. size of youtube videos: this is much smaller than a traditional video
    3. view frequencies: these do not fit a zipf distribution
- Considering this, prefix caching would work better than works full-object caching for web/segment caching
    - proxy can cache a 5 second initial clip, (~200KB of the video)
    - given the gamma distribution of view frequency (as determined in a previous section) and assuming that cache space is only devoted to popular videos, we would need about 1 GB of disk space for a 60% hit rate (which is acceptable)
    - [discuss further with tim — how would this impacted UX once the cached content was fully served up?]
- releasing a cache - when is the optimal time to do drop content?
    - according to measurement, most videos are active for a relatively short span of time
    - we can develop a predictor to forecast the length of a lifespan of a video and based on predicted lifespan, the proxy can drop videos which are past their lifespan
    - this can also be used to optimize disk management - we can develop a **hierarchy of storage** (videos past their lifespan can be moved to slower and cheaper storage media)
- further optimizations
    - there is a strong correlation between the number of views of a video and the average of the neighbours’ number of views (here, a neighbour = a related video). Using this relation, we can cache the prefixes of its directly related videos can also be prefetched and cached [arshia: especially useful with autoplaying videos]
    - interesting observation: caching this way is just as performant as caching prefixes of the most popular videos but the communication overhead is significantly lower because we don't have to keep track of the most popular videos list.

**Peer-to-peer sharing for YouTube?**

- p2p technology has been used to support live streaming and on demand streaming [potential future reading idea?]
- advantage of p2p:
    - each peer contributes its bandwidth to serve others, so this scales very well for large users and large amounts of content.
    - data is also more highly available since there is no single point of failure
    - However, youtube still uses the traditional client-server architecture, which hinders scalability. [talk to tim about this — why does youtube not currently use a p2p structure?]
- problems with p2p delivery:
    - youtube videos are quite short so the overlay would suffer from a high churn rate
    - "*Moreover, there are a huge number of videos, so the peer-to-peer overlays will appear very small."* - page 9 [ask tim what this means]
    - removing illegal content or content that violates copyright would take longer
- however, if a group of related videos could be considered one large video, the overlay could be much more stable
