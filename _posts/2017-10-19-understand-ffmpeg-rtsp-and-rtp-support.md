---
layout:     post
title:      "深入理解FFMPEG的RTSP以及RTP实现"
date:       2017-10-19
author:     "winson"
tags:
    - ffmpeg
    - rtsp
    - rtp
    - Android
---

# 深入理解FFMPEG的RTSP以及RTP实现

## 0x01 概述
由于工作需要而需要与IP摄像头打交道，而IP摄像头的视频流多数都是RTSP流，工作上遇到一些深层次的问题，必须要深入理解ffmpeg对rtsp/rtp的实现，于是便有了此文。

## 0x02 lavf
RTSP/RTP的实现位于libavformat这个库，这个库大体可以认为是封装库，所有容器格式的实现都在这里，即所有的demuxer与muxer的实现都在这里。RTSP在这里被当成一个容器来实现了。

在研究代码的时候会不可避免遇到lavf架构上的一些概念，在这里先解释一下。

lavf在架构上分成两层，即demuxer/muxer与URLProtocol。
demuxer/muxer就是解复用以及封装了，URLProtocol可以认为是传输方式的实现，比如tcp,udp,rtp,http等这些都被认为是传输方式，每一个传输方式被实现成了URLProtocol结构体的一个实例。

有人要问了，既然这样那就是所有的URL形式都应该有对应的URLProtocol实现拉，但你刚才说了RTSP被实现成容器了，这怎么解释， 那rtsp://xxxxx:xx/xxx这种格式的地址ffmpeg是怎么处理的？不实现成URLProtocol却实现成容器？

没错我也挺疑惑这个问题的，所以如果有人知道了就告诉我。。。

但毫无疑问的是RTSP确实是用容器的格式来实现的，而RTP这种更像容器的格式却被当成URLProtocol来实现了，确实有点怪。

这是我感觉ffmpeg rtsp/rtp相关代码有点乱的其中一个原因，架构上的概念在实现上对不上，不够清晰，很容易让人想错。

URLProtocol是内部数据结构，如下：

``` C
typedef struct URLProtocol {
    const char *name;
    int     (*url_open)( URLContext *h, const char *url, int flags);
    /**
     * This callback is to be used by protocols which open further nested
     * protocols. options are then to be passed to ffurl_open()/ffurl_connect()
     * for those nested protocols.
     */
    int     (*url_open2)(URLContext *h, const char *url, int flags, AVDictionary **options);
    int     (*url_accept)(URLContext *s, URLContext **c);
    int     (*url_handshake)(URLContext *c);

    /**
     * Read data from the protocol.
     * If data is immediately available (even less than size), EOF is
     * reached or an error occurs (including EINTR), return immediately.
     * Otherwise:
     * In non-blocking mode, return AVERROR(EAGAIN) immediately.
     * In blocking mode, wait for data/EOF/error with a short timeout (0.1s),
     * and return AVERROR(EAGAIN) on timeout.
     * Checking interrupt_callback, looping on EINTR and EAGAIN and until
     * enough data has been read is left to the calling function; see
     * retry_transfer_wrapper in avio.c.
     */
    int     (*url_read)( URLContext *h, unsigned char *buf, int size);
    int     (*url_write)(URLContext *h, const unsigned char *buf, int size);
    int64_t (*url_seek)( URLContext *h, int64_t pos, int whence);
    int     (*url_close)(URLContext *h);
    int (*url_read_pause)(URLContext *h, int pause);
    int64_t (*url_read_seek)(URLContext *h, int stream_index,
                             int64_t timestamp, int flags);
    int (*url_get_file_handle)(URLContext *h);
    int (*url_get_multi_file_handle)(URLContext *h, int **handles,
                                     int *numhandles);
    int (*url_get_short_seek)(URLContext *h);
    int (*url_shutdown)(URLContext *h, int flags);
    int priv_data_size;
    const AVClass *priv_data_class;
    int flags;
    int (*url_check)(URLContext *h, int mask);
    int (*url_open_dir)(URLContext *h);
    int (*url_read_dir)(URLContext *h, AVIODirEntry **next);
    int (*url_close_dir)(URLContext *h);
    int (*url_delete)(URLContext *h);
    int (*url_move)(URLContext *h_src, URLContext *h_dst);
    const char *default_whitelist;
} URLProtocol;
```

可以看到把基本的IO操作都抽象了出来，像open,read,write,close,seek这些就是文件操作，还有open_dir,read_dir这些就是文件夹操作(有些协议比如smb或ftp等就是支持这种操作)，看名字都很容易理解。URLProtocol还支持分步连接，这就是其中的url_handshake接口的作用，这个在我们下面的分析中不会用到，所以会被忽略。

有了数据结构，就必定有操作数据结构的方法。URLProtocol的使用接口是头文件avformat/url.h，接口格式几乎都是ffurl_xxx这种。我们看其中一个遇到最多的:

