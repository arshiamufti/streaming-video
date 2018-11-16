# Summary: Automated Control of Aggressive Prefetching for HTTP Streaming Video Servers

[link to original paper](https://cs.uwaterloo.ca/~bernard/systor-2014.pdf)


In this paper, the authors explore the size and controlling factors of application level prefetching for HTTP streaming video in an effort to increase the throughput of a web server serving video.

---

# Relationship between disk I/O and server throughput?

Video files on average are too large to fit into memory. Furthermore, services like Netflix and Youtube have such large video collections that it is not currently economically viable for all videos to be stored on SSDs. Due to these 2 factors, the majority of requests serviced by HTTP streaming video servers must be serviced from disk.

There are two techniques for using system memory to improve performance for disk-bound applications:

1. Caching, which attempts to take advantage of the **temporal** locality of data accesses. Caching can certainly provide benefits for streaming application, but its effectiveness is limited since most videos are seldom watched.
2. Prefetching (data that isn't needed yet by the user is stored in memory for future access) which attempts to take advantage of **spatial** locality by reading data into memory before an application needs to access it. Prefetching, when done right, can improve both disk and application performance three relevant ways
    1. minimizing CPU stalls by avoiding file system cache misses
        - not something we directly want to address since (a) web servers are designed to cope with CPU stalls (b) network delays dominate over disk i/o delays
    2. reducing the length of disk seeks by scheduling in batches
    3. improving throughput by reducing the number of seeks made by the disk
        - this is the the dimension the authors try to improve along

Access for video files is largely sequential, and hard drives provide their highest throughput with sequential workloads (for eg., prefetching works better with a sequential access pattern). However, for streaming applications, the OS alone does not provide sufficiently efficient disk access to service this volume of requests. Therefore, the question of application level prefetching arises, **in which the application serializes disk requests and aggressively prefetches data using reads in the order of a few MB.**

Once we begin to consider this idea, two interesting questions arise.

1. what is a good prefetch size for a given workload?
2. what factors affect this choice?

# The need for serialized, aggressive prefetching

To motivate the need for this application level prefetching, the authors run some simple tests:

- videos are stores as files, not chunks (to emulate the way Netflix/Youtube store data)
- they measure **actual** throughput (observed by the client) and **disk** throughput (observed at the disk) for serving workloads for video at different fixed prefetch sizes (sizes vary from 0 mb to 10 mb)
- this is performed for both HD and SD video

**Results**

- There is indeed a significant performance difference between the "vanilla" web server (no application level prefetching) and the modified web server (performs some prefetching).
- For SD video, 4 MB turns out to be the optimal prefetch size (a 2.5 improvement in actual throughput) and for HD video, 8 MB is the optimal prefetch size (3.4 improvement in actual throughput).
- As prefetch size increases beyond a point, throughput decreases because a lot of extra work has to be done to handle requests. This extra work could be due to file cache evictions (cached data is evicted and later requested again) or prefetch evictions (prefetched data is evicted and later prefetched again) or wasted prefetched (prefetched data that is never requested)

**Useful note:** Disk throughput is not always equal to observed throughput at the client. As prefetch sizes increase, so does memory pressure, and this reduced server throughput

However, the optimal prefetch size for an application is highly dependent on factors relating to the workload, so picking a fixed prefetch size isn't always viable. This motivates the need for an **adaptive algorithm** which dynamically selects the best prefetch size with the aim to increase the throughput of the system.

**Useful note:** This algorithm runs at the application level, instead of the OS level.

- **difficulty:** It is easier to modify, test and debug an application than the operating system.
- **accessibility:** It is easier to obtain useful application level information directly from the application (eg, video bitrate) rather than attempt to infer it in the operating system.
- **portability:** Changes made in the application are portable to other operating systems.

# An Adaptive Algorithm

**The Score:** While web server is running, a score S is continuously calculated. This represents the amount of work done and is a product of both the time to read from disk and the file cache miss ratio. A gradient descent algorithm is used to find the prefetch size that minimizes S. It's worth noting that for some prefetch sizes, a small increase in the file cache miss ratio can lead to a significant decrease in transfer time (and therefore, improve overall throughput).

- The algorithm starts with an initial prefetch size and continually measures the score while the server is running.
- At regular intervals, the algorithm compares the score at the current prefetch size (p) to the scores that were previously measured for the next larger (pL) and smaller (pS) prefetch sizes.
- If either pL or pS has a lower score than p, the prefetch size is adjusted towards that size.
- If the score has not yet been measured for pL and the measured score for ps is higher than the score for p, the algorithm switched to the unmeasured prefetch size, pL (and vice versa if pS has not been measured).
- If neither pL and pS are both unmeasured we try pL.

## **Light load**

When the system is under light load, the gradient descent method is not suitable. Increasing the prefetch size will improve the disk transfer time, but it may not cause a corresponding decrease in the cache miss ratio. This results in a large prefetch size that will cause low throughput when the load increases. To handle this, the automated algorithm is off when the load is too light. This is acceptable because under light load, the server is able to properly service all clients.

## **Heavy load**

When the server is under heavy memory pressure or is heavily loaded, the gradient descent method is does not change the prefetch size quickly enough to avoid client timeouts. This is addressed by quickly changing the prefetch size to a new value calculated using the **prefetch eviction time** (the average amount of time that prefetched data is resident in memory before it is evicted). Since the client request rate is known, it is possible to compute how much time (t) will elapse before the prefetched data will be accessed. If t is greater than the prefetch eviction time, then we know that prefetched data will have to be re-read from disk later. So the algorithm **reduces** the prefetch size to **increase** prefetch eviction time to avoid those re-reads.

## **Slow changes**

In other cases, the prefetch size is slowly adapted over time.

## Experiments

Experiments demonstrate that the algorithm is able to converge to a good prefetch size with different workloads (HD versus SD video), when when the initial prefetch size is less optimal.

# Relationship between bitrate and optimal prefetch size

Most experiments conducted by the authors were over fixed bitrates. However, since bitrates do impact the optimal prefetch size, it is useful to study this relationship. The authors find that when the prefetch sizes are proportional to the square root of the video bitrates, the # of disk seeks is minimum — and this metric is directly related to improving disk throughput.

Thus, when picking a bitrate-specific initial prefetch size, the authors specify a size for the highest bitrate video and scale down the prefetch size for lower bitrate videos proportionally.

**Useful note:** This analysis slightly contradicts other work by Gill et al. who demonstrate that the optimal prefetch size is directly proportional to the bitrate (not the square root). However, their definition of "optimal" is different — their goal is to optimize the cache hit rate, not the number of seeks.

# Other Factors

## Workload distribution

The distribution of the workload (value of alpha)directly affects the effectiveness of caching. If the popularity of the hottest files increases (i.e. alpha increases), the file system cache is more useful.  Although prefetching is not directly affected by the popularity distribution, caching and prefetching compete for system memory.

According to experiments, with a larger alpha parameter (i.e. increased popularity of files), the actual throughput is improved for all prefetch sizes (ranging from 0 to 12 mb). But for a given prefetch size, and a range of alpha values, there is little or no change in **disk and eviction throughput** for a given prefetch size. Thus, the increase in actual throughput is due to an increased cache hit rate. Since disk throughput stays the same, that tells us that the change in the workload distribution doesn't impact the amount of memory available for prefetching.

## Mixed Bitrates

The authors use a client workload that is 50% SD and 50% HD. They scale the prefetch sizes of the SD video w.r.t the HD video using various ratios such as 1, 0.1, 0.45, etc. The factor of .45 is of special interest due to the rule derived in a a previous section: the prefetch size should be proportional to the square root of the bitrate. With prefetch sizes fixed in this manner, the authors run their adaptive algorithm for this split-bitrate workload. They observe that:

1. There are only very small differences in server throughput across the different scaling factors (such as 1, 0.1, 0.45 etc), for their system configuration.
2. The throughput obtained with the scaling factor of 0.45 is as good or better than that achieved with the other scaling factors.

The authors also run these experiments while varying hardware related factors (such as the disk and system memory, not discussed in this summary).
