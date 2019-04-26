Using Open WebRTC Toolkit For Media Analytics
=============================================
In this document we introduce the media analytics functionality provided by Open WebRTC Toolkit(https://github.com/open-webrtc-toolkit/owt-server), namely OWT, and a step by step guide to implement your own 
media analytics plugins with Intel Distribution of OpenVION.

OWT Media Analytics Architecture
================================

The software block diagram of OWT Media Analytics：
![Analytics Arch](https://github.com/taste1981/owt-server/blob/master/doc/servermd/analytics_diagram.jpg)
OWT Server allows client applications to perform media analytics on any stream in the room with Analytics REST API, including the streams from WebRTC access node or SIP node, and streams from video mixing node. 
A successful media analytics request will generate a new stream in the room, which can be subscribed, analyzed, recorded or mixed like other streams.


Process Model
-------------
MCU will fork a new process for each analytics request by forming an integrated media analytics pipeline including decoder, pre-processor, video analyzer and encoder. Compressed bitstreams flow in/out of the media
analytics pipeline through the common Internal In/Out infrastrcture provided by MCU.

In current implementation, for each media analytics pipeline, only one media analaytics plugin is allowed to be loaded. The plugin loaded by the pipeline is controlled by the algorithm ID parameter in analytics
REST request.

Build and Installation
======================

System Requirement
------------------

Hardware： Intel(R) 6th - 8th generation Core(TM) platform with Intel integrated graphics.

OS： CentOS 7.4/7.6 or Ubuntu 16.04/18.04

Installation Steps
------------------

### Install OpenVINO

Follow the guideline from <https://docs.openvinotoolkit.org/latest/_docs_install_guides_installing_openvino_linux.html> to install Intel OpenVINO 2018 R5 which is the version currently supported by Open WebRTC Toolkit. 
Make sure OpenCV is installed during the installation process, and the samples provided by OpenVINO can be successfully run.


-   Inferencing with GPU requires OpenCL driver to be installed. If you have not installed OpenCL, make sure you go to ````/opt/intel/computer_vision_sdk/install_dependencies/````directory, and run:
````
sudo -E ./install_NEO_OCL_driver.sh
````

-   If other version of OpenVINO is already installed, make sure you uninstall before installing R5. The uninstallation script is typically under
````/opt/intel/computer_vision_sdk/openvino_toolkit_uninstall/````directory.

### Build Open WebRTC Toolkit Server

Download the server code from <https://github.com/open-webrtc-toolkit/owt-server> and following the README to build MCU server.

-   Install the build dependencies：

````scripts/installDeps.sh````


-   Build all modules including analytics module：

````scripts/build.js -t all --check````


-   Pack up all modules into MCU binary：

````scripts/pack.js -t all --install-module --sample-path \${webrtc-javascript-sdk-sample-conference-dist}````


Please be noted ````sample-path```` here should be pointed to the ````owt-client-javascript/dist/sample/conference```` directory which is built from OWT Javascript SDK. For more details about building OWT
Javascript SDK, pleae refer to <https://github.com/open-webrtc-toolkit/owt-client-javascript>.


Besides, MCU dependes on Node.js version 8. For installation of Node.js, refer to <https://nodesource.com/blog/installing-node-js-tutorial-using-nvm-on-mac-os-x-and-ubuntu/>.

### Build and Deploy Aanalytics Plugins Shipped with Open WebRTC Toolkit


After building and packing MCU, the analytics plugins shipped with MCU are in ````dist/analytics_agent/plugins/samples/```` directory as source code. 
You can also find the source on github: <https://github.com/open-webrtc-toolkit/owt-server/tree/master/source/agent/plugins>.


-   Before building the plugins, make sure OpenVINO environment is setup in current shell:

````source /opt/intel/computer_vision_sdk/bin/setupvars.sh````


-   Go to ````dist/analytics/plugins/samples```` directory, and run:

````./build_samples.sh````

The script will also check OpenVINO enviroment, and build will fail if your OpenVINO is not correctly installed.

-   The output dynamic libraries for plugins are under ````build/intel64/Release/lib/```` directory.

|Plugin library name        |             Functionality                       |
|---------------------------|-------------------------------------------------|
|libcpu_extension.so        |Shared CPU extesion for all plugins              |
|libfacedetectionplugin.so  |Face detection plugin                            |
|libfacerecgonitionplugin.so|Face recognition plugin                          |
|libsmartclassroomplugin.so |Action detection plugin                          |
|libmyplugin.so             |Dummy plugin that shows how to write plugins     |

Copy all output library files to ````analytics_agent/lib/```` directory, or to path specified by ````analytics.libpath````section in ````dist/analytics_agent/agent.toml```` file, 
which is by default ````dist/analytics_agent/pluginlibs/````directory.


Test Plugins Shipped with Open WebRTC Toolkit
=============================================
We introduce the usage of plugins shipped with OWT here, using face detection and face recognition plugins as examples:

Preparation
-----------

In OWT server, the analytics algorithms and plugin binaries are bound by ````dist/analytics_agent/plugin.cfg````configuration file.

````plugin.cfg````by default contains multiple mappings from algorithm to plugin binaries, for example:
````
[b849f44bee074b08bf3e627f3fc927c7] # Plugin unique ID used when issuing analytics request
description = 'face detection plugin'  
pluginversion = 1
apiversion = 400
name = 'libfacedetectionplugin.so' # plugin library name
libpath = 'pluginlibs/' # extra LD_LIBRARY_PATH for plugin, relative to dist/analytics_agent directory
configpath = 'pluginlibs/'
messaging = true # True if allows sending message from the plugin
inputfourcc = 'I420' # Video frame format into the plugin, currently must be I420
outputfourcc = 'I420' # Video frame format out of the plugin, currently must be I420
````

Make sure you copy plugin binaries to ````dist/analytics_agent/lib/````directory or the path specified in ````dist/analytics_agent/agent.toml````.

-   The basic_sample of MCU does not expose analytics functionality on UI so you have to replace the test page. Copy from <https://github.com/open-webrtc-toolkit/owt-server/tree/master/test/testingpage/> the ````index.html```` and ````script2.js```` to ````dist/extras/basic_example/public/````directory。

-   Add proxy API call to ````management analytics API````for analytics in ````dist/extras/basic_examples/samplertcservice.js````, adding below code:

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

Configure and Start MCU
-----------------------

Go to dist directory of MCU and install deps：

````bin/init-all.sh --deps````


Openh264 library will be installed for ````video agent```` and ````analytics agent````. Also RabbitMQ and MongoDB services will be installed and configured. You can skip the steps of updating RabbitMQ and MongoDB user accout update steps by selecting not to update.


Then finish the initialization：

````bin/init-all.sh````


Finally start up MCU(make sure you set OpenVION enviroment before starting MCU):

````bin/start-all.sh````


Test Media Analytics Plugins
----------------------------

Make sure the camera is accessible and start up Chrome browser on your desktop:

````https://your_mcu_url:3004````

There might be some security prompts about certificate issue. Select to trust those certificates. You might need to open the browser console to accept certificates for SSL connection to MCU port 8080, for example: <http://server_ip:8080/socket.io/?EIO=3&transport=polling&t=MfDld6->

Refresh the test page and your local stream should be published to MCU.


### Preparation：

-   On the test page, drop down from “Subscription” and select a remote stream in current room.

-   From the "subscribe" dropdown, select the remote stream you would like to analyze. the track-kind can be configured to "video" or "video-and-audio". If you select "video", make sure "video codec" item is specified, 
and other options are optional. If you specify "video-and-audio", please specify "audio codec" as well.

### Face Detection

Make sure face detection plugin has been installed. The source code of face detection plugin is under ````dist/analytics_agent/plugins/samples/face_detection_plugin/````, you can modify the implementatio if neccessary.


Check ````analytics_agent/plugin.cfg````， The plugin ID for face detection is ```b849f44bee074b08bf3e627f3fc927c7````, so on the test page, in "Analytics" configuration group, input the plugin ID in "analytics id" 
edit control, that is,  ````b849f44bee074b08bf3e627f3fc927c7```` without any extra spaces, and press "startanalytics" button. The stream you selected will be analyzed and mixed into mix streaam, with annotation on the faces in the stream.


### Face Recognition

Before face recognition, you need to add information into local database for the person you would like to recognize. In current plugin implementation, the database we read from is under ````dist/analytics_agent/vectors.txt````.

Actually when you build the plugins shipped with MCU, a tool named "preprocess_tool" will also be built under ````plugins/samples/build/intel64/Release/```` directory, and its source code is under ````analytics_agent/plugins/samples/face_recognition_plugin/preprocess_tool/```` directory.


It will fetch pictures from your local album，do face detection and feature abstraction，generating multiple feature vectors，associate them with the person's name correponding the the picture, and store them in vectors.txt.

A few dependencies of preprocess_tool:

1.  libcpu_extension.so， Please make sure ````libcpu_extension.so```` is within the LD_LIBRARY_PATH.

2.  preprocess_tool uses Intel OpenVINO for inferencing, so make sure OpenVINO enviroment is setup when you run the tool.

3.  preprocess_tool will extract pictures from ````raw_photos```` directory, with each person's name as sub-directory name. You can put one or more JPG format pictures for a person under each sub-directory.

No other inputs are required. The resultant vectors.txt will be generated under the same directory. Copy it to ````dist/analytics_agent/```` directory.

After that, you can follow the same procedure to perform face recgonition by replacing the plugin ID to ````3f932ff2a80341faa0a73ebb3bcfb85d```` before you press ````start analytics id```` button.


Develop and Deploy Your Own Media Analytics Plugins
===================================================

MCU supports implementing your own medai analytics plugins and deploy them. Below are the detailed steps. You can also refer to dummy plugin(under ````plugins/samples/dummy_plugin/````) or face detection plugin for more details.


Develop Plugins
---------------

Plugin exists in the format of an implementation class of ````rvaPlugin```` interface,  which will be built into an .so library. The defnition of ````rvaPlugin```` interface is under ````plugins/include/plugin.h````.

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

The main interfaces for a plugin implementation are ```PluginInit()```,  ````ProcessFrameAnsync()```` 和````RegisterFrameCallback()````.

In ````PluginInit()````implemntation，you should create a new thread for some time-consuming tasks such as inferencing engine initialization, model and extension loading, as well as configuring the input/output of the networks.

The sample code of inferencing engine initialization and extension loading:
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


The sample for loading networks and input/output configuration:
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

````ProcessFrameAsync()```` accepts frames from MCU and its prototype is：
````
rvaStatus ProcessFrameAsync(std::unique_ptr<owt::analytics::AnalyticsBuffer> buffer)；
````
The input buffer foramt is **I420**，with Y/U/V plane layout continually in memory. Buffer will be freed after the call.

Below we show the ProcessFrameAsync implementation in dummy plugin without inferencing involved. For samples of inferencing in ProcessFrameAsync you can refer to face detection plugin.
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

Be noted ````frame_callback->OnPluginFrame(std::move(buffer))```` here. MCU calls ````RegisterFrameCallback（）```` to register a callback object to plugin for receiving frames after analytics.
After plugin output an I420 frame that is already analyzed, it will invoke the callback object's ````OnPluginFrame()```` callback to return the frame to MCU for further encoding and transport.


The ````RegisterFrameCallback```` interface should be implemented like：
````
virtual rvaStatus RegisterFrameCallback(rvaFrameCallback* pCallback) { 
  frame_callback = pCallback; 
  return RVA_ERR_OK; 
}
````

After you finish ````rvaPlugin```` interface implementation，Please include such **DECLARE_PLUGIN** macro to delcare the plugin to MCU：
````
DECLARE_PLUGIN(YourPluginName)
````


Here ````YourPluginName```` must be the class name of your ````rvaPlugin```` implementation.

Finally build and copy all binaries to the same directory where you put plugins shipped with MCU, including the libaries you plugin depends on that are not within ````LD_LIBRARY_PATH````.


Deploy Plugins
--------------

Like other plugins shipped with MCU, you need to add an entry into ````dist/analytics_agent/plugin.cfg```` for your plugin by generating a new UUID, using that to start a new section in ````plugin.cfg````.

Restart analytics agent to make the changes effective, and then you can use the new UUID(the plugin ID) to test your plugin:
````
bin/daemon.sh stop analytics-agent 
bin/daemon.sh start analytics-agent 
````

You are also welcomed to submit your plugins to <https://github.com/open-webrtc-toolkit/owt-server/tree/master/source/agent/plugins/samples> to add more new analytics features for OWT MCU.