``` c
int ffurl_open_whitelist(URLContext **puc, const char *filename, int flags,
               const AVIOInterruptCB *int_cb, AVDictionary **options,
               const char *whitelist, const char* blacklist,
               URLContext *parent);
```

其作用是打开一个连接并返回结果。

第一个参数URLContext重点解释一下。这是个输出参数，可以认为是URLProtocol的实例，URLProtocol只有一个，而每打开一个url都对应一个URLContext。类似于打开文件会返回一个句柄，这个就是句柄的意思。看下URLContext的定义：

``` c
typedef struct URLContext {
    const AVClass *av_class;    /**< information for av_log(). Set by url_open(). */
    const struct URLProtocol *prot;
    void *priv_data;
    char *filename;             /**< specified URL */
    int flags;
    int max_packet_size;        /**< if non zero, the stream is packetized with this max packet size */
    int is_streamed;            /**< true if streamed (no seek possible), default = false */
    int is_connected;
    AVIOInterruptCB interrupt_callback;
    int64_t rw_timeout;         /**< maximum time to wait for (network) read/write operation completion, in mcs */
    const char *protocol_whitelist;
    const char *protocol_blacklist;
    int min_packet_size;        /**< if non zero, the stream is packetized with this min packet size */
} URLContext;
```

要注意的是其中的priv_data，当每次打开url的时候，会给这个priv_data分配空间，其大小为URLProtocol.priv_data_size，而每个URLProtocol都会把这个大小设为自己私有数据结构的大小，在为URLProtocol分配内存的时候，也会给priv_data分配空间，有了这个每个实例都有的priv_data，就能跟踪每个实例的情况了。具体到RTP协议，就是RTPContext这个结构。

RTP对URLProtocol的具体实现我们下面说到RTP实现的时候再分析。

## 0x03 RTSP Demuxer

> **本文只讨论demuxer的情况。**

实现文件rtspdec.c，拉到文件底部看到关键结构：

``` c
static const AVClass rtsp_demuxer_class = {
    .class_name     = "RTSP demuxer",
    .item_name      = av_default_item_name,
    .option         = ff_rtsp_options,
    .version        = LIBAVUTIL_VERSION_INT,
};

AVInputFormat ff_rtsp_demuxer = {
    .name           = "rtsp",
    .long_name      = NULL_IF_CONFIG_SMALL("RTSP input"),
    .priv_data_size = sizeof(RTSPState),
    .read_probe     = rtsp_probe,
    .read_header    = rtsp_read_header,
    .read_packet    = rtsp_read_packet,
    .read_close     = rtsp_read_close,
    .read_seek      = rtsp_read_seek,
    .flags          = AVFMT_NOFILE,
    .read_play      = rtsp_read_play,
    .read_pause     = rtsp_read_pause,
    .priv_class     = &rtsp_demuxer_class,
};
```

所有的demuxer都要实现AVInputFormat这个结构。而demuxer的操作都是那几样, probe(容器格式探测), read_header(读容器头信息), read_packet(读具体的数据), seek, close, play, pause。 

rtsp_probe非常简单，就是只要遇到rtsp开头的就百分百匹配。

``` c
static int rtsp_probe(AVProbeData *p)
{
    if (
#if CONFIG_TLS_PROTOCOL
        av_strstart(p->filename, "rtsps:", NULL) ||
#endif
        av_strstart(p->filename, "rtsp:", NULL))
        return AVPROBE_SCORE_MAX;
    return 0;
}
```

重点是rtsp_read_header这个方法。其运行过程完成了RTSP协议连接的所有过程，包括OPTIONS,DESCRIB,SETUP,PLAY。

read_header的调用发生在avformat_open_input。我们平时使用的这个函数之后，就能知道AVFormatContext里面的流信息，方法是AVFormatContext.streams成员。

于是我们重点分析rtsp demuxer是如何完成RTSP协议这四个基本过程，以及如何对接ffmpeg的数据结构的(生成AVStream的各种信息)。贴一下这个函数的代码

``` c
static int rtsp_read_header(AVFormatContext *s)
{
    RTSPState *rt = s->priv_data;
    int ret;

    ...

    if (rt->rtsp_flags & RTSP_FLAG_LISTEN) {
        ...
    } else {
        ret = ff_rtsp_connect(s);
        if (ret)
            return ret;

        ...

        if (rt->initial_pause) {
            /* do not start immediately */
        } else {
            if ((ret = rtsp_read_play(s)) < 0) {
                ff_rtsp_close_streams(s);
                ff_rtsp_close_connections(s);
                return ret;
            }
        }
    }

    return 0;
}
```

由于我们只关注demuxer的情况，故直接从ff_rtsp_connect开始看就行了，ff_rtsp_connect的实现在rtsp.c中，我把无关部分都省略了，只留重要的点。

> 以rtsp_xxx开头的函数一般是rtsp demuxer的接口函数，只是一个壳，文件是rtspdec.c，而以ff_rtsp_xxx开头的一般是具体协议逻辑的实现函数，在rtsp.c中。

