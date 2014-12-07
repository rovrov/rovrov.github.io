---
layout: post
category: blog
tags: [c#, network]
date: 2014-12-06 12:00:00 -0500
---
{% include JB/setup %}

Apparently, limiting file download speed it is not straightforward with the standard web helper classes provied by .NET - [WebClient](http://www.codeproject.com/Questions/163737/Webclient-download-speed-limit) or [HttpWebResponse](http://stackoverflow.com/questions/15449147/limiting-httpwebresponse-stream-reading-speed). Someone even re-implemented a [socket-based HTTP client](http://www.codeproject.com/Articles/100769/Socket-based-HTTP-Client-with-Bandwidth-Limit) to add this feature. While the latter might be a solution, relying on an incomplete and potentially buggy HTTP implementation is not something I like. [Libcurl](http://en.wikipedia.org/wiki/CURL#libcurl), on the other hand, has been around for ages and also has an option to limit the download speed. The library is ported to many OSes including Windows and wrapped for .NET as well, so I gave it a try.

<!-- more -->

[CurlSharp](https://github.com/masroore/CurlSharp), an improved fork of the official [libcurl.NET](http://sourceforge.net/projects/libcurl-net/), looked like a good start. The use is pretty straightforward - the wrapper comes with a few examples and pre-compiled libcurl binaries including dependencies:

{% highlight bash %}
libcurl.dll      libeay32.dll  msvcr120.dll  zlib1.dll
libcurlshim.dll  libssh2.dll   ssleay32.dll
{% endhighlight %}

The curl option for limiting download speed was missing from the wrapper at the time of writing, but it can be  easily added. According to [CURLOPT_MAX_RECV_SPEED_LARGE docs](http://curl.haxx.se/libcurl/c/CURLOPT_MAX_RECV_SPEED_LARGE.html), the corresponding data type is `curl_off_t` which [seems to match](http://curl.haxx.se/dev/readme-curl_off_t.html) the signed 64-bit type, i.e. C# `long`.

{% highlight c# %}
// ----- CurlEasy.cs -----
private long _maxRecvSpeedLarge;

public long MaxRecvSpeedLarge
{
    get { return _maxRecvSpeedLarge; }
    set { setLongOption(CurlOption.MaxRecvSpeedLarge, ref _maxRecvSpeedLarge, value); }
}

// ----- CurlOption.cs -----
MaxRecvSpeedLarge = 30146, // CURLOPT_MAX_RECV_SPEED_LARGE
{% endhighlight %}

Now this can be used, for example in the [EasyGet](https://github.com/masroore/CurlSharp/blob/master/Samples/EasyGet/EasyGet.cs) sample:

{% highlight c# %}
easy.BufferSize = 4096;
easy.MaxRecvSpeedLarge = 10*1024;
{% endhighlight %}

Setting a buffer size smaller than default makes it easier to see the data chunks going in at the reduced rate. Also, as the speed limit is most likely implemented through throttling, a big buffer with a small speed limit would yield in a large delay between downloading the chunks and the connection might timeout.

It is worth noting that when the speed is limited this way, the actual network usage is likely to be non-constant over time (contain bursts) because of the additional network buffering by OS. For example, here is what it looks like on my Windows 8.1 laptop:

<img src="/files/2014-12-06-limiting-file-download-speed-in-csharp/network_usage.png" class="imgmb" alt="Network usage"/>