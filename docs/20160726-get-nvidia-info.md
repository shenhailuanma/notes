## 如何查看Nvidia的GPU运行状态

　　在使用nvidia的GPU进行运算的时候，通常会有需要了解GPU运行状态需求。在下面的文章中，将会介绍我在实际使用中到的方法。
　　


### 1.使用"**nvidia-smi**"查看GPU状态信息
　　通常安装好nvidia驱动程序后会自动安装好nvidia-smi，使用nvidia-smi是最简单直接的查看GPU状态信息的方法。


直接运行"nvidia-smi"，就可以查看当前的运行状态。可以查看到GPU的各个核心的工作状态、温度、内存信息、及使用GPU的进程信息等。
```bash
+-----------------------------------------------------------------------------+
| NVIDIA-SMI 367.27                 Driver Version: 367.27                    |
|-------------------------------+----------------------+----------------------+
| GPU  Name        Persistence-M| Bus-Id        Disp.A | Volatile Uncorr. ECC |
| Fan  Temp  Perf  Pwr:Usage/Cap|         Memory-Usage | GPU-Util  Compute M. |
|===============================+======================+======================|
|   0  GRID K1             Off  | 0000:03:00.0     Off |                  N/A |
| N/A   63C    P0    18W /  31W |    545MiB /  4036MiB |     28%      Default |
+-------------------------------+----------------------+----------------------+
|   1  GRID K1             Off  | 0000:04:00.0     Off |                  N/A |
| N/A   58C    P0    14W /  31W |    195MiB /  4036MiB |      6%      Default |
+-------------------------------+----------------------+----------------------+
|   2  GRID K1             Off  | 0000:05:00.0     Off |                  N/A |
| N/A   40C    P0    16W /  31W |     99MiB /  4036MiB |     18%      Default |
+-------------------------------+----------------------+----------------------+
|   3  GRID K1             Off  | 0000:06:00.0     Off |                  N/A |
| N/A   60C    P0    16W /  31W |    200MiB /  4036MiB |     30%      Default |
+-------------------------------+----------------------+----------------------+
                                                                               
+-----------------------------------------------------------------------------+
| Processes:                                                       GPU Memory |
|  GPU       PID  Type  Process name                               Usage      |
|=============================================================================|
|    0      9133    C   ...e/ffmpeg_nvenc_decode/_release/bin/ffmpeg    48MiB |
|    0      9136    C   ...e/ffmpeg_nvenc_decode/_release/bin/ffmpeg    48MiB |
|    1      9132    C   ...e/ffmpeg_nvenc_decode/_release/bin/ffmpeg    38MiB |
|    1      9133    C   ...e/ffmpeg_nvenc_decode/_release/bin/ffmpeg    38MiB |
|    1      9134    C   ...e/ffmpeg_nvenc_decode/_release/bin/ffmpeg    38MiB |
|    1      9135    C   ...e/ffmpeg_nvenc_decode/_release/bin/ffmpeg    38MiB |
|    1      9136    C   ...e/ffmpeg_nvenc_decode/_release/bin/ffmpeg    38MiB |
|    2      9132    C   ...e/ffmpeg_nvenc_decode/_release/bin/ffmpeg    48MiB |
|    2      9134    C   ...e/ffmpeg_nvenc_decode/_release/bin/ffmpeg    48MiB |
|    3      9135    C   ...e/ffmpeg_nvenc_decode/_release/bin/ffmpeg    48MiB |
+-----------------------------------------------------------------------------+

```

如果想了解更多信息，运行"nvidia-smi -a"或"nvidia-smi -q"，可以查看到详尽的GPU信息。比如我个人比较关心使用GPU进行视频编解码的时候的GPU使用情况，就可以通过"Utilization"字段下的信息查看到各种使用率的情况。

```bash
    Utilization
        Gpu                         : 5 %
        Memory                      : 6 %
        Encoder                     : 40 %
        Decoder                     : 0 %

```

### 2.使用"**NVML API**"获取GPU状态信息

使用"nvidia-smi"虽然简单方便，但因为它是个应用程序，不是很方便在代码中进行使用。所以就尝试找找官方是否提供了获取GPU信息的方法，果然"NVML API"就是所需。在之后的时间里，通过参考API文档，比较快速的整理出我希望获取到的GPU运行状态信息。

"NVML API"主要需要库libnvidia-ml，我用它主要是获取GPU的基本信息和各种使用率。具体使用了哪些API这里不做赘述，其实直接看API文档和我的测试源码更容易理解。