``` c
int ff_rtsp_connect(AVFormatContext *s)
{
    ...
    if (s->max_delay < 0) /* Not set by the caller */
        s->max_delay = s->iformat ? DEFAULT_REORDERING_DELAY : 0;
    ...
    ff_url_join(rt->control_uri, sizeof(rt->control_uri), proto, NULL,
        host, port, "%s", path);
    ...
    if (rt->control_transport == RTSP_MODE_TUNNEL) {
        ...
    } else {
       int ret;
		/* open the tcp connection */
		ff_url_join(tcpname, sizeof(tcpname), lower_rtsp_proto, NULL,
			host, port,
			"?timeout=%d", rt->stimeout);
		if ((ret = ffurl_open_whitelist(&rt->rtsp_hd, tcpname, AVIO_FLAG_READ_WRITE,
                    &s->interrupt_callback, NULL, 
                    s->protocol_whitelist, s->protocol_blacklist, NULL)) < 0) {
			err = ret;
			goto fail;
		}
		rt->rtsp_hd_out = rt->rtsp_hd; 
    }
    rt->seq = 0;

    tcp_fd = ffurl_get_file_handle(rt->rtsp_hd);

    for (rt->server_type = RTSP_SERVER_RTP;;) {
        ...
        ff_rtsp_send_cmd(s, "OPTIONS", rt->control_uri, cmd, reply, NULL);
        ...
    }

    ...

    if (CONFIG_RTSP_DEMUXER && s->iformat)
		err = ff_rtsp_setup_input_streams(s, reply);
	else if (CONFIG_RTSP_MUXER)
		err = ff_rtsp_setup_output_streams(s, host);
	else
		av_assert0(0);
	if (err)
        goto fail;
        
    do {
        ...
        err = ff_rtsp_make_setup_request(s, host, port, lower_transport,
			rt->server_type == RTSP_SERVER_REAL ?
            real_challenge : NULL);
        ...
    } while (err);
    ...
}
```

> 分析代码的时候要时刻谨记RTSPState是RTSP demuxer的上下文对象（即priv_data，私有数据结构，其大小由AVInputFormat.priv_data_size指出），其成员可以通过ffmpeg的options机制初始化，所以很多成员你都找不到初始化的地方，好像直接上来就用了。绝大部分都是用的初始值，除非你手动赋值了，可以直接找option结构体来看，对于rtsp，就是ff_rtsp_options这个变量。

首先是上来就设置max_delay的默认值，这个值很重要，可以认为demuxer在收到数据后最多延迟这个时间才去处理数据。举个例子，因为IP摄像头的RTP流一般都是UDP的，而UDP的流是乱序的，接收方要根据RTP头的seq来重组。当reader来读的时候，他会按照前一个数据包的seq号来读下一个RTP包，当他发现下一个RTP包的seq不匹配的时候他就会等，直到下一个RTP包接收的时间跟现在对比已经超过max_delay了，这样他就不等了，直接处理这个包，这时候就会有打印出现:

``` c
av_log(s, AV_LOG_WARNING,
                "max delay(%dms) reached. need to consume packet(now=%"PRId64", wait_end=%"PRId64", delta=%"PRId64")\n", s->max_delay, now_time, wait_end, wait_end - now_time);
```

在一般网络情况ok或者码流不大的情况下。除非你码流已经超过你cpu的处理能力了，或者wifi垃圾导致经常丢包，否则一般是不会出现这个问题。

然后是设置control_uri，熟悉RTSP的这个应该不会有疑问，就是RTSP的控制命令都发往这个地址。

然后判断了一下传输模式，加密传输我们先不提。然后直接ffurl_open了RTSP的tcp连接。完了后直接想control_uri发送了OPTIONS命令。

ff_rtsp_send_cmd这个函数很简单，发送命令后会等待回复再返回。

发送完OPTIONS，然后就是DESCRIBE命令了。

ff_rtsp_setup_input_streams这个函数里面会发送DESCRIBE命令，并分析返回的SDP报文.

``` c
int ff_rtsp_setup_input_streams(AVFormatContext *s, RTSPMessageHeader *reply)
{
    ...
    ff_rtsp_send_cmd(s, "DESCRIBE", rt->control_uri, cmd, reply, &content);
    ...
    /* now we got the SDP description, we parse it */
    ret = ff_sdp_parse(s, (const char *)content);
    ...
}
```

异常清晰，发送命令，分析回复。下面来看看对SDP报文的分析。

``` c
int ff_sdp_parse(AVFormatContext *s, const char *content)
{
    ...
    for (;;) {
        ...
        sdp_parse_line(s, s1, letter, buf);
        ...
    }
}
```

ff_sdp_parse这个方法只有sdp_parse_line这一行值得关注。

