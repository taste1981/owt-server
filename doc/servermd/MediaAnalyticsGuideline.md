                                       Open WebRTC Toolkit 之视频分析简介

本文介绍Open WebRTC Toolkit中的视频分析功能的使用，以及如何使用Intel
Distribution of OpenVINO实现自定义的视频分析。

OWT视频分析架构
===============

OWT 视频分析的系统架构如下：
![Analytics Arch](https://github.com/taste1981/owt-server/blob/master/doc/servermd/analytics_diagram.jpg)
OWT Server中允许用户通过Analytics REST
API对某个Room中的任意一个流进行视频分析，包括从WebRTC以及SIP节点导入的流，以及从混流节点等其它节点导入的流。一个成功的视频分析请求将在同一个Room内生成一个新的流。

分析产生的流与Room中的其它流一样，可被订阅，分析，录制或者混流。

进程模型
--------

每一个新的视频分析请求都将在系统中创建一个新的视频分析进程，包含一个完整的媒体分析管道。这个分析管道包含解码器，前处理器，视频分析器，编码器4个部分。压缩过的码流通过OWT MCU Server的Internal In/Out模块出入当前的媒体分析管道。

当前的实现中，每个视频分析进程的视频分析器仅允许加载一个视频分析插件。具体加载的插件由发起的分析REST
请求中包含的算法ID指定。

编译与安装
==========

系统要求
--------

硬件要求： Intel(R) 第六代–第八代Core(TM)平台,含Intel集成显卡。

软件要求： CentOS 7.4/7.6 或者Ubuntu 16.04/18.04

安装步骤
--------

### 安装OpenVINO

按照https://docs.openvinotoolkit.org/latest/_docs_install_guides_installing_openvino_linux.html中提供的步骤安装Intel OpenVINO 2018 R5（Open WebRTC Toolkit当前默认只支持该版本）。请确认安装步骤中包含了OpenCV ，且OpenVINO提供的示例可以正常工作。

-   使用GPU
    Plugin的推导程序要求系统中安装OpenCL驱动程序。如果那你的系统中尚未安装OpenCL驱动，那么需要在OpenVINO安装之后，到/opt/intel/computer_vision_sdk/install_dependencies/目录下，执行
````
sudo -E ./install_NEO_OCL_driver.sh
````

-   如果你的系统中已经安装过其它版本的OpenVINO，请确保先卸载其它版本的OpenVINO。卸载脚本通常在````/opt/intel/computer_vision_sdk/openvino_toolkit_uninstall/````目录下。

### 编译Open WebRTC Toolkit 服务器

从https://github.com/open-webrtc-toolkit/owt-server下载代码并安装页面上的说明编译OWT
MCU服务器。

-   首先安装编译所需的依赖：

````scripts/installDeps.sh````


-   然后编译所有的模块（包括analytics模块）：

````scripts/build.js -t all --check````


-   最后将所有的模块打包：

````scripts/pack.js -t all --install-module --sample-path \${webrtc-javascript-sdk-sample-conference-dist}````



注意这里的sample-path指向你编译好OWT Javascript
SDK的owt-client-javascript/dist/sample/conference目录。更多关于OWT Javascript
SDK编译相关的信息请查阅<https://github.com/open-webrtc-toolkit/owt-client-javascript>的说明文档。

另外，MCU当前依赖于node.js版本8。 请确保MCU运行环境为node.js 8.
关于node的安装，可参考<https://nodesource.com/blog/installing-node-js-tutorial-using-nvm-on-mac-os-x-and-ubuntu/>。

### 编译和部署Open WebRTC Toolkit的内置插件

OWT MCU
server在编译和打包完毕之后，其支持的内置视频分析插件的源代码位于dist/analytics_agent/plugins/samples/目录。打包之前的内置视频分析插件的源代码位于<https://github.com/open-webrtc-toolkit/owt-server/tree/master/source/agent/plugins>目录下。

-   在编译之前，确保你已在当前shell下执行如下命令确保OpenVINO的环境变量已配置：

````source /opt/intel/computer_vision_sdk/bin/setupvars.sh````


-   进入dist/analytics/plugins/samples目录，运行编译脚本：

````./build_samples.sh````


此脚本同样会检查OpenVINO配置，如果你的OpenVINO未正确安装，编译将失败。

-   编译出的插件的动态库在samples目录下的build/intel64/Release/lib目录下。 

|插件名                      |             插件功能                        |
|---------------------------|--------------------------------------------|
|libcpu_extension.so        |所有plugin运行在CPU模式下时公用的扩展模块       |
|libfacedetectionplugin.so  | 人脸检测插件                                 |
|libfacerecgonitionplugin.so| 人脸识别插件                                 |
|libsmartclassroomplugin.so | 动作识别插件                                 |
|libmyplugin.so             |示例插件，仅改变输入视频中的部分颜色，不做实际分析。|

将编译出的所有动态库拷贝至````analytics_agent/lib/````目录，或者检查````analytics_agent/agent.toml````文件的````analytics.libpath````字段（默认为pluginlibs），也可将所有的动态库拷贝至改字段指向的目录（即默认为analytics_agent/pluginlibs/目录）。

测试Open WebRTC Toolkit的内置插件：
=======================================================================
我们以人脸检测插件和人脸识别插件为例介绍内置插件的使用:

准备工作
--------

在OWT Server中，视频分析插件和具体的plugin的binary的绑定是通过````analytics_agent/plugin.cfg````来实现的。

````plugin.cfg````中包含了多个内置plugin到其binary的映射, 比如：
````
[b849f44bee074b08bf3e627f3fc927c7] # 插件的唯一ID号，发起分析请求时使用
description = 'face detection plugin'  
pluginversion = 1
apiversion = 400
name = 'libfacedetectionplugin.so' # 插件的动态库名称
libpath = 'pluginlibs/' # 插件的动态库及依赖的额外加载路径
configpath = 'pluginlibs/'
messaging = true # 是否允许分析流的同时推送消息 
inputfourcc = 'I420' # 进入插件的流格式，目前必须为I420 
outputfourcc = 'I420' # 离开插件的流格式，目前必须为I420.
````

请确保测试开始前相应的动态库已拷贝至````dist/analytics_agent/lib/````目录或````dist/analytics_agent/agent.toml````中指定的目录。

-   MCU的默认Web APP中未暴露分析功能。测试时需将<https://github.com/open-webrtc-toolkit/owt-server/tree/master/test/testingpage/>中的````index.html````及````script2.js````拷贝至````dist/extras/basic_example/public````目录中。

-   为MCU的sample应用添加至````management analytics API````调用的代理接口。修改````dist/extras/basic_examples/samplertcservice.js````，添加如下的函数实现：

````
app.post('/rooms/:room/analytics', function(req, res) { 
  'use strict';
   var room = req.params.room,
       algorithm = req.body.algorithm,
       media = req.body.media; 
       icsREST.API.startAnalyzing(room, 
                                  algorithm,
                                  media, 
                                  function(info) { 
                                    res.send(info); 
                                  }, 
                                  function(err) { 
                                    res.send(err); 
                                  });
}); 

app.delete('/rooms/:room/analytics/:id', function(req, res) {
  'use strict';
  var room = req.params.room,
      id = req.params.id; 
      icsREST.API.stopAnalyzing(room,
                                id, 
                                function(result) {
                                  res.send(result);
                                }, 
                                function(err) {
                                  res.send(err); 
                                });
}); 
````

配置和启动MCU
-------------

进入编译好的MCU的dist目录，对MCU进行初始化：

````bin/init-all.sh --deps````


这一步骤将会下载openh264的库并安装至````video agent````以及````analytics agent````, 并安装和配置RabbitMQ服务和MongoDB服务。对于RabbitMQ和MongoDB，可以选择不更新账户。

然后执行：

````bin/init-all.sh````


最后执行如下命令启动MCU（执行前请确保当前shell下已经执行了”source
/opt/intel/computer_vision_sdk/bin/setupvars.sh”。内置的插件需要使用OpenVINO进行推导所以OpenVINO的环境必须配置好）：

````bin/start-all.sh````


测试视频分析功能
------------

确保你的摄像头可以使用，访问MCU的测试页面：

````https://your_mcu_url:3004````

页面可能会有若干关于证书的安全性提示。请选择信任不可信的证书。必要的时候请打开浏览器的控制台，
点开到mcu的8080端口的比如<http://server_ip:8080/socket.io/?EIO=3&transport=polling&t=MfDld6->这样的链接并选择信任证书。

刷新测试页面，本地摄像头的流将被发布到MCU。

### 准备工作：

-   在测试页面上，点开“Subscription”下拉框，选择一个当前房间中的视频流。

-   在subscribe下拉框中，选择你要分析的remote stream，track-kind可配置为“video”或者“video-and-audio”。当指定为“video”时，请配置“video codec”项目，其余项可选；当指定为“video-and-audio“时，请额外指定“audio codec”。

### 人脸检测

测试前请确保人脸检测插件已经被安装。MCU安装后，人脸检测插件的源代码位于````dist/analytics_agent/plugins/samples/face_detection_plugin/````目录下，如果需要你可以自行更改实现。

检查````analytics_agent/plugin.cfg````， 人脸检测的plugin ID为````b849f44bee074b08bf3e627f3fc927c7````。
因此，在测试页面的“Analytics“配置组中，在“analytics id” 输入框中输入该plugin id
即b849f44bee074b08bf3e627f3fc927c7，注意不要添加空格，然后点击“startanalytics”，随后你选择的流将被加入到mix流中并在页面上可以看到（人脸检测流会在人脸部位加上框）。

### 人脸识别

在进行人脸识别前，需要做一些准备工作，将人脸数据添加至本地的识别数据库（当前的插件实现例子中，读取的是analytics_agent/vectors.txt）。

在前述插件编译步骤中，实际上会编译出一个新的工具preprocess_tool提供此功能（在````plugins/samples/build/intel64/Release/````目录下），源代码位于````analytics_agent/plugins/samples/face_recognition_plugin/preprocess_tool/````目录下。

preprocess_tool会从你提供的本地照片库中提取照片，对其进行人脸检测和人脸特征提取，产生一个或多个特征向量，与照片对应的人名相关联并记录在vectors.txt中。

preprocess_tool工具有如下依赖：

1.  libcpu_extension.so， 因此在运行前请确保````build/intel64/Release/lib/````目录下的````libcpu_extension.so````在你的LD_LIBRARY_PATH下。

2.  preprocess_tool使用了Intel OpenVINO作为推导引擎，所以在运行preprocess_tool前请确保执行了````source /opt/intel/computer_vision_sdk/bin/setupvars.sh````

3.  preprocess_tool将从当前目录下的raw_photos子目录提取照片。照片应为JPG格式，每一个可以识别的人在raw_photos目录下为一个子目录，目录名应为人名。在同一人名目录下可以放置一张或多张照片。

运行preprocess_tool不需要提供其它参数。运行的结果是在同一目录下生成vectors.txt。将其拷贝到````dist/analytics_agent/````目录下。

这些准备工作完成后，可按照类似人脸检测类似的方式在测试页面上发起测试。不同之处在于````start analytics id````输入框中需要填写人脸识别的plugin ID即````3f932ff2a80341faa0a73ebb3bcfb85d````

开发及部署自己的视频分析插件
============================

MCU支持实现自定义视频插件并部署到系统中。以下是详细步骤。你也可以参考dummy插件的实现（代码在````plugins/samples/dummy_plugin/````目录下）以及人脸检测插件的实现。

插件的开发
----------

插件以一个实现类的形式存在，需要被编译成可动态加载的库。该类的接口定义在````plugins/include/plugin.h````中，类名为````rvaPlugin````.

````
class rvaPlugin {
  public:                                                                                           
 /**                                                                                              
 @brief Initializes a plugin with provided params after MCU creates the plugin.                    
 @param params unordered map that contains name-value pair of parameters                           
 @return RVA_ERR_OK if no issue initialize it. Other return code if any failure.                   
 */                                                                                                
 virtual rvaStatus PluginInit(std::unordered_map<std::string, std::string> params) = 0;           
 /**                                                                                              
 @brief Release internal resources the plugin holds before MCU destroy the plugin.                 
 @return RVA_ERR_OK if no issue close the plugin. Other return code if any failure.                
 */                                                                                                
 virtual rvaStatus PluginClose() = 0;                                                               
 /**                                                                                              
 @brief MCU will use this interface to fetch current applied params on the plugin.                 
 @param params name-value pair will be returned to the MCU provided unordered_map.                 
 @return RVA_ERR_OK if params are successfull filled in, or empty param is provided.               
 Other return code if any failure.                                                                  
 */                                                                                                
 virtual rvaStatus GetPluginParams(std::unordered_map<std::string, std::string> &params) = 0;     
 /**                                                                                              
 @brief MCU will use this interface to update params on the plugin.                                
 @param params name-value pair to be set.                                                          
 @return RVA_ERR_OK if params are successfull updated.                                             
 Other return code if any failure.                                                                  
 */                                                                                                
 virtual rvaStatus SetPluginParams(std::unordered_map<std::string, std::string> params) = 0;      
 /**                                                                                              
 @brief MCU pushes a video frame to the plugin for processing. Note this processing                
 must be asynchronous on other thread and should return immediately to caller.                      
 @param frame the video frame for processing                                                       
 @return RVA_ERR_OK if no issue. Other return code if any failure.                                 
 */                                                                                                
 virtual rvaStatus ProcessFrameAsync(std::unique_ptr<owt::analytics::AnalyticsBuffer> buffer) = 0;
 /**                                                                                              
 @brief Register a callback on the plugin for receiving frames from the plugin.                    
 @param pCallback the frame callback function registered by MCU.                                   
 @return RVA_ERR_OK if no issue. Other return code if any failure.                                 
 */                                                                                                
 virtual rvaStatus RegisterFrameCallback(rvaFrameCallback* pCallback) = 0;                         
 /**                                                                                              
 @brief unregister the pre-registered frame callback on the plugin.                                
 @return RVA_ERR_OK if no issue. Other return code if any failure.                                 
 */                                                                                                
 virtual rvaStatus DeRegisterFrameCallback() = 0;                                                   
 /**                                                                                              
 @brief Register a callback on the plugin for receiving events from the plugin.                    
 @param pCallback the event callback function registered by MCU.                                   
 @return RVA_ERR_OK if no issue. Other return code if any failure.                                 
 */                                                                                                
 virtual rvaStatus RegisterEventCallback(rvaEventCallback* pCallback) = 0;                         
 /**                                                                                              
 @brief unregister the pre-registered event callback on the plugin.                                
 @return RVA_ERR_OK if no issue. Other return code if any failure.                                 
 */                                                                                                
 virtual rvaStatus DeRegisterEventCallback() = 0;                                                   
 };                                                                                                 
````                                                                                                

插件的实现中比较关键的API主要有```PluginInit()```,  ````ProcessFrameAnsync()```` 和````RegisterFrameCallback()````.

在````PluginInit()````调用中，一般我们进行模型的加载等初始化工作。由于模型加载比较耗时，需要在其他线程中异步进行，主要包括推导引擎的初始化，extension的加载，神经网络模型的加载，神经网络输入输出的配置等等。

推导引擎的初始化及extension的加载的示例如下：
````
if (pluginsForDevices.find(deviceName) != pluginsForDevices.end()) {
  continue;
}

InferencePlugin plugin = PluginDispatcher({""}).getPluginByDevice(deviceName);

/* Load extensions for the CPU plugin */
if ((deviceName.find(device_for_faceDetection) != std::string::npos) && deviceName == "CPU") {
  plugin.AddExtension(std::make_shared<Extensions::Cpu::CpuExtensions>());
} 

pluginsForDevices[deviceName] = plugin;
Load(FaceDetection).into(pluginsForDevices[device_for_faceDetection]);
````	


加载网络模型以及配置网络输入输出的示例代码如下：
````
    InferenceEngine::CNNNetReader netReader;
    /** Read network model **/
    netReader.ReadNetwork(commandLineFlag);
    /** Set batch size to 1 **/
    std::cout << "Batch size is set to  "<< maxBatch << std::endl;
    netReader.getNetwork().setBatchSize(maxBatch);
    /** Extract model name and load it's weights **/
    std::string binFileName = fileNameNoExt(commandLineFlag) + ".bin";
    netReader.ReadWeights(binFileName);
    /** Read labels (if any)**/
    std::string labelFileName = fileNameNoExt(commandLineFlag) + ".labels";

    std::ifstream inputFile(labelFileName);
    std::copy(std::istream_iterator<std::string>(inputFile),
              std::istream_iterator<std::string>(),
              std::back_inserter(labels));
    

    /* SSD-based network should have one input and one output  */
    /* ---------------------------Check inputs ---------------- */
    InferenceEngine::InputsDataMap inputInfo(netReader.getNetwork().getInputsInfo());
    if (inputInfo.size() != 1) {
        throw std::logic_error("Face Detection network should have only one input");
    }
    auto& inputInfoFirst = inputInfo.begin()->second;
    inputInfoFirst->setPrecision(Precision::U8);
    inputInfoFirst->getInputData()->setLayout(Layout::NCHW);

    /*---------------------------Check outputs ------------------*/
    InferenceEngine::OutputsDataMap outputInfo(netReader.getNetwork().getOutputsInfo());
    if (outputInfo.size() != 1) {
        throw std::logic_error("Face Detection network should have only one output");
    }
    auto& _output = outputInfo.begin()->second;
    output = outputInfo.begin()->first;

    const auto outputLayer = netReader.getNetwork().getLayerByName(output.c_str());
    if (outputLayer->type != "DetectionOutput") {
        throw std::logic_error("Face Detection network output layer(" + outputLayer->name +
            ") should be DetectionOutput, but was " +  outputLayer->type);
    }

    if (outputLayer->params.find("num_classes") == outputLayer->params.end()) {
        throw std::logic_error("Face Detection network output layer (" +
            output + ") should have num_classes integer attribute");
    }

    const int num_classes = outputLayer->GetParamAsInt("num_classes");
    if (labels.size() != num_classes) {
        if (labels.size() == (num_classes - 1)) 
            labels.insert(labels.begin(), "fake");
        else
            labels.clear();
    }
    const InferenceEngine::SizeVector outputDims = _output->dims;
    maxProposalCount = outputDims[1];
    objectSize = outputDims[0];
    if (objectSize != 7) {
        throw std::logic_error("Face Detection network output layer should have 7 as a last dimension");
    }
    if (outputDims.size() != 4) {
        throw std::logic_error("Face Detection network output dimensions not compatible shoulld be 4, but was " + std::to_string(outputDims.size()));
    }
    _output->setPrecision(Precision::FP32);
    _output->setLayout(Layout::NCHW);
    input = inputInfo.begin()->first;
    return netReader.getNetwork();
}

