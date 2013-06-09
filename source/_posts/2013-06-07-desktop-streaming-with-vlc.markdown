---
layout: post
title: "Desktop streaming with VLC"
date: 2013-06-07 17:45
comments: true
categories:
- VLC
- VNC
- Desktop streaming
author: Tiago Stürmer Daitx
---

Stream your desktop with VLC using the command line.

## tl;dr ##
For smallest possible latency:
{% codeblock  %}
$ cvlc screen:// --screen-fps=25 --live-caching=10 --sout "#transcode{vcoded=mp2v,vb=256,fps=25,acodec=none}:std{access=udp{caching=10},mux=raw,dst=192.168.0.10:1234}"
{% endcodeblock %}

On the client side run `vlc udp://@:1234`.

For lowest CPU usage:
{% codeblock  %}
$ cvlc screen:// --screen-fps=5 --live-caching=100 --sout "#transcode{venc=x264{preset=ultrafast,tune=zerolatency,qp=10,nr=1000},vcodec=x264,vb=100,fps=5,acodec=none}:std{access=http,mux=ts,dst=/stream}"
{% endcodeblock %}

On the client side run `vlc http://<host>:8080/stream`

## Motivation ##

While investigating ways to share my desktop for remote presentations I found out that VLC is able to stream my desktop. I figured out that it might be a good alternative to read-only VNC sessions for some settings and got into testing it in my local network. I also did a simple comparison of it against [x11vnc][].

Now, some VLC setup.
<!-- more -->

## Machine specs ##
Everything was tested on a Sony Vaio VPC-SA35GB, Core i5 processor, running Fedora 18 x64 on a single screen with resolution 1600x900.

## Latency ##
I experienced latency as high as 20 seconds and as low as half a second (maybe even less, I had no way to measure something that low). Latency depends on: audio, client cache, server codec settings, muxing type, transport protocol, and server transport cache.

### Audio ###
Unless you can live with a higher latency, it is highly recommended to disable audio. I have seen latency grow from 3 seconds to 10+ seconds because I enable audio. There might be some optimum combination of options on the codecs to get a smaller latency with audio, but I didn't try it - let me know if you do know or try it.

### Client side latency ###
It's really importante to properly configure the client cache in order to reduce latency. The actual value will depend on the network: a smaller cache leads to lower delays but clients might stagger and the result will seem like an even bigger latency; a bigger cache will help solving staggering problems but video delays will get higher. I recommend you test it beforehand, if at all possible try to emulate your slowest client in the network.

On a private Wifi network I usually set the client cache from 20 ms to 100 ms.

### Server side latency ###
The x264 codec can intruce a high latency depending on the preset chosen. In all examples I used the ultrafast preset, there are [other presets available][x264 settings wiki page].

Muxing also introduces latency. The TS muxer is the usual choice for network streaming, since it was build to be resilient to data loss. The problem is that it introduces (at least) around 1-2 seconds of latency. The other option is to use the RAW muxer, but only few codecs support that: according to the [VLC streaming features][] it is supposed to work with mp1v, mp2v, and mp4v. I was only ble to get mp1v and mp2v working with it, mp4v did not work.

The protocol also affects how much latency you get. In my tests HTTP was slower than UDP by at least 2 seconds. UDP also allows the server to configure caching, with enables really slow latency when combined with the right codec and muxer.

## VLC server codecs ##
I tested 4 video codecs: x264, mp4v, mp2v, and mp1v.

### x264 ####
On a static screen, x264 gave me the lowest bandwidth, using around 30 kbit/s, but it also got over 2.5 mbit/s with high dynamic content - eg. playing a video. Latency can be kept as low as 3 seconds for UDP and 5 seconds for HTTP by using the proper preset (ultrafast). CPU usage was usually steady, but from times to time there were peaks even on static content. CPU usage didn't change much when the screen changed by a small ammount. Overall CPU usage by x264 seemed to be lower than all other tested codecs.

### mp4v, mp2v, and mp1v ####
Bandwidth for static screen:

 **mp4v:** 1 mbit/s ❘ **mp2v:** 512 kbit/s ❘ **mp1v:** 64 kbit/s

High dynamic content (video) demands a much higher bandwidth: 4~5 mbit/s using the settings bellow. mp1v demanded the most CPU, while mp4v demanded the least amount of CPU. mp2v stayed in the middle ground.

### Frames per second ### 
Minimum fps exist for both mp1v and mp2v codecs: you must use at least 25 fps, see [MPEG-1 and 2 wiki page][mpeg-1 wiki]. Otherwise I recommend sticking to 5 fps for mp4v and x264, this allows the fewest CPU usage while providing good feedback for the clients - for some reason VLC clients did not respond well when I set fps lower than 5.

## Server side examples ##
For the smalest latency (&lt; 1 second) and medium to high CPU usage the mp2v+UDP+RAW+noaudio combination provided the best choice. Use it on a network where you know latency and data loss to be low. I was able to set client cache to 30 ms in a private Wifi network. mp1v can be used instead of mp2v, but it uses more CPU and seems to provide lower video quality.
{% codeblock  %}
$ cvlc screen:// --screen-fps=25 --live-caching=10 --sout "#transcode{vcoded=mp2v,vb=256,fps=25,acodec=none}:std{access=udp{caching=10},mux=raw,dst=192.168.0.10:1234}"
{% endcodeblock %}
BTW, you might get a message from VLC complaining that UDP should be used with TS muxer, ignore it. According to the [VLC streaming features] it is a valid combination.

