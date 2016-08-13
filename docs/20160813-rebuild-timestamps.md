#### 一个时间戳重建机制

##### 场景介绍
在实际的音视频时间戳处理中，有时候会遇到时间戳异常的问题，比如：时间戳跨度太大，或时间戳非单调递增。

因为对于时间戳的问题，不同的场景有不同的解决方法，所以，首先先介绍一下本文的场景。本文中所设计的时间戳重建机制主要是为了能够解决输入为连续不断的一些帧（音频或者视频），这些帧直接存在时间戳跨度太大，或时间戳非单调递增问题，期望经过时间戳重建，输出连续的、单调递增、间隔正确的序列。可以参考的实际场景类似多个视频文件拼接成一个视频文件。


##### 解决方案

** 思路 **

其实这个场景源是多个不同的时间戳体系，目标是输出一个时间戳体系。对于这种“时间戳跨度太大，或时间戳非单调递增”的场景，我们需要对关注“时间戳间隔”这个值。在正常的媒体流中，时间戳间隔基本保持在一个很小的范围内，所以若超过正常范围，则认为时间戳发生异常。

>例如：视频时间戳间隔在帧率为25帧每秒的情况下是1000ms/25=40ms，在30帧每秒的情况下是1000ms/30=33.333ms（不可整除），因为时间戳在实际代码操作中往往是整数，所以时间戳间隔在33ms上下来回抖动。

![流程图](https://raw.githubusercontent.com/shenhailuanma/notes/master/images/20160813-rebuild-timestamps-01.png)

** 代码实现 **

下面的代码片段是我在实际使用中实现的时间戳重建机制。利用它可以将接收到的不稳定的时间戳重建成单调递增且同步。

```c
#define SMEM_MAX_STREAM     64
#define SMEM_TIME_BASE      1000000
#define SMEM_TIME_BASE_Q    (AVRational){1, SMEM_TIME_BASE}

#define SMEM_NUM_IN_RANGE(n,min,max)  ((n) > (max) ? (max) : ((n) < (min) ? (min) : (n)))
#define SMEM_NUM_IF_OUT_RANGE(n,min,max)  ((n) > (max) ? 1 : ((n) < (min) ? 1 : 0))

typedef struct smem_dec_ctx {
    const AVClass *class;

    // for timestamp
    int       base_stream_start;
    int64_t   out_index; // the stream index of the out timestamp base used
    int64_t   last_out_ts[SMEM_MAX_STREAM]; // the last out ts for each stream,timebase={1, 1000000}
    int64_t   last_in_ts[SMEM_MAX_STREAM]; // the last in ts for each stream
    int64_t   first_in_ts[SMEM_MAX_STREAM];
    int should_duration[SMEM_MAX_STREAM];
    AVRational in_timebase[SMEM_MAX_STREAM];

}smem_dec_ctx;

static int rebuild_timestamp(AVFormatContext *avctx, struct memory_info * m_info, int64_t * out_pts, int64_t * out_dts)
{
    struct smem_dec_ctx * ctx = avctx->priv_data;

    av_log(avctx, AV_LOG_VERBOSE, "[rebuild_timestamp] before rebuild pts: %lld, dts: %lld\n", m_info->pts, m_info->dts);

    // 将时间戳转换为指定时间基础
    int64_t pts = av_rescale_q(m_info->pts, ctx->in_timebase[m_info->index], SMEM_TIME_BASE_Q);
    int64_t dts = av_rescale_q(m_info->dts, ctx->in_timebase[m_info->index], SMEM_TIME_BASE_Q);

    int64_t dt; // 记录差值

    if(m_info->index == 0){
        // the first number stream is the timestamp rebuild base stream 

        if(SMEM_NUM_IF_OUT_RANGE(dts - ctx->last_in_ts[m_info->index], 0, 5*ctx->should_duration[m_info->index])){
            ctx->first_in_ts[m_info->index] = dts;

            av_log(avctx, AV_LOG_WARNING, "[rebuild_timestamp] index:%d, last dts: %lld, the new dts: %lld is out of the range\n", m_info->index, ctx->last_in_ts[m_info->index], dts);
            *out_dts = ctx->last_out_ts[m_info->index] + ctx->should_duration[m_info->index];
            

        }else{
            *out_dts = ctx->last_out_ts[m_info->index] + (dts - ctx->last_in_ts[m_info->index]);
        }

        *out_pts = *out_dts + (pts - dts) + 2*ctx->should_duration[m_info->index];

        ctx->base_stream_start = 1;

    }else{
        if(ctx->base_stream_start != 1){
            av_log(avctx, AV_LOG_WARNING, "[rebuild_timestamp] index:%d, base stream not start, skip.\n", m_info->index);
            return -1;
        }

        // other number streams check the timestamp by the base stream
        if(SMEM_NUM_IF_OUT_RANGE(dts - ctx->last_in_ts[m_info->index], 0, 5*ctx->should_duration[m_info->index])){
            ctx->first_in_ts[m_info->index] = dts;

            av_log(avctx, AV_LOG_WARNING, "[rebuild_timestamp] index:%d, last dts: %lld, the new dts: %lld is out of the range\n", m_info->index, ctx->last_in_ts[m_info->index], dts);
            

            // the others streams should after base stream
            if(ctx->first_in_ts[m_info->index] < ctx->first_in_ts[0]){
                av_log(avctx, AV_LOG_WARNING, "[rebuild_timestamp] index:%d, the local stream time(%lld) < base stream time(%lld), skip.\n"
                    , m_info->index, ctx->first_in_ts[m_info->index], ctx->first_in_ts[0]);

                return -1;
            }

            dt = ctx->first_in_ts[m_info->index] - ctx->first_in_ts[0]; // the dt of the new streams

            // the streams out ts should same as dt
            *out_dts = ctx->last_out_ts[0] + SMEM_NUM_IN_RANGE(dt, ctx->should_duration[m_info->index], 10*ctx->should_duration[m_info->index]);

            if(*out_dts < ctx->last_out_ts[m_info->index]){
                av_log(avctx, AV_LOG_WARNING, "[rebuild_timestamp] index:%d, the local out time(%lld) < last out time(%lld), skip.\n",
                    m_info->index,*out_dts, ctx->last_out_ts[m_info->index]);
                return -1;
            }

        }else{


            *out_dts = ctx->last_out_ts[m_info->index] + (dts - ctx->last_in_ts[m_info->index]);
        }

        *out_pts = *out_dts + (pts - dts) + 2*ctx->should_duration[m_info->index];

    }

    ctx->last_in_ts[m_info->index] = dts;
    ctx->last_out_ts[m_info->index] = *out_dts;


    av_log(avctx, AV_LOG_VERBOSE, "[rebuild_timestamp] after rebuild pts: %lld, dts: %lld\n", *out_pts, *out_dts);


    return 0;
}
```

