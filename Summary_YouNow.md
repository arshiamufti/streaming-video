# An analysis of the YouNow live streaming platform

[link to paper](https://ieeexplore.ieee.org/document/7365913)

YouNow is a platform similar to Twitch. This paper takes on a similar approach to the analysis of Twitch by Zhang et al. and applies this to the YouNow platform. Data is collected for just over a month for around 86k users

## Architecture

A noticeable difference in the design of YouNow (over Twitch) is video playback. FOr playback, a flash container is created and embedded in the website and playback occurs over the Real Time Messaging Protocol (RTMP). The video streams are encoded using AVC (as is most common), but the adaptive playback doesn't occur over HTTP. The content is then served using the Wowza Streaming Engine through Akamai.

## Findings

**Session Duration**

About 93% of the broadcasting sessions lasted less than 100 minutes. This dramatically different from Twitch usage where 30% of the sessions last more than 4 hours. A possible reason for this is that most broadcasting sessions originate from mobile devices and are conducted over mobile networks, which imposes limits in terms of data contracts and battery time. YouNow is used significantly differently from Twitch in this respect.

**Session Popularity**

However, similar to Twitch, there exists a small group of highly active broadcasters that is willing to broadcast several times daily as well, responsible for a large fraction of views and viewing sessions. The authors observe that the top 10% of broadcasts are responsible for more than 80% of all views.

**Device Heterogeneity**

The authors observe that around 70% of the view requests originate from Apple devices even though notwithstanding the fact that Apple's market share worldwide was (at the time) lower than 20%. However, a large proportion of teenagers own Apple devices, and they comprise most of YouNow's user base.

**Video Quality**

- Parameters that control the QoE like Frames per Second (FPS) and bitrate, change depending on the type of network used by a  viewer relies on to broadcast a video. FPS and bitrates are highest when a broadcaster is connected through Wi-Fi and degrade if 4G, 3G or 2G are used instead.
- Overall, video quality is lower than in other platforms like Twitch.
- There is evidence for the existence of an adaptive broadcast upload, either based on the network fluctuations or bandwidth measurements (similar to Twitch)