``` c
static void sdp_parse_line(AVFormatContext *s, SDPParseState *s1,
	int letter, const char *buf)
{
    ...
    switch (letter) {
    case 'c':
        ...
    case 's':
        ...
    case 'i':
        ...
    case 'm':
        ...
        if (!strcmp(st_type, "audio")) {
			codec_type = AVMEDIA_TYPE_AUDIO;
		}
		else if (!strcmp(st_type, "video")) {
			codec_type = AVMEDIA_TYPE_VIDEO;
		}
		else if (!strcmp(st_type, "application")) {
			codec_type = AVMEDIA_TYPE_DATA;
		}
		else if (!strcmp(st_type, "text")) {
			codec_type = AVMEDIA_TYPE_SUBTITLE;
        }
        ...
        rtsp_st = av_mallocz(sizeof(RTSPStream));
        ...
        if (!strcmp(ff_rtp_enc_name(rtsp_st->sdp_payload_type), "MP2T")) {
            ...
        } else if...{
            ...
        } else {
            st = avformat_new_stream(s, NULL);
            ...
            rtsp_st->stream_index = st->index;
            ...
        }
    case 'a':
        if (av_strstart(p, "control:", &p)) {
            ...
        } else if (av_strstart(p, "rtpmap:", &p) && s->nb_streams > 0) {
            /* NOTE: rtpmap is only supported AFTER the 'm=' tag */
			get_word(buf1, sizeof(buf1), &p);
			payload_type = atoi(buf1);
			rtsp_st = rt->rtsp_streams[rt->nb_rtsp_streams - 1];
            if (rtsp_st->stream_index >= 0) {
				st = s->streams[rtsp_st->stream_index];
				sdp_parse_rtpmap(s, st, rtsp_st, payload_type, p);
            }
            s1->seen_rtpmap = 1;
			if (s1->seen_fmtp) {
				parse_fmtp(s, rt, payload_type, s1->delayed_fmtp);
			}
        } else if (av_strstart(p, "fmtp:", &p) ||
            ...
            if (s1->seen_rtpmap) {
				parse_fmtp(s, rt, payload_type, buf);
            }
            ...
        }
}
```

只分析m与a标签。
一个m就是一个AVStream，取决于m的类型。`rtsp_st->stream_index = st->index`这句关联起了RTSPStream与AVStream。
然后是跟在标签m后面的a属性，这个是关键点，关注其中的`sdp_parse_rtpmap`与`parse_fmtp`，据RFC 6184，rtpmap参数提供了payload使用的编码与时基,而对于H264来说fmtp则提供了额外的编码信息，包括profile与level，sps与pps等。

sdp_parse_rtpmap这个函数比较简单，就是分析是哪个编码然后给AVStream选择相应的codecid，然后再设置一下timebase。

而parse_fmtp这个则与具体的编码相关，因为编码参数不同的编码器肯定不一样，所以lavf又搞出了一个Dynamic Handler的机制，用来处理RTP包中不同编码fmtp行的参数分析。

对于H264，就对应到rtpdec_h264.c。

``` c
RTPDynamicProtocolHandler ff_h264_dynamic_handler = {
    .enc_name         = "H264",
    .codec_type       = AVMEDIA_TYPE_VIDEO,
    .codec_id         = AV_CODEC_ID_H264,
    .need_parsing     = AVSTREAM_PARSE_FULL,
    .priv_data_size   = sizeof(PayloadContext),
    .parse_sdp_a_line = parse_h264_sdp_line,
    .close            = h264_close_context,
    .parse_packet     = h264_handle_packet,
};
```

看看parse_fmtp这个函数：

``` c
static void parse_fmtp(AVFormatContext *s, RTSPState *rt,
	int payload_type, const char *line)
{
	int i;

	for (i = 0; i < rt->nb_rtsp_streams; i++) {
		RTSPStream *rtsp_st = rt->rtsp_streams[i];
		if (rtsp_st->sdp_payload_type == payload_type &&
			rtsp_st->dynamic_handler &&
			rtsp_st->dynamic_handler->parse_sdp_a_line) {
			rtsp_st->dynamic_handler->parse_sdp_a_line(s, i,
				rtsp_st->dynamic_protocol_context, line);
		}
	}
}
```

于是转到parse_sdp_a_line这个函数：

``` c
static int parse_h264_sdp_line(AVFormatContext *s, int st_index,
                               PayloadContext *h264_data, const char *line)
{
    ···
    } else if (av_strstart(p, "fmtp:", &p)) {
        return ff_parse_fmtp(s, stream, h264_data, p, sdp_parse_fmtp_config_h264);
    }
    ···
}
```

这个函数是一个壳，重点是后面那个sdp_parse_fmtp_config_h264：

