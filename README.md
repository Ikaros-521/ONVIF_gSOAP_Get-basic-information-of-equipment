## 相关配置
ONVIF官网：[http://www.onvif.org/](http://www.onvif.org/)
gSOAP安装配置：[gSOAP安装配置+使用案例参考+参考链接](https://blog.csdn.net/Ikaros_521/article/details/103232677)
操作系统：CentOS7

## 资料参考
许振坪的ONVIF专栏：[传送门](https://blog.csdn.net/benkaoya/category_6924052.html)
大佬写的 [6](https://blog.csdn.net/benkaoya/article/details/72466827)， [7](https://blog.csdn.net/benkaoya/article/details/72476120)，[8](https://blog.csdn.net/benkaoya/article/details/72476787)三篇文章（主要针对）。
设备发现参考：[ONVIF 设备发现（网络摄像头）——实例笔记](https://blog.csdn.net/Ikaros_521/article/details/103361177)

## 代码实战
完整源码下载：[GitHub](https://github.com/Ikaros-521/ONVIF_gSOAP_Get-basic-information-of-equipment)，[码云](https://gitee.com/ikaros-521/ONVIF_gSOAP_Get-basic-information-of-equipment)
如何生成ONVIF框架参考：[ONVIF协议网络摄像机（IPC）客户端程序开发（6）：使用gSOAP生成ONVIF框架代码](https://blog.csdn.net/benkaoya/article/details/72466827)
因为大佬没有在第8篇文章中给出完整代码，所以直接编译肯定过不了，再此进行补充
要做的是 将第7篇中的代码和第8篇中的代码连起来，去掉大佬没有提供的头文件和函数，改为自己的。相关修改过程已经在大佬的文章下面评论了，大家可以自行参考。

以下是我整理好后的代码：
**main.c**

```c
#include <stdio.h>
#include <stdlib.h>
#include <assert.h>
#include "soapH.h"
#include "wsaapi.h"
#include "wsdd.nsmap"
//#include "onvif_dump.h"

#define SOAP_ASSERT     assert
#define SOAP_DBGLOG     printf
#define SOAP_DBGERR     printf

#define SOAP_TO         "urn:schemas-xmlsoap-org:ws:2005:04:discovery"
#define SOAP_ACTION     "http://schemas.xmlsoap.org/ws/2005/04/discovery/Probe"

#define SOAP_MCAST_ADDR "soap.udp://239.255.255.250:3702"                       // onvif规定的组播地址

#define SOAP_ITEM       ""                                                      // 寻找的设备范围
#define SOAP_TYPES      "dn:NetworkVideoTransmitter"                            // 寻找的设备类型

#define SOAP_SOCK_TIMEOUT    (10)               // socket超时时间（单秒秒）

#define SOAP_CHECK_ERROR(result, soap, str) \
    do { \
        if (SOAP_OK != (result) || SOAP_OK != (soap)->error) { \
            soap_perror((soap), (str)); \
            if (SOAP_OK == (result)) { \
                (result) = (soap)->error; \
            } \
            goto EXIT; \
        } \
} while (0)

void soap_perror(struct soap *soap, const char *str)
{
    if (NULL == str) {
        SOAP_DBGERR("[soap] error: %d, %s, %s\n", soap->error, *soap_faultcode(soap), *soap_faultstring(soap));
    } else {
        SOAP_DBGERR("[soap] %s error: %d, %s, %s\n", str, soap->error, *soap_faultcode(soap), *soap_faultstring(soap));
    }
    return;
}

void* ONVIF_soap_malloc(struct soap *soap, unsigned int n)
{
    void *p = NULL;

    if (n > 0) {
        p = soap_malloc(soap, n);
        SOAP_ASSERT(NULL != p);
        memset(p, 0x00 ,n);
    }
    return p;
}

struct soap *ONVIF_soap_new(int timeout)
{
    struct soap *soap = NULL;                                                   // soap环境变量

    SOAP_ASSERT(NULL != (soap = soap_new()));

    soap_set_namespaces(soap, namespaces);                                      // 设置soap的namespaces
    soap->recv_timeout    = timeout;                                            // 设置超时（超过指定时间没有数据就退出）
    soap->send_timeout    = timeout;
    soap->connect_timeout = timeout;

#if defined(__linux__) || defined(__linux)                                      // 参考https://www.genivia.com/dev.html#client-c的修改：
    soap->socket_flags = MSG_NOSIGNAL;                                          // To prevent connection reset errors
#endif

    soap_set_mode(soap, SOAP_C_UTFSTRING);                                      // 设置为UTF-8编码，否则叠加中文OSD会乱码

    return soap;
}

void ONVIF_soap_delete(struct soap *soap)
{
    soap_destroy(soap);                                                         // remove deserialized class instances (C++ only)
    soap_end(soap);                                                             // Clean up deserialized data (except class instances) and temporary data
    soap_done(soap);                                                            // Reset, close communications, and remove callbacks
    soap_free(soap);                                                            // Reset and deallocate the context created with soap_new or soap_copy
}

/************************************************************************
**函数：ONVIF_init_header
**功能：初始化soap描述消息头
**参数：
        [in] soap - soap环境变量
**返回：无
**备注：
    1). 在本函数内部通过ONVIF_soap_malloc分配的内存，将在ONVIF_soap_delete中被释放
************************************************************************/
void ONVIF_init_header(struct soap *soap)
{
    struct SOAP_ENV__Header *header = NULL;

    SOAP_ASSERT(NULL != soap);

    header = (struct SOAP_ENV__Header *)ONVIF_soap_malloc(soap, sizeof(struct SOAP_ENV__Header));
    soap_default_SOAP_ENV__Header(soap, header);
    header->wsa__MessageID = (char*)soap_wsa_rand_uuid(soap);
    header->wsa__To        = (char*)ONVIF_soap_malloc(soap, strlen(SOAP_TO) + 1);
    header->wsa__Action    = (char*)ONVIF_soap_malloc(soap, strlen(SOAP_ACTION) + 1);
    strcpy(header->wsa__To, SOAP_TO);
    strcpy(header->wsa__Action, SOAP_ACTION);
    soap->header = header;

    return;
}

/************************************************************************
**函数：ONVIF_init_ProbeType
**功能：初始化探测设备的范围和类型
**参数：
        [in]  soap  - soap环境变量
        [out] probe - 填充要探测的设备范围和类型
**返回：
        0表明探测到，非0表明未探测到
**备注：
    1). 在本函数内部通过ONVIF_soap_malloc分配的内存，将在ONVIF_soap_delete中被释放
************************************************************************/
void ONVIF_init_ProbeType(struct soap *soap, struct wsdd__ProbeType *probe)
{
    struct wsdd__ScopesType *scope = NULL;                                      // 用于描述查找哪类的Web服务

    SOAP_ASSERT(NULL != soap);
    SOAP_ASSERT(NULL != probe);

    scope = (struct wsdd__ScopesType *)ONVIF_soap_malloc(soap, sizeof(struct wsdd__ScopesType));
    soap_default_wsdd__ScopesType(soap, scope);                                 // 设置寻找设备的范围
    scope->__item = (char*)ONVIF_soap_malloc(soap, strlen(SOAP_ITEM) + 1);
    strcpy(scope->__item, SOAP_ITEM);

    memset(probe, 0x00, sizeof(struct wsdd__ProbeType));
    soap_default_wsdd__ProbeType(soap, probe);
    probe->Scopes = scope;
    probe->Types  = (char*)ONVIF_soap_malloc(soap, strlen(SOAP_TYPES) + 1);     // 设置寻找设备的类型
    strcpy(probe->Types, SOAP_TYPES);

    return;
}

void ONVIF_DetectDevice(void (*cb)(char *DeviceXAddr))
{
    int i;
    int result = 0;
    unsigned int count = 0;                                                     // 搜索到的设备个数
    struct soap *soap = NULL;                                                   // soap环境变量
    struct wsdd__ProbeType      req;                                            // 用于发送Probe消息
    struct __wsdd__ProbeMatches rep;                                            // 用于接收Probe应答
    struct wsdd__ProbeMatchType *probeMatch;

    SOAP_ASSERT(NULL != (soap = ONVIF_soap_new(SOAP_SOCK_TIMEOUT)));

    ONVIF_init_header(soap);                                                    // 设置消息头描述
    ONVIF_init_ProbeType(soap, &req);                                           // 设置寻找的设备的范围和类型
    result = soap_send___wsdd__Probe(soap, SOAP_MCAST_ADDR, NULL, &req);        // 向组播地址广播Probe消息
    while (SOAP_OK == result)                                                   // 开始循环接收设备发送过来的消息
    {
        memset(&rep, 0x00, sizeof(rep));
        result = soap_recv___wsdd__ProbeMatches(soap, &rep);
        if (SOAP_OK == result) {
            if (soap->error) {
                soap_perror(soap, "ProbeMatches");
            } else {                                                            // 成功接收到设备的应答消息
                printf("__sizeProbeMatch:%d\n",rep.wsdd__ProbeMatches->__sizeProbeMatch);
                //dump__wsdd__ProbeMatches(&rep);

                if (NULL != rep.wsdd__ProbeMatches) {
                    count += rep.wsdd__ProbeMatches->__sizeProbeMatch;
                    for(i = 0; i < rep.wsdd__ProbeMatches->__sizeProbeMatch; i++) {
                        probeMatch = rep.wsdd__ProbeMatches->ProbeMatch + i;
                        if (NULL != cb) {
                            cb(probeMatch->XAddrs);                             // 使用设备服务地址执行函数回调
                        }
                    }
                }
            }
        } else if (soap->error) {
            break;
        }
    }

    SOAP_DBGLOG("\ndetect end! It has detected %d devices!\n", count);

    if (NULL != soap) {
        ONVIF_soap_delete(soap);
    }

    return ;
}


/************************************************************************
**函数：ONVIF_GetDeviceInformation
**功能：获取设备基本信息
**参数：
        [in] DeviceXAddr - 设备服务地址
**返回：
        0表明成功，非0表明失败
**备注：
************************************************************************/
int ONVIF_GetDeviceInformation(const char *DeviceXAddr)
{
    int result = 0;
    struct soap *soap = NULL;
    struct _tds__GetDeviceInformation           devinfo_req;
    struct _tds__GetDeviceInformationResponse   devinfo_resp;

    SOAP_ASSERT(NULL != DeviceXAddr);
SOAP_ASSERT(NULL != (soap = ONVIF_soap_new(SOAP_SOCK_TIMEOUT)));

    memset(&devinfo_req, 0x00, sizeof(devinfo_req));
    memset(&devinfo_resp, 0x00, sizeof(devinfo_resp));
    result = soap_call___tds__GetDeviceInformation(soap, DeviceXAddr, NULL, &devinfo_req, &devinfo_resp);
    SOAP_CHECK_ERROR(result, soap, "GetDeviceInformation");

    //dump_tds__GetDeviceInformationResponse(&devinfo_resp);
    printf("Manufacturer:%s\n",devinfo_resp.Manufacturer);
    printf("Model:%s\n",devinfo_resp.Model);
    printf("FirmwareVersion:%s\n",devinfo_resp.FirmwareVersion);
    printf("SerialNumber:%s\n",devinfo_resp.SerialNumber);
    printf("HardwareId:%s\n",devinfo_resp.HardwareId);

EXIT:

    if (NULL != soap) {
        ONVIF_soap_delete(soap);
    }
    return result;
}

void cb_discovery(char *DeviceXAddr)
{
    ONVIF_GetDeviceInformation(DeviceXAddr);
}

int main(int argc, char **argv)
{
    ONVIF_DetectDevice(cb_discovery);
    return 0;
}
```
终端输入 `gcc -o main main.c stdsoap2.c soapC.c soapClient.c wsaapi.c duration.c` 进行编译
运行效果如下：
一个设备获取不到，另一个已经成功打印相关信息。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20191204160917518.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0lrYXJvc181MjE=,size_16,color_FFFFFF,t_70)
## 补充
我提供的打印信息还不够完整。
**//dump__wsdd__ProbeMatches(&rep); 166行左右
//dump_tds__GetDeviceInformationResponse(&devinfo_resp); 217行左右** 
 **__wsdd__ProbeMatches 和 _tds__GetDeviceInformationResponse**
搜索这2个变量的类型
设计到相关结构体，可到 **soapStub.h** 中搜索
![在这里插入图片描述](https://img-blog.csdnimg.cn/20191204161153799.png)


以下提供下我搜到的一些内容，结构体__wsdd__ProbeMatches下一层套一层，我受不了了。

```c
struct __wsdd__ProbeMatches {
        /** Optional element 'wsdd:ProbeMatches' of XML schema type 'wsdd:ProbeMatchesType' */
        struct wsdd__ProbeMatchesType *wsdd__ProbeMatches;
};
```

```c
struct wsdd__ProbeMatchesType {
        /** Sequence of elements 'wsdd:ProbeMatch' of XML schema type 'wsdd:ProbeMatchType' stored in dynamic array ProbeMatch of length __sizeProbeMatch */
        int __sizeProbeMatch;
        struct wsdd__ProbeMatchType *ProbeMatch;
};
```

```c
struct wsdd__ProbeMatchType {
        /** Required element 'wsa:EndpointReference' of XML schema type 'wsa:EndpointReference' */
        struct wsa__EndpointReferenceType wsa__EndpointReference;
        /** Optional element 'wsdd:Types' of XML schema type 'xsd:QName' */
        char *Types;
        /** Optional element 'wsdd:Scopes' of XML schema type 'wsdd:ScopesType' */
        struct wsdd__ScopesType *Scopes;
        /** Optional element 'wsdd:XAddrs' of XML schema type 'wsdd:UriListType' */
        char *XAddrs;
        /** Required element 'wsdd:MetadataVersion' of XML schema type 'xsd:unsignedInt' */
        unsigned int MetadataVersion;
};
```

```c
struct _tds__GetDeviceInformationResponse {
        /** Required element 'tds:Manufacturer' of XML schema type 'xsd:string' */
        char *Manufacturer;
        /** Required element 'tds:Model' of XML schema type 'xsd:string' */
        char *Model;
        /** Required element 'tds:FirmwareVersion' of XML schema type 'xsd:string' */
        char *FirmwareVersion;
        /** Required element 'tds:SerialNumber' of XML schema type 'xsd:string' */
        char *SerialNumber;
        /** Required element 'tds:HardwareId' of XML schema type 'xsd:string' */
        char *HardwareId;
};
```
