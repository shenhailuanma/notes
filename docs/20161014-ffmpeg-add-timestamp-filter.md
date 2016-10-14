### 在ffmpeg中的drawtext中增加自定义函数用于显示时间戳

在实际开发中，有个测试需求，要求直播流显示编码时刻的时间戳。已知drawtext中有'localtime'和'gmtime'两个函数均可以叠加时间戳，不过其基于strtime()，精度只到“秒”，用作普通时间显示还行，对于时间戳，需要精度到“毫秒”。


研究了一下drawtext的源码，觉得直接增加一个新函数，实现时间戳功能倒是很方便。


#### 1. 代码修改

具体的修改如下：

``` c
// 在functions[]中增加"timestamp"函数的定义
static const struct drawtext_function {
    const char *name;
    unsigned argc_min, argc_max;
    int tag;                            /**< opaque argument to func */
    int (*func)(AVFilterContext *, AVBPrint *, char *, unsigned, char **, int);
} functions[] = {
    { "expr",      1, 1, 0,   func_eval_expr },
    { "e",         1, 1, 0,   func_eval_expr },
    { "expr_int_format", 2, 3, 0, func_eval_expr_int_format },
    { "eif",       2, 3, 0,   func_eval_expr_int_format },
    { "pict_type", 0, 0, 0,   func_pict_type },
    { "pts",       0, 3, 0,   func_pts      },
    { "gmtime",    0, 1, 'G', func_strftime },
    { "localtime", 0, 1, 'L', func_strftime },
    { "frame_num", 0, 0, 0,   func_frame_num },
    { "n",         0, 0, 0,   func_frame_num },
    { "metadata",  1, 1, 0,   func_metadata },
    { "timestamp",  0, 0, 0,  func_timestamp },
};

// "timestamp"函数的定义
static int func_timestamp(AVFilterContext *ctx, AVBPrint *bp,char *fct, unsigned argc, char **argv, int tag)
{

    int64_t ms = av_gettime()/1000;
    time_t  s = (time_t)(ms/1000);

    struct tm tm;

    localtime_r(&s, &tm);
    av_bprint_strftime(bp, "%H:%M:%S", &tm);
    av_bprintf(bp, ".%03d", (int)(ms % 1000));

    return 0;
}

```

#### 2. 编译与测试

在drawtext中增加了新的函数"timestamp"，使用方法如下所示，其它参数设置方法详见官方文档（http://ffmpeg.org/ffmpeg-all.html#drawtext-1）。

时间戳显示在中间：
./ffmpeg -re -i /home/lian.mp4 -s 1280x720 -r 25 -vf drawtext="fontfile=FreeSans.ttf:fontcolor=white:fontsize=30:x=(w-tw)/2:y=(h-th)/2:text='%{timestamp}'" -t 30  -f mp4 -y /tmp/draw.mp4

![中间](https://raw.githubusercontent.com/shenhailuanma/notes/master/images/20161014-ffmpeg-add-timestamp-filter-01.png)

时间戳显示在左上角：
./ffmpeg -re -i /home/lian.mp4 -s 1280x720 -r 25 -vf drawtext="fontfile=FreeSans.ttf:fontcolor=white:fontsize=10:x=0:y=0:text='%{timestamp}'" -t 30  -f mp4 -y /tmp/draw.mp4

![左上](https://raw.githubusercontent.com/shenhailuanma/notes/master/images/20161014-ffmpeg-add-timestamp-filter-02.png)

时间戳显示在右上角：
./ffmpeg -re -i /home/lian.mp4 -s 1280x720 -r 25 -vf drawtext="fontfile=FreeSans.ttf:fontcolor=white:fontsize=10:x=(w-tw):y=th:text='%{timestamp}'" -t 30  -f mp4 -y /tmp/draw.mp4

![右上](https://raw.githubusercontent.com/shenhailuanma/notes/master/images/20161014-ffmpeg-add-timestamp-filter-03.png)