``` c
static int sdp_parse_fmtp_config_h264(AVFormatContext *s,
                                      AVStream *stream,
                                      PayloadContext *h264_data,
                                      const char *attr, const char *value)
{
    AVCodecParameters *par = stream->codecpar;

    if (!strcmp(attr, "packetization-mode")) {
        av_log(s, AV_LOG_DEBUG, "RTP Packetization Mode: %d\n", atoi(value));
        h264_data->packetization_mode = atoi(value);
        /*
         * Packetization Mode:
         * 0 or not present: Single NAL mode (Only nals from 1-23 are allowed)
         * 1: Non-interleaved Mode: 1-23, 24 (STAP-A), 28 (FU-A) are allowed.
         * 2: Interleaved Mode: 25 (STAP-B), 26 (MTAP16), 27 (MTAP24), 28 (FU-A),
         *                      and 29 (FU-B) are allowed.
         */
        if (h264_data->packetization_mode > 1)
            av_log(s, AV_LOG_ERROR,
                   "Interleaved RTP mode is not supported yet.\n");
    } else if (!strcmp(attr, "profile-level-id")) {
        if (strlen(value) == 6)
            parse_profile_level_id(s, h264_data, value);
    } else if (!strcmp(attr, "sprop-parameter-sets")) {
        int ret;
        if (value[strlen(value) - 1] == ',') {
            av_log(s, AV_LOG_WARNING, "Missing PPS in sprop-parameter-sets, ignoring\n");
            return 0;
        }
        par->extradata_size = 0;
        av_freep(&par->extradata);
        ret = ff_h264_parse_sprop_parameter_sets(s, &par->extradata,
                                                 &par->extradata_size, value);
        av_log(s, AV_LOG_DEBUG, "Extradata set to %p (size: %d)\n",
               par->extradata, par->extradata_size);
        return ret;
    }
    return 0;
}
```

然后这里就完成了fmtp行的解析。ff_h264_parse_sprop_parameter_sets完成对sps与pps的解析。就是很简单的先base64解码然后copy数据。

在分析完SDP后，我们就有了新的AVStream，以及对应这个AVStream的一些参数。于是ff_rtsp_setup_input_streams这个函数就完成功能了。

DESCRIBE完了之后就是SETUP了。

``` c
int ff_rtsp_make_setup_request(AVFormatContext *s, const char *host, int port,
	int lower_transport, const char *real_challenge)
{
    ...
    for (j = rt->rtp_port_min + port_off, i = 0; i < rt->nb_rtsp_streams; ++i) {
        ...
        /* RTP/UDP */
		if (lower_transport == RTSP_LOWER_TRANSPORT_UDP) {
            ...
			while (j <= rt->rtp_port_max) {
				...
				ff_url_join(buf, sizeof(buf), "rtp", NULL, host, -1,
					"?localport=%d", j);
				/* we will use two ports per rtp stream (rtp and rtcp) */
				j += 2;
				err = ffurl_open_whitelist(&rtsp_st->rtp_handle, buf, AVIO_FLAG_READ_WRITE,
					&s->interrupt_callback, &opts, s->protocol_whitelist, s->protocol_blacklist, NULL);
				...
            }
            ...
        }
        ...
        ff_rtsp_send_cmd(s, "SETUP", rtsp_st->control_url, cmd, reply, NULL);
        ...
        if ((err = ff_rtsp_open_transport_ctx(s, rtsp_st)))
			goto fail;
    }
}
```

拼接一个rtp://xxxx/xxx的连接，然后用ffurl_open打开这个连接。

既然是用ffurl_open打开的rtp开头的连接，那就说明肯定有rtp的URLProtocol实现，位于rtpproto.c：

``` c
const URLProtocol ff_rtp_protocol = {
	.name = "rtp",
	.url_open = rtp_open,
	.url_read = rtp_read,
	.url_write = rtp_write,
	.url_close = rtp_close,
	.url_get_file_handle = rtp_get_file_handle,
	.url_get_multi_file_handle = rtp_get_multi_file_handle,
	.priv_data_size = sizeof(RTPContext),
	.flags = URL_PROTOCOL_FLAG_NETWORK,
	.priv_data_class = &rtp_class,
};
```

其中的rtp_open函数：

``` c
static int rtp_open(URLContext *h, const char *uri, int flags)
{
    ...
    for (i = 0; i < max_retry_count; i++) {
        ...
        build_udp_url(s, buf, sizeof(buf),
                      hostname, rtp_port, s->local_rtpport,
                      sources, block);
        if (ffurl_open_whitelist(&s->rtp_hd, buf, flags, &h->interrupt_callback,
                                 NULL, h->protocol_whitelist, h->protocol_blacklist, h) < 0)
            goto fail;
        ...
        build_udp_url(s, buf, sizeof(buf),
                      hostname, s->rtcp_port, s->local_rtcpport,
                      sources, block);
        if (ffurl_open_whitelist(&s->rtcp_hd, buf, rtcpflags, &h->interrupt_callback,
                                 NULL, h->protocol_whitelist, h->protocol_blacklist, h) < 0)
            goto fail;
        break;
    }
}
```

很简单，先建立rtp连接，再建立rtcp连接，两个都是udp连接（然而UDP并木有连接，只是建立了socket），所以ffurl_open的时候会调用udp URLProtocol的url_open，位于udp.c：

