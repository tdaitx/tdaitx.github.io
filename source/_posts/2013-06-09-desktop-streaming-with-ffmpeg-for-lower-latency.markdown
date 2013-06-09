---
layout: post
title: "Desktop streaming with FFmpeg for lower latency"
date: 2013-06-09 15:22
comments: true
categories: 
- ffmpeg
- VLC
- Desktop streaming
author: Tiago Stürmer Daitx
---

Compared to [VLC][desktop streaming with vlc], desktop streaming through [FFmpeg][] allowed me to reduce latency to a minimum, usually less than a second. To achieve such latencies I also had to rely on [FFplay][] instead of [VLC][] as a client.

## tl;dr ##
For smallest possible latency:
{% codeblock  %}
$ ffmpeg -f x11grab -s 1600x900 -r 50 -vcodec mpeg2video -b:v 8000 -f mpegts udp://192.168.0.10:1234
{% endcodeblock %}

Using rtp gives second best latency
{% codeblock  %}
$ ffmpeg -f x11grab -s 1600x900 -r 50 -vcodec mpeg2video -b:v 8000 -f rtp rtp://192.168.0.10:1234
{% endcodeblock %}

Much higher quality with a bit higher latency:
{% codeblock  %}
$ ffmpeg -f x11grab -s 1600x900 -r 50 -vcodec libx264 -preset ultrafast -tune zerolatency -crf 18 -f mpegts udp://localhost:1234
{% endcodeblock %}

For lowest CPU usage and even higher latency (screen rate reduced to 5 fps):
{% codeblock  %}
$ ffmpeg -f x11grab -s 1600x900 -r 5 -vcodec libx264 -preset ultrafast -tune zerolatency -crf 18 -f mpegts udp://localhost:1234
{% endcodeblock %}

For Windows use `dshow` instead of `x11grab` (that's from [examples][ I saw, I didn't actually test it).

Remember to replace 1600x900 with your own screen resolution - on Linux `xrandr` is a handy way to figure that out.

On the client side run `ffplay udp://192.168.0.10:1234`.

## Motivation ##
As in the [previous article][desktop streaming with vlc] I'm looking for a way to share my desktop (read-only) with a broader audience. I was particularly interested in seeing how fast I could push it, given the reports I have read about very low latency streaming.

Keep reading for the full report.
<!-- more -->

## Machine specs ##
Same as [last time][desktop streaming with vlc], everything was tested on a Sony Vaio VPC-SA35GB, Core i5 processor, running Fedora 18 x64 on a single screen with resolution 1600x900.

## Latency ##
As [previously][desktop streaming with vlc], latency is a factor of client and server settings. [FFmpeg][] was able to provide much better latency in both sides than [VLC][]. Audio is still an issue and makes latency way higher and depending on the combination it's completely out of sync with video. For this reason everything was tested with no audio stream whatsoever.

On x264 latency is highly dependend on the encoder settings. **Reducing latency is possible, but reduces quality**. This means that we have to use higher bandwidth to keep a similar quality in low latency scenarios. 

**A highly recommended way to lower bandwidth and CPU usage is by using lower screen resolutions.**

## Server side codecs ##
I only tested mpeg2 and x264 codecs.

### mpeg2video ###
mpeg2 stream through UDP or RTP provided the most responsive stream I could achieve. Quality is way lower than x264 for the same bit rate and CPU usage was about the same, but responsiveness was really good. Not close to real time, more like VNC over the internet (100+ milliseconds), but way better than anything I had achieved so far.

I really need a proper way to compare how much delay I'm getting in such low-latency settings, if you know how, please share it on the comments and I will redo my tests.

### x264 ###
One way to improve latency in x264 is using higher fps for encoding. This increases the ammount of frames available for the encoding process and allows the data to be sent out much earlier. One easy way to see the difference is changing the fps from 50 to 5.

For low latency, I found an excellent discussion of x264 capabilities at [this post][x264 low latency] which I higly recommend. Check the comments for actual settings, there were some *really* good comments there. There are discussions about how to calculate latency, required bandwidth, suggest settings, how some options affect quality and/or latency, and much more. The [x264 settings wiki page][] provides a very good reference for x264 options and the [x264 encoder latency] shows how to calculate the actual latency.

One problem with very low latencies is bandwidth. In order to achieve it the encoder may only process very few frames before sending them out, so any x264 optimization that requires a lot of frames has to be disabled to keep buffering to a minimum, because of that quality will be much lower given the same bandwidth. For desktop streaming we can only lower bandwidth so much before text becomes unreadable.

Most comments ([this][comment2408], [this][comment2505], and [this][comment2509]) suggest using `crf` with the proper `vbv-maxrate` and `vbv-bufsize`. For desktop streaming I found out that this works well for high bandwidth scenarios, for they introduce some annoying artifacts on the stream under low bandwidth. I would stick with `qp` in place of `crf` otherwise - note that using `qp` means we can't set any *vbv* options (they are simply ignored).

I didn't notice any particular difference using `intra-refresh` and `slice-max-size` with `zerolatency` enabled. Also `vbv-maxrate` and `vbv-bufsize` have to be carefully picked when using `crf`, too low and I ended up with vbv overflow problems. I decided to keep all *vbv* settings out, the x264 seemed to pick sane defaults for both. Of course, all that might be due to the late hour of the night. Let me know if you think different.

In case a 1 second or higher latency is acceptable, do not use `-tune zerolatency` and rely instead in `intra-refresh`, `slice-max-size`, `vbv-maxrate`, `vbv-bufsize`, and/or more demanding presets like `fast` or `slower`. This will reduce the required bandwidth and produce a much higher quality video while still keeping latency in the range of 1 to 5 seconds.

### Conclusion ###
[FFmpeg][] allows us to achieve very low latency with high quality, the only downside being higher bandwidth usage. The client also plays an important role, so stick with faster players such as [FFplay][].

* * *

#### References #####
I relied on the following guides for testing out the various codecs:
 * [Grab the desktop with FFmpeg][]
 * [FFmpeg streaming guide][]
 * [The awesome post introducing low latency for x264 by Dark Shikari][x264 low latency]

[desktop streaming with vlc]: /blog/2013/06/07/desktop-streaming-with-vlc/ "Desktop streaming with VLC"
[ffmpeg]: http://www.ffmpeg.org/ "FFmpeg is a complete, cross-platform solution to record, convert and stream audio and video."
[ffplay]: http://ffmpeg.org/ffplay.html "FFplay is a very simple and portable media player using the FFmpeg libraries and the SDL library."
[x264 settings wiki page]: http://mewiki.project357.com/wiki/X264_Settings "a comprehensive guide to options in x264"
[x264 encoder latency]: http://mewiki.project357.com/wiki/X264_Encoding_Suggestions#Encoder_latency "improving x264 latency"
[vlc]: http://www.videolan.org/vlc/ "VLC media player"
[ffmpeg streaming guide]: http://ffmpeg.org/trac/ffmpeg/wiki/StreamingGuide "FFmpeg Streaming Guide"
[x264 low latency]: http://x264dev.multimedia.cx/archives/249 "Introducing low latency for x264 by Dark Shikari"
[comment2408]: http://x264dev.multimedia.cx/archives/249#comment-2408
[comment2505]: http://x264dev.multimedia.cx/archives/249#comment-2505
[comment2509]: http://x264dev.multimedia.cx/archives/249#comment-2509
[x264 FFmpeg Options Guide]: https://sites.google.com/site/linuxencoding/x264-ffmpeg-mapping