````

````ProcessFrameAsync````函数是接受到MCU传入的视频帧的接口。其原型为：
````
rvaStatus ProcessFrameAsync(std::unique_ptr<owt::analytics::AnalyticsBuffer> buffer)；
````
传入的视频的buffer为**I420**格式，且Y/U/V平面在内存中连续分布。buffer中的数据在调用完毕后会自动销毁。

下面是dummy插件中ProcessFrameAsync的实现（不包含推导逻辑，你可以参考人脸检测插件的实现了解如何使用OpenVINO进行推导）：
````
rvaStatus MyPlugin::ProcessFrameAsync(std::unique_ptr<owt::analytics::AnalyticsBuffer> buffer) {
  if (!buffer->buffer) { 
    return RVA_ERR_OK; 
  } 
  if (buffer->width >=320 && buffer->height >=240) {
    memset(buffer->buffer + buffer->width * buffer->height, 28, buffer->width * buffer->height/16); 
  } 
  if (frame_callback) {
    frame_callback->OnPluginFrame(std::move(buffer)); 
  } 
  return RVA_ERR_OK; 
}
````

请注意这里的
````frame_callback-\>OnPluginFrame(std::move(buffer))````调用。MCU通过````RegisterFrameCallback（）````调用向plugin注册接受分析后的视频帧的回调对象。 Plugin在获得一帧处理过的I420格式的视频数据后，通过该回调对象的````OnPluginFrame()````方法向MCU返回处理过的帧以进行编码和传输等处理。