``` c
static int udp_open(URLContext *h, const char *uri, int flags)
{
    ...
    udp_fd = udp_socket_create(h, &my_addr, &len, localaddr);
    ···
    ff_socket_nonblock(udp_fd, 1);
    ...
}
```

在建立好rtp以及rtcp两个udp连接后，rtp_open的任务也完成了，然后就返回到了ff_rtsp_make_setup_request。连接建立好之后才发出SETUP。然后调用这个函数：

``` c
int ff_rtsp_open_transport_ctx(AVFormatContext *s, RTSPStream *rtsp_st)
{
    ...
    else if (CONFIG_RTPDEC)
		rtsp_st->transport_priv = ff_rtp_parse_open(s, st, rtsp_st->sdp_payload_type,
            reordering_queue_size);
    ...
}
```

RTSP可以由多种方式来传输实际的音视频数据，可以使用RTP,RDP, 甚至RAW格式的都行（即不使用RTP封装，直接UDP发数据）。RTSPStream里面有个成员叫transport_priv，代表具体的传输方式的实现。

这个函数看函数名字就知道应该是建立RTSPStream的transport_priv成员了，对于Input RTP via UDP来说，transport context就是RTPDemuxContext这个对象，其中包含有RTCP需要的统计信息。本文暂时不分析RTCP相关，感兴趣的可以先去看RFC3550然后对照着协议来看实现。另外还有在接收数据过程中需要用到的各种变量，比如基础信息payload_type、seq、ssrc， RTP重组需要用到的RTPPacket，数据加解密相关的SRTPContext，对payload做分析用到的dynamic handler（比如从RTP payload中分离出H264 NALU）， 以及跟RTCP有关的一些数据结构。总之这个RTPDemuxContext包含了解析RTP包的一切。是非常重要的一个结构。

这样SETUP也完成了，从ff_rtsp_open_transport_ctx经过ff_rtsp_make_setup_request，一路返回到了一开始的地方，再次贴一下rtsp_read_header函数：

``` c
static int rtsp_read_header(AVFormatContext *s)
{
    ...
    ret = ff_rtsp_connect(s);
    ...
    if (rt->initial_pause) {
            /* do not start immediately */
    } else {
        if ((ret = rtsp_read_play(s)) < 0) {
            ff_rtsp_close_streams(s);
            ff_rtsp_close_connections(s);
            return ret;
        }
    }
    ...
}
```

rt是RTSPState，其中的成员initial_pause可以从ff_rtsp_options看出有初始值0，所以会直接调用rtsp_read_play：

``` c
static int rtsp_read_play(AVFormatContext *s)
{
    ...
    ff_rtsp_send_cmd(s, "PLAY", rt->control_uri, cmd, reply, NULL);
    ...
}
```

在经过上面一大堆准备后，play函数就简单得多了，总体来说就是一些兼容性代码，然后发送一个PLAY命令，就完了。

也就是说，在我们调用avformat_open_input这个函数之后，RTP流已经开始传输了。

## 0x03 RTP数据的分析及其IO模型

通常，我们会从av_read_frame开始分析。

首先我们要明白一件事情，就是公共API与私有API是怎么一回事。在C++,JAVA等语言中，由于有语言机制来解决这方面的需求，就没那么多奇怪的代码出现，显得比较统一。但在C语言里面就不一样了，你需要自己想一个办法来区分开公共API与私有API，为啥呢？因为公共API是不能随便变的（包括结构体成员的增删），否则会产生不兼容的情况，库一升级了妈蛋以前代码不能用了这不是坑爹么。

在FFMPEG这里，解决办法就是将公共的成员放到公共的结构体里，私有成员放到私有结构体里。我们经常接触到的AVFormatContext, AVStream, AVCodec，这些就是公共API结构体，其成员几乎都是固定不变的，你可以直接使用，只有在大版本变化的时候才可能会变。

于是我们可以看到类似AVFormatInternal以及AVStreamInternal这种东西的存在，它们都是对应于其公共API的，只给内部使用的数据结构。没有头文件提供，不能直接使用。这些数据结构改了也不会引起二进制兼容的问题。

为啥要说这个呢，因为FFMPEG里面都是这种设计，结构体太多搞不清楚相关关系很容易让人头晕。

好了继续。

在去掉所有读数据无关部分后，我们关注了read_frame_internal这个函数：

``` c
int av_read_frame(AVFormatContext *s, AVPacket *pkt)
{
    ...
    if (!genpts) {
        ret = s->internal->packet_buffer
              ? read_from_packet_buffer(&s->internal->packet_buffer,
                                        &s->internal->packet_buffer_end, pkt)
              : read_frame_internal(s, pkt);
        ...
    }
    ...
}

static int read_frame_internal(AVFormatContext *s, AVPacket *pkt)
{
    ...
    while (!got_packet && !s->internal->parse_queue) {
        AVStream *st;
        AVPacket cur_pkt;

        /* read next packet */
        ret = ff_read_packet(s, &cur_pkt);
        ...
    }
    ...
}
```