API文档：http://docs.nvidia.com/deploy/pdf/NVML_API_Reference_Guide.pdf

最后，简单暴力的附上代码:https://github.com/shenhailuanma/selfTestCode/blob/master/getNvidiaInfo/vn_info.c
```
#include <stdio.h>
#include <stdint.h>
#include <string.h>
#include <stdlib.h>
#include <unistd.h>
#include <time.h>


#if defined(_WIN32)
#include <windows.h>
#else
#include <dlfcn.h>
#endif

#if defined(_WIN32)
#define CUDAAPI __stdcall
#else
#define CUDAAPI
#endif

#if defined(_WIN32)
#define LOAD_FUNC(l, s) GetProcAddress(l, s)
#define DL_CLOSE_FUNC(l) FreeLibrary(l)
#else
#define LOAD_FUNC(l, s) dlsym(l, s)
#define DL_CLOSE_FUNC(l) dlclose(l)
#endif

/** 
 * Return values for NVML API calls. 
 */
typedef enum nvmlReturn_enum 
{
    NVML_SUCCESS = 0,                   //!< The operation was successful
    NVML_ERROR_UNINITIALIZED = 1,       //!< NVML was not first initialized with nvmlInit()
    NVML_ERROR_INVALID_ARGUMENT = 2,    //!< A supplied argument is invalid
    NVML_ERROR_NOT_SUPPORTED = 3,       //!< The requested operation is not available on target device
    NVML_ERROR_NO_PERMISSION = 4,       //!< The current user does not have permission for operation
    NVML_ERROR_ALREADY_INITIALIZED = 5, //!< Deprecated: Multiple initializations are now allowed through ref counting
    NVML_ERROR_NOT_FOUND = 6,           //!< A query to find an object was unsuccessful
    NVML_ERROR_INSUFFICIENT_SIZE = 7,   //!< An input argument is not large enough
    NVML_ERROR_INSUFFICIENT_POWER = 8,  //!< A device's external power cables are not properly attached
    NVML_ERROR_DRIVER_NOT_LOADED = 9,   //!< NVIDIA driver is not loaded
    NVML_ERROR_TIMEOUT = 10,            //!< User provided timeout passed
    NVML_ERROR_UNKNOWN = 999            //!< An internal driver error occurred
} nvmlReturn_t;

typedef void * nvmlDevice_t;

/* Memory allocation information for a device. */
typedef struct nvmlMemory_st 
{
    unsigned long long total;        //!< Total installed FB memory (in bytes)
    unsigned long long free;         //!< Unallocated FB memory (in bytes)
    unsigned long long used;         //!< Allocated FB memory (in bytes). Note that the driver/GPU always sets aside a small amount of memory for bookkeeping
} nvmlMemory_t;


/* Information about running compute processes on the GPU */
typedef struct nvmlProcessInfo_st
{
    unsigned int pid;                 //!< Process ID
    unsigned long long usedGpuMemory; //!< Amount of used GPU memory in bytes.
                                      //!< Under WDDM, \ref NVML_VALUE_NOT_AVAILABLE is always reported
                                      //!< because Windows KMD manages all the memory and not the NVIDIA driver
} nvmlProcessInfo_t;

/* Utilization information for a device. */
typedef struct nvmlUtilization_st 
{
    unsigned int gpu;                //!< Percent of time over the past second during which one or more kernels was executing on the GPU
    unsigned int memory;             //!< Percent of time over the past second during which global (device) memory was being read or written
} nvmlUtilization_t;

typedef nvmlReturn_t(CUDAAPI *NVMLINIT)(void);  // nvmlInit
typedef nvmlReturn_t(CUDAAPI *NVMLSHUTDOWN)(void);  // nvmlShutdown 
typedef nvmlReturn_t(CUDAAPI *NVMLDEVICEGETCOUNT)(unsigned int *deviceCount); // nvmlDeviceGetCount
typedef nvmlReturn_t(CUDAAPI *NVMLDEVICEGETHANDLEBYINDEX)(unsigned int index, nvmlDevice_t *device); // nvmlDeviceGetHandleByIndex
typedef nvmlReturn_t(CUDAAPI *NVMLDEVICEGETDECODERUTILIZATION)(nvmlDevice_t device, unsigned int *utilization,unsigned int *samplingPeriodUs); // nvmlDeviceGetDecoderUtilization
typedef nvmlReturn_t(CUDAAPI *NVMLDEVICEGETENCODERUTILIZATION)(nvmlDevice_t device, unsigned int *utilization,unsigned int *samplingPeriodUs); // nvmlDeviceGetEncoderUtilization 
typedef nvmlReturn_t(CUDAAPI *NVMLDEVICEGETMEMORYINFO)(nvmlDevice_t device, nvmlMemory_t *memory); // nvmlDeviceGetMemoryInfo
typedef nvmlReturn_t(CUDAAPI *NVMLDEVICEGETRUNNINGPROCESSES)(nvmlDevice_t device, unsigned int *infoCount,nvmlProcessInfo_t *infos);// nvmlDeviceGetComputeRunningProcesses
typedef nvmlReturn_t(CUDAAPI *NVMLDEVICEGETPPROCESSNAME)(unsigned int pid, char *name, unsigned int length); // nvmlSystemGetProcessName
typedef nvmlReturn_t(CUDAAPI *NVMLDEVICEGETUTILIZATIONRATES)(nvmlDevice_t device, nvmlUtilization_t *utilization); // nvmlDeviceGetUtilizationRates
typedef nvmlReturn_t(CUDAAPI *NVMLDEVICEGETTEMPERATURE)(nvmlDevice_t device, int sensorType, unsigned int *temp); // nvmlDeviceGetTemperature


typedef struct nvGpuUnitInfo_st
{
    unsigned int decoder_utilization;
    unsigned int encoder_utilization;
    unsigned int gpu_utilization;
    unsigned int memory_utilization;
    unsigned int temperature;

    unsigned long long memory_total;
    unsigned long long memory_free;
    unsigned long long memory_used;

}nvGpuUnitInfo_t;

#define GPU_MAX_SIZE    64

typedef struct nvGpuInfo_st
{
    unsigned int device_count;
    nvGpuUnitInfo_t devices[GPU_MAX_SIZE];

}nvGpuInfo_t;


#define RETURN_SUCCESS     0
#define RETURN_ERROR_LOAD_LIB       (-1)
#define RETURN_ERROR_LOAD_FUNC      (-2)
#define RETURN_ERROR_LIB_FUNC       (-3)
#define RETURN_ERROR_NULL_POINTER   (-4)


#define CHECK_LOAD_NVML_FUNC(t, f, s) \
do { \
    (f) = (t)LOAD_FUNC(nvml_lib, s); \
    if (!(f)) { \
        printf("Failed loading %s from NVML library\n", s); \
        retCode = RETURN_ERROR_LOAD_FUNC; \
        goto gpu_fail; \
    } \
} while (0)

static int check_nvml_error(int err, const char *func)
{
    if (err != NVML_SUCCESS) {
        printf(" %s - failed with error code:%d\n", func, err);
        return 0;
    }
    return 1;
}
#define check_nvml_errors(f) \
do{ \
    if (!check_nvml_error(f, #f)) { \
        retCode = RETURN_ERROR_LIB_FUNC; \
        goto gpu_fail;\
    }\
}while(0)



static int get_gpu_info(nvGpuInfo_t *infos)
{

    if(infos == NULL){
        return RETURN_ERROR_NULL_POINTER;
    }

    int retCode = RETURN_SUCCESS;
    void* nvml_lib;
    NVMLINIT                    nvml_init;
    NVMLSHUTDOWN                nvml_shutdown;
    NVMLDEVICEGETCOUNT          nvml_device_get_count;
    NVMLDEVICEGETHANDLEBYINDEX  nvml_device_get_handle_by_index;
    NVMLDEVICEGETDECODERUTILIZATION     nvml_device_get_decoder_utilization;
    NVMLDEVICEGETENCODERUTILIZATION     nvml_device_get_encoder_utilization;
    NVMLDEVICEGETMEMORYINFO     nvml_device_get_memory_info;
    NVMLDEVICEGETRUNNINGPROCESSES       nvml_device_get_running_processes;
    NVMLDEVICEGETPPROCESSNAME   nvml_device_get_process_name;
    NVMLDEVICEGETUTILIZATIONRATES       nvml_device_get_utilization_rates;
    NVMLDEVICEGETTEMPERATURE    nvml_device_get_temperature;

    nvmlDevice_t device_handel;


    unsigned int utilization_value = 0;
    unsigned int utilization_sample = 0;
    int best_gpu = 0;
    unsigned int decoder_used = 100;

    // open the libnvidia-ml.so
    nvml_lib = NULL;
    #if defined(_WIN32)
        if (sizeof(void*) == 8) {
            nvml_lib = LoadLibrary(TEXT("nvidia-ml.dll"));
        } else {
            nvml_lib = LoadLibrary(TEXT("nvidia-ml.dll"));
        }
    #else
        nvml_lib = dlopen("libnvidia-ml.so", RTLD_LAZY);
    #endif

    if(nvml_lib == NULL){
        return RETURN_ERROR_LOAD_LIB;
    }


    CHECK_LOAD_NVML_FUNC(NVMLINIT, nvml_init, "nvmlInit");
    CHECK_LOAD_NVML_FUNC(NVMLSHUTDOWN, nvml_shutdown, "nvmlShutdown");

    CHECK_LOAD_NVML_FUNC(NVMLDEVICEGETCOUNT, nvml_device_get_count, "nvmlDeviceGetCount");
    CHECK_LOAD_NVML_FUNC(NVMLDEVICEGETHANDLEBYINDEX, nvml_device_get_handle_by_index, "nvmlDeviceGetHandleByIndex");
    CHECK_LOAD_NVML_FUNC(NVMLDEVICEGETDECODERUTILIZATION, nvml_device_get_decoder_utilization, "nvmlDeviceGetDecoderUtilization");
    CHECK_LOAD_NVML_FUNC(NVMLDEVICEGETENCODERUTILIZATION, nvml_device_get_encoder_utilization, "nvmlDeviceGetEncoderUtilization");
    CHECK_LOAD_NVML_FUNC(NVMLDEVICEGETMEMORYINFO, nvml_device_get_memory_info, "nvmlDeviceGetMemoryInfo");
    CHECK_LOAD_NVML_FUNC(NVMLDEVICEGETRUNNINGPROCESSES, nvml_device_get_running_processes, "nvmlDeviceGetComputeRunningProcesses");
    CHECK_LOAD_NVML_FUNC(NVMLDEVICEGETPPROCESSNAME, nvml_device_get_process_name, "nvmlSystemGetProcessName");
    CHECK_LOAD_NVML_FUNC(NVMLDEVICEGETUTILIZATIONRATES, nvml_device_get_utilization_rates, "nvmlDeviceGetUtilizationRates");
    CHECK_LOAD_NVML_FUNC(NVMLDEVICEGETTEMPERATURE, nvml_device_get_temperature, "nvmlDeviceGetTemperature");


    // get gpu info
    check_nvml_errors(nvml_init());
    unsigned int device_count = 0;

    check_nvml_errors(nvml_device_get_count(&device_count));
    infos->device_count = device_count;


    nvmlMemory_t memory_info;
    nvmlUtilization_t gpu_utilization;


    int i = 0;
    for(i = 0; i < device_count; i++){
        check_nvml_errors(nvml_device_get_handle_by_index(i, &device_handel));
        check_nvml_errors(nvml_device_get_decoder_utilization(device_handel, &infos->devices[i].decoder_utilization, &utilization_sample));
        check_nvml_errors(nvml_device_get_encoder_utilization(device_handel, &infos->devices[i].encoder_utilization, &utilization_sample));

        check_nvml_errors(nvml_device_get_memory_info(device_handel, &memory_info));
        infos->devices[i].memory_total = memory_info.total;
        infos->devices[i].memory_free  = memory_info.free;
        infos->devices[i].memory_used  = memory_info.used;

        check_nvml_errors(nvml_device_get_utilization_rates(device_handel, &gpu_utilization));
        infos->devices[i].gpu_utilization = gpu_utilization.gpu;
        infos->devices[i].memory_utilization = gpu_utilization.memory;

        check_nvml_errors(nvml_device_get_temperature(device_handel, 0, &infos->devices[i].temperature));

    }

gpu_fail:

    nvml_shutdown();

    return retCode;
}


static void print_gpu_info(nvGpuInfo_t * infos)
{
    printf("device count:%u\n", infos->device_count);

    int i = 0;

    for(i = 0; i < infos->device_count; i++){
        printf("GPU:%d\t, Utilization:[decoder:%u, encoder:%u, gpu:%u, memory:%u], Temperature:%uC, Memory:[total:%llu, free:%llu, used:%llu]\n ", 
            i, infos->devices[i].decoder_utilization, infos->devices[i].encoder_utilization, infos->devices[i].gpu_utilization, infos->devices[i].memory_utilization,
            infos->devices[i].temperature, infos->devices[i].memory_total, infos->devices[i].memory_free, infos->devices[i].memory_used);
    }
}

int main(void)
{


    nvGpuInfo_t gpu_buf;

    int ret = get_gpu_info(&gpu_buf);

    if(!ret)
        print_gpu_info(&gpu_buf);

    return ret;
}

```