我们可以看到，dummy插件中的RegisterFrameCallback的实现如下。一般的插件都可以这样实现：
````
virtual rvaStatus RegisterFrameCallback(rvaFrameCallback* pCallback) { 
  frame_callback = pCallback; 
  return RVA_ERR_OK; 
}
````

当你完成````rvaPlugin````接口的实现后，记得调用**DECLARE_PLUGIN**宏向MCU声明接口：
````
DECLARE_PLUGIN(YourPluginName)
````


这里的````YourPluginName````即是你的实现类的类名。

最后将你的插件编译成动态库并拷贝至与MCU内置插件同样的目录。所有你的插件使用的不在````LD_LIBRARY_PATH````中的动态库依赖也应一并拷贝过去。

插件的部署
----------

与其它MCU内置的插件一样，需要在````dist/analytics_agent/plugin.cfg````中为你的插件添加一条记录。

你需要为你的插件提供产生一个新的UUID，并在````plugin.cfg````中使用该UUID创建一个新的子字段。该子字段的属性的配置可以参考其它插件的配置。

然后重启视频分析节点让配置生效，这样你就可以在测试页面上使用你的UUID（即plugin ID）测试你自己开发的插件了：
````
bin/daemon.sh stop analytics-agent 
bin/daemon.sh start analytics-agent 
````

你也可以将自己的插件实现提交至<https://github.com/open-webrtc-toolkit/owt-server/tree/master/source/agent/plugins/samples>，为OWT MCU添加更丰富的视频分析功能。