于是又一个函数进入我们的眼中，ff_read_packet：

``` c
int ff_read_packet(AVFormatContext *s, AVPacket *pkt)
{
    ...
    ret = s->iformat->read_packet(s, pkt);
    ...
}
```

iformat自然就是我们的类型为AVInputFormat的demuxer ff_rtsp_demuxer了。

于是就进入到我们的rtspdec.c了：

``` c
static int rtsp_read_packet(AVFormatContext *s, AVPacket *pkt)
{
    ...
    ret = ff_rtsp_fetch_packet(s, pkt);
    ...
}
```

这个函数只需要关注ff_rtsp_fetch_packet这一行。

``` c
int ff_rtsp_fetch_packet(AVFormatContext *s, AVPacket *pkt)
{
    ...
    switch (rt->lower_transport) {
        ...
        case RTSP_LOWER_TRANSPORT_UDP:
        case RTSP_LOWER_TRANSPORT_UDP_MULTICAST:
            len = udp_read_packet(s, &rtsp_st, rt->recvbuf, RECVBUF_SIZE, wait_end);
            if (len > 0 && rtsp_st->transport_priv && rt->transport == RTSP_TRANSPORT_RTP)
                ff_rtp_check_and_send_back_rr(rtsp_st->transport_priv, rtsp_st->rtp_handle, NULL, len);
            break;
        ...
    }
    ...
    } else if (rt->transport == RTSP_TRANSPORT_RTP) {
        ret = ff_rtp_parse_packet(rtsp_st->transport_priv, pkt, &rt->recvbuf, len);
        ...
    }
}
```

看函数名字就是读一个数据包，然后分析，然后返回分析后的AVPacket。总体来说，是这样的，但是具体的实现会有点那啥，因为就函数名字的话udp_read_packet我肯定以为是仅读UDP包了，谁知道其实这里面是所有的IO操作呢，包括RTSP包的接收，以及RTP/RTCP数据包的接收与回复。

``` c
static int udp_read_packet(AVFormatContext *s, RTSPStream **prtsp_st,
                           uint8_t *buf, int buf_size, int64_t wait_end)
{
    ...

    //收集需要处理的fd
    if (rt->rtsp_hd) {
        p[rt->max_p].fd = ffurl_get_file_handle(rt->rtsp_hd);
        p[rt->max_p++].events = POLLIN;
    }
    for (i = 0; i < rt->nb_rtsp_streams; i++) {
        rtsp_st = rt->rtsp_streams[i];
        if (rtsp_st->rtp_handle) {
            if (ret = ffurl_get_multi_file_handle(rtsp_st->rtp_handle,
                                                    &fds, &fdsnum)) {
                av_log(s, AV_LOG_ERROR, "Unable to recover rtp ports\n");
                return ret;
            }
            if (fdsnum != 2) {
                av_log(s, AV_LOG_ERROR,
                        "Number of fds %d not supported\n", fdsnum);
                return AVERROR_INVALIDDATA;
            }
            for (fdsidx = 0; fdsidx < fdsnum; fdsidx++) {
                p[rt->max_p].fd       = fds[fdsidx];
                p[rt->max_p++].events = POLLIN;
            }
            av_freep(&fds);
        }
    }
    ...
    //一起poll
    for(;;){
        ...
        n = poll(p, rt->max_p, POLL_TIMEOUT_MS);
        if (n > 0) {
            //先读RTP/RTCP包
            for (i = 0; i < rt->nb_rtsp_streams; i++) {
                ...
                if (p[j].revents & POLLIN || p[j+1].revents & POLLIN) {
                    ret = ffurl_read(rtsp_st->rtp_handle, buf, buf_size);
                    if (ret > 0) {
                        *prtsp_st = rtsp_st;
                        return ret;
                    }
                }                
            }
        }
        //再读RTSP包
#if CONFIG_RTSP_DEMUXER
        if (rt->rtsp_hd && p[0].revents & POLLIN) {
            return parse_rtsp_message(s);
        }
#endif
        ...
    }
}
```

采用阻塞的方式来读，有超时时间，默认是100ms。由于我们是客户端，读到RTSP包直接无视，继续poll，除非对面返回不OK。读到RTP/RTCP数据后进入到ff_rtp_parse_packet开始分析。

