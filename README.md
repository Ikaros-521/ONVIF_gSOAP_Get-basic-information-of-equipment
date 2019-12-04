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
