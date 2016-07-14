# Nvidia硬解码总结

#### 1.前言

　　本文的主要目的是对近期进行的nvidia硬件解码工作的记录和总结。至于为什么研究nvidia硬件解码的具体内容，其实主要是为了在项目中能够利用nvidia的硬件解码和编码能力，提高单机的编解码并行能力。截止当前，nvidia的硬件编码官方提供了nvenc的方法，且在ffmpeg中已经增加了对nvenc的编码库。对于硬件解码，官方提供了基于cuda的解码方法，但是ffmpeg中还没有相应的解码库。所以，我的目的就是调研一下这个硬解方案，并将其自定义增加到ffmpeg中。

　　官方提供的资料比较少，只包括一页的[视频解码器介绍](http://docs.nvidia.com/cuda/video-decoder/index.html)和[示例代码](http://developer.download.nvidia.com/assets/cuda/files/nvidia_video_sdk_6.0.1.zip?autho=1466495108_e257516a4c8e5ec4f6992e5b812a8b7e&file=nvidia_video_sdk_6.0.1.zip)。

　　吐槽一下：官网那个一页的介绍参考量真不大，主要还是参考例程代码。


#### 2.例程介绍

　　官网提供的例程代码解压后如下图所示，因为是调用解码，所以主要参考了"NvDecodeD3D9"和"NvTranscoder"的代码。

　　总的来说，nvidia提供了source, parser, decoder三个基本模块。其中source是用来解析视频文件(例如：纯h.264文件)，parser是用来解析视频并得到一帧帧的数据，decoder就是解码了。


```sequence
title: 流程图
source-->parser: h.264数据
parser-->pfnSequenceCallback: 视频格式参数
parser-->pfnDecodePicture: 视频帧内部封装
pfnDecodePicture-->decoder: 视频帧内部封装
parser-->pfnDisplayPicture: YUV数据内部封装数据

```

　　这三个模块相辅相成，其主要操作流程如上图所示。source模块输出h264数据，parser解析这些h264数据，并通过3个重要的回调函数（pfnSequenceCallback， pfnDecodePicture， pfnDisplayPicture）完成解码及输出功能。其中，pfnSequenceCallback是parser解析到序列及图像参数信息时的回调函数，其传入的参数是parser解析好的视频参数，可以用于初始化解码器或重置解码器。pfnDecodePicture是parser解析到视频编码数据后的回调函数，其传入的参数parser处理好待解码的视频编码数据，需要在该函数中调用decoder的接口进行解码操作。pfnDisplayPicture是parser对解码后的数据处理的回调函数，可以在该回调中对已解码的数据进行获取（从显存到系统内存）并处理。


#### 3.主要接口说明

　　**cuvidCreateVideoSource** : 该接口的作用是创建source，主要参数是设置视频文件路径和回调函数。source会去解析指定视频文件，并通过回调函数实现对视频数据的自定义处理。源码中在视频数据回调函数中，调用了cuvidParseVideoData，即向parser中传递数据。

```c
    //init video source
    CUVIDSOURCEPARAMS oVideoSourceParameters;
    memset(&oVideoSourceParameters, 0, sizeof(CUVIDSOURCEPARAMS));
    oVideoSourceParameters.pUserData = this;
    oVideoSourceParameters.pfnVideoDataHandler = HandleVideoData;
    oVideoSourceParameters.pfnAudioDataHandler = NULL;

    oResult = cuvidCreateVideoSource(&m_videoSource, videoPath, &oVideoSourceParameters);
    if (oResult != CUDA_SUCCESS) {
        fprintf(stderr, "cuvidCreateVideoSource failed\n");
        fprintf(stderr, "Please check if the path exists, or the video is a valid H264 file\n");
        exit(-1);
    }
```


　　**cuvidCreateVideoParser** : 该接口是用来创建video parser，主要参数是设置三个回调函数，实现对解析出来的数据的处理。


```c
    //init video parser
    CUVIDPARSERPARAMS oVideoParserParameters;
    memset(&oVideoParserParameters, 0, sizeof(CUVIDPARSERPARAMS));
    oVideoParserParameters.CodecType = oVideoDecodeCreateInfo.CodecType;
    oVideoParserParameters.ulMaxNumDecodeSurfaces = oVideoDecodeCreateInfo.ulNumDecodeSurfaces;
    oVideoParserParameters.ulMaxDisplayDelay = 1;
    oVideoParserParameters.pUserData = this;
    oVideoParserParameters.pfnSequenceCallback = HandleVideoSequence;
    oVideoParserParameters.pfnDecodePicture = HandlePictureDecode;
    oVideoParserParameters.pfnDisplayPicture = HandlePictureDisplay;

    oResult = cuvidCreateVideoParser(&m_videoParser, &oVideoParserParameters);
    if (oResult != CUDA_SUCCESS) {
        fprintf(stderr, "cuvidCreateVideoParser failed, error code: %d\n", oResult);
        exit(-1);
    }
```

　　**cuvidParseVideoData** : 该接口是用来向parser塞数据，通过不断地塞h.264数据，parser会通过回调接口对解析出来的数据进行处理。在例程中，cuvidParseVideoData是在source的pfnVideoDataHandler回调中被使用的，即source获取到视频数据，就将其传递给parser。

```c
    // the callback of source pfnVideoDataHandler
    static int CUDAAPI HandleVideoData(void* pUserData, CUVIDSOURCEDATAPACKET* pPacket)
    {
        assert(pUserData);
        CudaDecoder* pDecoder = (CudaDecoder*)pUserData;

        CUresult oResult = cuvidParseVideoData(pDecoder->m_videoParser, pPacket);
        if(oResult != CUDA_SUCCESS) {
            printf("error!\n");
        }

        return 1;
    }
```

　　**cuvidCreateDecoder** : 该接口是用来创建decoder，通过设置一些解码参数，会返回一个decoder的句柄。这个句柄会在之后的解码接口中被使用。该接口的具体使用方法在例程中有详细的参数设置，这里就繁琐地描述了。


　　**cuvidDecodePicture** : 该接口就是向解码器传递待解码的数据。需要说明一下，该接口是异步解码，不能通过该接口得到解码后的视频数据，它只是向解码器传数据而已。解码后的数据，是通过parser的pfnDisplayPicture回调得到。




#### 4.技术点说明

##### 


#### 5.向ffmpeg中增加解码器