``` c
int ff_rtp_parse_packet(RTPDemuxContext *s, AVPacket *pkt,
                        uint8_t **bufptr, int len)
{
    ...
    rv = rtp_parse_one_packet(s, pkt, bufptr, len);
    ...
}

static int rtp_parse_one_packet(RTPDemuxContext *s, AVPacket *pkt,
                                uint8_t **bufptr, int len)
{
    ...
    if ((buf[0] & 0xc0) != (RTP_VERSION << 6)) {
		return -1;
	}
    if (RTP_PT_IS_RTCP(buf[1])) {
        return rtcp_parse_packet(s, buf, len);
    }

    if (s->st) {
        int64_t received = av_gettime_relative();
        uint32_t arrival_ts = av_rescale_q(received, AV_TIME_BASE_Q,
                                           s->st->time_base);
        timestamp = AV_RB32(buf + 4);
        // Calculate the jitter immediately, before queueing the packet
        // into the reordering queue.
        rtcp_update_jitter(&s->statistics, timestamp, arrival_ts);
    }

    if ((s->seq == 0 && !s->queue) || s->queue_size <= 1) {
        /* First packet, or no reordering */
        return rtp_parse_packet_internal(s, pkt, buf, len);
    } else {
        uint16_t seq = AV_RB16(buf + 2);
        int16_t diff = seq - s->seq;
        if (diff < 0) {
            /* Packet older than the previously emitted one, drop */
            av_log(s->ic, AV_LOG_WARNING,
                   "RTP: dropping old packet received too late\n");
            return -1;
        } else if (diff <= 1) {
            /* Correct packet */
            rv = rtp_parse_packet_internal(s, pkt, buf, len);
            return rv;
        } else {
            /* Still missing some packet, enqueue this one. */
            rv = enqueue_packet(s, buf, len);
            if (rv < 0)
                return rv;
            *bufptr = NULL;
            /* Return the first enqueued packet if the queue is full,
             * even if we're missing something */
            if (s->queue_len >= s->queue_size) {
                av_log(s->ic, AV_LOG_WARNING, "jitter buffer full\n");
                return rtp_parse_queued_packet(s, pkt);
            }
            return -1;
        }
    }
}
```

rtp_parse_one_packet主要做一些分析前的工作，比如重组，确保包的处理顺序是单调递增的。如果发现这是个RTCP包，直接分析然后返回。如果是RTP包，则需要看情况是否需要进行重组，因为下一个seq可能不是收到的这个RTP包，需这时候要重组。但如果超过重组队列设置的最大大小，就马上处理数据，这时候就会丢数据，造成花屏。

在一切正常的情况下，进入rtp_parse_packet_internal这个函数。

``` c
static int rtp_parse_packet_internal(RTPDemuxContext *s, AVPacket *pkt,
                                     const uint8_t *buf, int len)
{
    unsigned int ssrc;
    int payload_type, seq, flags = 0;
    int ext, csrc;
    AVStream *st;
    uint32_t timestamp;
    int rv = 0;

    csrc         = buf[0] & 0x0f;
    ext          = buf[0] & 0x10;
    payload_type = buf[1] & 0x7f;
    if (buf[1] & 0x80)
        flags |= RTP_FLAG_MARKER;
    seq       = AV_RB16(buf + 2);
    timestamp = AV_RB32(buf + 4);
    ssrc      = AV_RB32(buf + 8);
    /* store the ssrc in the RTPDemuxContext */
    s->ssrc = ssrc;

    /* NOTE: we can handle only one payload type */
    if (s->payload_type != payload_type)
        return -1;

    st = s->st;
    // only do something with this if all the rtp checks pass...
    if (!rtp_valid_packet_in_sequence(&s->statistics, seq)) {
        av_log(s->ic, AV_LOG_ERROR,
               "RTP: PT=%02x: bad cseq %04x expected=%04x\n",
               payload_type, seq, ((s->seq + 1) & 0xffff));
        return -1;
    }

    if (buf[0] & 0x20) {
        int padding = buf[len - 1];
        if (len >= 12 + padding)
            len -= padding;
    }

    s->seq = seq;
    len   -= 12;
    buf   += 12;

    len   -= 4 * csrc;
    buf   += 4 * csrc;
    if (len < 0)
        return AVERROR_INVALIDDATA;

    /* RFC 3550 Section 5.3.1 RTP Header Extension handling */
    if (ext) {
        if (len < 4)
            return -1;
        /* calculate the header extension length (stored as number
         * of 32-bit words) */
        ext = (AV_RB16(buf + 2) + 1) << 2;

        if (len < ext)
            return -1;
        // skip past RTP header extension
        len -= ext;
        buf += ext;
    }

    if (s->handler && s->handler->parse_packet) {
        rv = s->handler->parse_packet(s->ic, s->dynamic_protocol_context,
                                      s->st, pkt, &timestamp, buf, len, seq,
                                      flags);
    } else if (st) {
        if ((rv = av_new_packet(pkt, len)) < 0)
            return rv;
        memcpy(pkt->data, buf, len);
        pkt->stream_index = st->index;
    } else {
        return AVERROR(EINVAL);
    }

    // now perform timestamp things....
    finalize_packet(s, pkt, timestamp);

    return rv;
}
```

rtp_parse_packet_internal这个函数是对RTP包真正进行分析的地方。对照着RFC 3550一个一个来看就行。关注一下那个handler->parse_packet。对于RTP H264而言，需要从SDP从提取出sps与pps。

完了后调用av_new_packet为AVPacket分配buf，然后copy未解码的数据到里面。完成。

---
未完待续。