For lowest CPU usage, HTTP streaming with x264, no audio, and 5 fps did a great job. I got ~5 seconds latency with a client cache of 100 ms.
{% codeblock  %}
$ cvlc screen:// --screen-fps=5 --live-caching=100 --sout "#transcode{venc=x264{preset=ultrafast,tune=zerolatency,qp=10,nr=1000},vcodec=x264,vb=100,fps=5,acodec=none}:std{access=http,mux=ts,dst=/stream}"
{% endcodeblock %}

Or use mp4v
{% codeblock  %}
$ cvlc screen:// --screen-fps=5 --sout "#transcode{vcodec=mp4v,vb=64,fps=5,acodec=none}:std{access=http,mux=ts,dst=/stream}"
{% endcodeblock %}

Use UDP streaming to get an even lower latency (~3 secs) with the same low CPU usage
{% codeblock  %}
$ cvlc screen:// --screen-fps=5 --sout "#transcode{vcodec=mp4v,acodec=none,fps=5}:std{access=udp,mux=ts,dst=192.168.0.1:2345}"
{% endcodeblock %}

To add audio from Pulseaudio use
{% codeblock  %}
$ cvlc screen:// --screen-fps=5 --input-slave=pulse:// --sout "#transcode{vcodec=mp4v,acodec=mp3,vb=64,ab=32,fps=5,audio-sync}:std{access=udp,mux=ts,dst=192.168.0.1:2345}"
{% endcodeblock %}

See more streaming and codec options on the [VLC streaming howto][] and [VLC codecs wiki page][].

## Watching the stream ##
Watch a HTTP stream
{% codeblock  %}
$ vlc http://<host>:8080/stream
{% endcodeblock %}

Watch a UDP stream 
{% codeblock %}
$ vlc udp://@:2345
{% endcodeblock %}

## Comparison with VNC ##
I did a very simple comparison using the default [x11vnc][] settings and [Remmina][].

**Latency:**
VNC session latency was in the order of milliseconds while VLC was on the order of seconds for most scenarios.

**Bandwidth:**
VNC used as much as 32 mbit/s for bandwidth in high content settings and around 30 kbits/s on static content. VLC streaming never got past 5 mbit/s (remember it was constrained to 5 fps), but static content might take 1 mbit/s or more depending on the codec - there's the option to use UDP multicast if the network allows for it.

**Content quality:**
VNC has the highest quality. VLC quality is highly dependent on the coded, from the highest quality to lowest: x264, mp4v, mp2v, and mp1v.

**CPU usage:**
VNC kept CPU usage to a minimum. VLC depends on the selected codec, but it's usually way higher than VNC.

**Encryption:**
VNC allows connection encryption and VLC allows [streaming over HTTPS][vlc streaming howto].

**Authentication:**
VNC allows for password and/or certificate authentication. VLC streaming provides [basic HTTP authentication][vlc streaming howto] - it is highly recommended that HTTPS is enabled as well, otherwise the user and password will be sent without encryption.

## Conclusion ##
It seems that it is feasible to rely on VLC streaming for presentations under some settings. Latency seems to be the biggest issue, so be sure that your audience and presentation style can handle delays as high as 20 seconds.

Using HTTP also enables any browser (with the proper plugin) to access the presentation as well. VNC also supports browser, either using the java plugin (I find that a hassle) or a HTML5 client such as [noVNC][] (requires server support or a WebSocket proxy).

Remember that VNC can use better compression algorithms and both server and client can limit the color depth to limit bandwidth usage. This means each client can be configured in a way to make better use of the available network - of course, the server must be able to support the client settings.

## Future ##
Those tests were all run in a private wifi network. I intent to do some testing over slower networks to compare VNC sharing and VLC streamming in terms of quality and reliability.

I have also been playing with ffmpeg directly and results are even better than VLC. **Update:** see [Desktop streaming with FFmpeg for lower latency][desktop streaming with ffmpeg]

Keep posted.

* * *

#### References #####
I first found about desktop streaming using VLC in [lnostdal's blog](http://blog.nostdal.org/2012/10/vlc-h264-desktop-and-audio-recording-or.html "VLC H264 desktop and audio recording or streaming via HTTP on Linux"). From there I learned about the various streaming options in the [VLC streaming howto][]. The [VLC streaming features][] described both available and valid combinations of video, audio, muxing type, and transport method. Codec names and a simple transcoding example are described at [VLC codecs wiki page][]. I found everything I needed about the x264 codec options at the [x264 settings wiki page][].

[My original post on Google+](https://plus.google.com/109931565080058432516/posts/hrUGwzJ6UNt) and a nice guide for [desktop streaming using VLC GUI wizard](http://grok.lsu.edu/article.aspx?articleid=14625) posted not long ago.

[desktop streaming with ffmpeg]: /blog/2013/06/09/desktop-streaming-with-ffmpeg-for-lower-latency/ "Desktop streaming with FFmpeg for lower latency"
[x11vnc]: http://www.karlrunge.com/x11vnc/ "x11vnc a VNC server for real X displays"
[remmina]: http://remmina.sourceforge.net "Remmina VNC client"
[novnc]: http://kanaka.github.io/noVNC/‎ "HTML5 VNC Client"
[vlc streaming howto]: https://www.videolan.org/doc/streaming-howto/en/ch03.html "VLC - Chapter 3: Advanced streaming using the command line"
[vlc streaming features]: https://www.videolan.org/streaming-features.html "Streaming features list: video, audio, muxing, transport"
[vlc codecs wiki page]: http://wiki.videolan.org/Codec "VLC codecs list"
[mpeg-1 wiki]: http://wiki.videolan.org/MPEG-1#MPEG-1_and_2 "MPEG-1 and 2 wiki page"
[x264 settings wiki page]: http://mewiki.project357.com/wiki/X264_Settings "a comprehensive guide to options in x264"
