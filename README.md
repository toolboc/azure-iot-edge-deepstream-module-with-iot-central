# Azure IoT Edge DeepStream Module with IoT Central

Demonstration using the the [Nvidia DeepStream Module from the Azure Marketplace](https://azuremarketplace.microsoft.com/marketplace/apps/nvidia.deepstream-iot?WT.mc_id=iot-0000-pdecarlo) with [Azure IoT Central](https://docs.microsoft.com/azure/iot-central/preview/overview-iot-central?WT.mc_id=iot-0000-pdecarlo).

To setup the demo for yourself, or to use as a baseline for your own custom project, follow the instructions below to get started!


# Supplementary Materials 

* [Presentation Deck](http://aka.ms/DeepStreamIoTEdgeDeck)
* [Presentation Video](http://aka.ms/DeepStreamIoTEdgeVideo)
- Note: The Presentation Video includes a full walkthrough for setting up and using the demo content in this repo

## Prerequisites

This lab requires that you have the following:

Hardware:
* [Nvidia Jetson Nano Device](https://amzn.to/2WFE5zF)
* A [cooling fan](https://amzn.to/2ZI2ki9) installed on or pointed at the Nvidia Jetson Nano device 
* USB Webcam (Optional) 
  - Note: The power consumption will require that your device is configured to use a [5V/4A barrel adapter](https://amzn.to/32DFsTq) as mentioned [here](https://www.jetsonhacks.com/2019/04/10/jetson-nano-use-more-power/) with an [Open-CV compatible camera](https://web.archive.org/web/20120815172655/http://opencv.willowgarage.com/wiki/Welcome/OS/).

Development Environment:
- [Visual Studio Code (VSCode)](https://code.visualstudio.com/Download?WT.mc_id=iot-0000-pdecarlo)
- [Git command line](https://git-scm.com/) 

Cloud Services:
- An Active [Microsoft Azure Subscription](https://azure.microsoft.com/get-started?WT.mc_id=iot-0000-pdecarlo)

# Creating an Azure IoT Central App 

Sign up for an [Azure Account](https://azure.microsoft.com/get-started?WT.mc_id=iot-0000-pdecarlo), or sign in if you already have one.

![](./assets/azureacct.png)

Navigate to the [Build Your IoT Application section](https://apps.azureiotcentral.com/build?WT.mc_id=webinar-particle-pdecarlo) of IoT Central and select the "Custom App" template.

![](./assets/iotcentralbuild.png)

Give the IoT Central Application a name and URL (both must be globally unique). Then, select "Custom application" for the application template, choose a pricing tier, select a subscription, and choose "United States" for the location. Finally, click "Create" and wait for the deployment to complete.

![](./assets/newapplication.png)

When the deployment completes, you will be navigated to the newly deployed IoT Central Instance.

![](./assets/newdashboard.png)

If you need to find the url to your IoT Central instance, you can view a list of all of the IoT Central instances for your subscription at https://apps.azureiotcentral.com/myapps

# Create the Device Template

In your IoT Central Instance, select "Device Templates"

![](./assets/newdevicetemplate.png)

Next, select "New" and choose "Azure IoT Edge" for the device template type.

![](./assets/newdevicetemplate.png)

When prompted, do NOT upload a manifest, instead select "Skip+Review"

![](./assets/skipreview.png)

Next, choose "Create" 

![](./assets/createtemplate.png)

Next, select "Import Capability Model"

![](./assets/importcapabilitymodel.png)

Then, upload the "nvidia-jetson-nano-dcm.json" file included in the *iotcentral* folder of this repo

![](./assets/uploaddcm.png)

Next, select "Replace Manifest"

![](./assets/replacemanifest.png)

Then, upload the "iotedge-deployment-manifest.json" file include in the *iotcentral* folder of this repo

![](./assets/uploadmanifest.png)

This manifest contains the specification for the containerized modules which will be deployed to the Nvidia Jetson device by the IoT Edge Runtime. This deployment contains two modules, the "NVIDIADeepStreamSDK" module from the Azure marketplace and an additional "NVIDIANanoIotCModule". If you look carefully at this manifest, you will notice that it is configured to route all messages from the "NVIDIADeepStreamSDK" module to a filter module named "NVIDIANanoIotCModule" as specified in the routes section:

    "routes": {

        "DeepstreamToFilter": "FROM /messages/modules/NVIDIADeepStreamSDK/outputs/* INTO BrokeredEndpoint(\"/modules/NVIDIANanoIotCModule/inputs/dsmessages\")",

        "filterToIoTHub": "FROM /messages/* INTO $upstream"

    }

The "NVIDIANanoIotCModule" transforms the output from the NVIDIADeepStreamSDK module in order to conform to IoT Central's specification for use in custom dashboards etc.

Finally, select "Publish" to publish your template

![](./assets/publishtemplate.png)


# Installing IoT Edge onto the Jetson Nano Device

Before we install IoT Edge, we need to install an additional utility onto the Nvidia Jetson Nano device with:

```
sudo apt-get install -y curl nano 
```

ARM64 builds of IoT Edge are currently being offered in preview and will eventually go into General Availability.  We will make use of the ARM64 builds to ensure that we get the best performance out of our IoT Edge solution.

These builds are provided starting in the [1.0.8 release tag](https://github.com/Azure/azure-iotedge/releases/tag/1.0.8).  To install the 1.0.8 or greater release of IoT Edge, run the following from a terminal on your Nvidia Jetson device:

```
# You can copy the entire text from this code block and 
# paste in terminal. The comment lines will be ignored.

# Install the IoT Edge repository configuration
curl https://packages.microsoft.com/config/ubuntu/18.04/multiarch/prod.list?WT.mc_id=iot-0000-pdecarlo > ./microsoft-prod.list

# Copy the generated list
sudo cp ./microsoft-prod.list /etc/apt/sources.list.d/

# Install the Microsoft GPG public key
curl https://packages.microsoft.com/keys/microsoft.asc?WT.mc_id=iot-0000-pdecarlo | gpg --dearmor > microsoft.gpg
sudo cp ./microsoft.gpg /etc/apt/trusted.gpg.d/

# Perform apt update
sudo apt-get update

# Install IoT Edge and the Security Daemon
sudo apt-get install iotedge

```

## Installing the custom DeepStream Configuration

The following steps will require that you have an Nvidia Jetson device configured with the latest JetPack offering from Nvidia and an ability to interact with the device via ssh or a connected keyboard/mouse/monitor.

First, we will need to [download the latest DeepStream 4.0.2 .deb installer](https://developer.nvidia.com/deepstream-402-jetson-deb) for your Jetson Device. This will require that you create an Nvidia Developer Account. Once you have registered and logged in, you will be presented with the ability to download the SDK using the aforementioned link:

![](./assets/deepstreamdeb.png)

After downloading the .deb, navigate to the directory where the .deb installer was saved (e.g. ~/Downloads) and install it by running the following:
cd ~/Downloads
sudo apt-get install ./deepstream-4.0_4.0.2-1_arm64.deb

If you have issues installing the DeepStream SDK, refer to the [official DeepStream Developer Guide](https://docs.nvidia.com/metropolis/deepstream/dev-guide/index.html#page/DeepStream_Development_Guide%2Fdeepstream_quick_start.html) for more information.

The SDK technically does not need to be present on the host machine for use with the Azure IoT Edge DeepStream Module. However, by having it installed, we can test configurations locally and potentially make use of the included examples.

You may want to explore the "/opt/nvidia/deepstream/deepstream-4.0/samples/configs/deepstream-app" for example configurations.  

For our demonstration, we will pull in a mostly complete configuration and serve it to the Azure IoT Edge DeepStream module using an associated [bind mount](https://github.com/toolboc/azure-iot-edge-deepstream-module-with-iot-central/blob/master/deployment.template.json#L74).

To pull in this configuration, run the following in a terminal on the Nvidia Jetson Nano device:

    wget https://raw.githubusercontent.com/toolboc/azure-iot-edge-deepstream-module-with-iot-central/master/deepstream/configs/msgconv_sample_config.txt -P /opt/nvidia/deepstream/deepstream-4.0/samples/configs/deepstream-app

    wget https://raw.githubusercontent.com/toolboc/azure-iot-edge-deepstream-module-with-iot-central/master/deepstream/configs/source4_usb_rtsp_dec_infer_resnet_int8.txt -P /opt/nvidia/deepstream/deepstream-4.0/samples/configs/deepstream-app

Next, you will need to modify these configurations to match your setup.  

Take a look at the contents of msgconv_sample_config.txt with:

    nano /opt/nvidia/deepstream/deepstream-4.0/samples/configs/deepstream-app/msgconv_sample_config.txt

You will want to adjust the "id" values to describe your cameras.  If you are using more or less cameras than what is included in this configuration, you may add or remove the entries accordingly.  

Next, take a look at source4_usb_rtsp_dec_infer_resnet_int8.txt with:
    
    nano /opt/nvidia/deepstream/deepstream-4.0/samples/configs/deepstream-app/source4_usb_rtsp_dec_infer_resnet_int8.txt

The `source0` entry is an example for using a USB Camera, it is disabled by default, be sure to enable it if you are planning to use one.  This can be particularly useful for testing.

The `source1`, `source2`, and `source3` entries are for using RTSP streaming protocol with remote cameras that are connected via WiFi or Ethernet which are accessible from the Jetson Nano.  They are all enabled by default, you may need to disable or add more depending on your setup.  The Nvidia Jetson Nano can support a maximum of 8 simultaneous RTSP streams.  To use with your RTSP capable cameras, provide a valid connection in the uri section, Ex: rtsp://user:password@ipaddress:port/videosource

Following the source entries, you will notice a number of entries for `sink0` and `sink1`.  Sinks are where output is delivered and can vary on what that means depending on the type of sink employed.  

For `sink0` we employ the overlay type which draws the video output with processing results and detected bounding boxes directly to the framebuffer.  This means it will draw the output over anything currently on screen. With proper configuration, you can draw output to a window using the EglSink type but that will not be covered as it requires special configuration of x11 permissions in the IoT Edge deployment manifest and host machine in order to access the host's X11 server from a container.  

For `sink1` we employ the MsgConvBroker to format the detection output into a consumable format for Azure IoT Edge.  This produces lightweight object detection output that looks like the following:

    [101|192|264|236|458|Person]

These are the Tracking Id, 4 bounding box coordinates, and classifier object that was detected.

The other sink options demonstrate outputting to a video file.  Notice that output can also be provided as an RTSP stream served from the device.

It is important to note the `primary-gie` section of our configuration.  This specifies the object detection model to be used by DeepStream for inference processing.  The default model includes the ability to detect Car, Person, Vehicle, and Roadsign objects as described in */opt/nvidia/deepstream/deepstream-4.0/samples/models/Primary_Detector_Nano/labels.txt*.  DeepStream is compatible with Tensor-RT compatible models and it is possible to point this configuration at a compatible model if you wish to change the default.

## Testing the custom DeepStream Configuration

To test your configuration, it is important to temporarily disable `sink1` by setting enable=0, then you can run using deepstream-app to verify your video input sources are working with:

    deepstream-app -c /opt/nvidia/deepstream/deepstream-4.0/samples/configs/deepstream-app/source4_usb_rtsp_dec_infer_resnet_int8.txt

This is what this looks like when configured with three RTSP sources:

![](./assets/deepstreamtest.png)

When you have verified your configuration is working, re-enable `sink1` by setting enable=1 as it is necessary for publishing data to IoT Central when running as an IoT Edge module, which we will do in the next steps.

## Provisioning the IoT Edge Runtime on the Jetson Nano Device with DPS

To automatically provision your device with [DPS](https://docs.microsoft.com/azure/iot-dps?WT.mc_id=iot-0000-pdecarlo), you need to provide it with appropriate device connection information obtained from IoT Central.

To locate this information, in your IoT Central app, select "Devices" then choose the name of your newly created device template.

![](./assets/prepnewdevice.png)

Next, select "New" to create a new IoT Edge device using your created template:

![](./assets/createnewdevice.png)

Once created, select the newly created device and choose the "Connect" option

![](./assets/connectdevice.png)

You should see a display of information which we will use in the next steps

![](./assets/connectdeviceinfo.png)

Once you have obtained a device connection info, open the iot edge configuration file on your Jetson Nano with:

```
sudo nano /etc/iotedge/config.yaml
```

Find the provisioning section of the file and comment out the manual provisioning mode. Uncomment the "DPS symmetric key provisioning configuration" section as shown.

```
# Manual provisioning configuration
#provisioning:
#  source: "manual"
#  device_connection_string: "<ADD DEVICE CONNECTION STRING HERE>"

# DPS TPM provisioning configuration
# provisioning:
#   source: "dps"
#   global_endpoint: "https://global.azure-devices-provisioning.net"
#   scope_id: "{scope_id}"
#   attestation:
#     method: "tpm"
#     registration_id: "{registration_id}"

# DPS symmetric key provisioning configuration
provisioning:
  source: "dps"
  global_endpoint: "https://global.azure-devices-provisioning.net"
  scope_id: "{scope_id}"
  attestation:
    method: "symmetric_key"
    registration_id: "{registration_id}"
    symmetric_key: "{symmetric_key}"
```

Next update the value of `{scope_id}` with the "ID Scope" value in IoT Central, then update the value of `{registration_id}` with the "Device ID" value in IoT Central, and finally update the value of `{symmetric_key}` with either the "Primary Key" or "Secondary key" value from IoT Central.


After you have updated these values, restart the iotedge service with:

```
sudo service iotedge restart
```

Initially, your device will show a status of "Registered" in the IoT Central portal

![](./assets/deviceregistered.png)

Double-check your settings under "Administration" => "Device Connection".  If "Auto Approve" is disabled, you will need to manually approve the device.

![](./assets/autoapprovedisabled.png)

To manually approve the device, select the "Approve" option after selecting the device on the "Devices" page.  If "Auto Approve" is enabled, you may skip this step.

![](./assets/deviceapprove.png)

Once the iotedge service has restarted successfully and connected to IoT Central the status will change to "Provisioned"  

![](./assets/deviceprovisioned.png)

You can check the status of the IoT Edge Daemon on the Jetson Nano using:

```
systemctl status iotedge
```

Examine daemon logs using:
```
journalctl -u iotedge --no-pager --no-full
```

And, list running modules with:

```
sudo iotedge list
```

## Creating a Custom Dashboard in IoT Central for displaying Telemetry

Now that our newly created device is provisioned, we can use it to create visualizations for data produced by our Nvidia Jetson device.

Navigate to the recently created device template

![](./assets/devicetemplate.png)

Select "Views" => "Visualizing the Device"

![](./assets/newview.png)

Name the View "Device" and add the following properties:

![](./assets/deviceview.png)

Select "Add Tile" then "Save":

![](./assets/newviewsaved.png)

Create another View by selecting "Views" => "Visualizing the Device"

![](./assets/newdashboardview.png)

Name the View "Dashboard" and add the following:

![](./assets/dashboardhealth.png)

Select "Add Tile" then choose the gear icon and rename the chart to "System Health" the choose "Update" and "Save"

![](./assets/systemhealth.png)

Next, add a new tile by selecting "Module State" then "Add Tile", then select the measure icon and choose "Last Known Value"

![](./assets/modulestate.png)

Next, add a new tile for "Primary Detection Count" by selecting it and choosing "Add Tile" the repeate for "Secondary Detection Count".

![](./assets/detectioncounts.png)


Finally, add a new tile for the following fields and name it "events"

![](./assets/events.png)

Now we just need to verify that our dashboard is ready for publication.  Before we do that, we will test it by configuring it to use data from our current device.  Select "Configure Preview Device" then choose your device from the populated list:

![](./assets/previewdevice.png)

After a short while you should see the view populated with data from the device

![](./assets/sampledata.png)

When you are satisfied with your dashboard, publish it by selecting "Publish" 

![](./assets/publishdashboard.png)

## Viewing the Dashboard for our live device

Select "Devices" then choose your newly created device underneath the heading of your recently published device template.

![](./assets/deviceupdated.png)

You should be greeted by the "Device" dashboard which now has live information pulled from the device:

![](./assets/devicedash.png)

Select "Dashboard" to view the additional live data:

![](./assets/dashboarddash.png)

## Configuring the NVIDIANanoIotCModule from IoT Central

If you recall earlier, our model is capable of detecting Car, Person, Vehicle, and Roadsign objects as described in */opt/nvidia/deepstream/deepstream-4.0/samples/models/Primary_Detector_Nano/labels.txt*

By default, the NVIDIANanoIotCModule will forward all detections matching person as it is the default value the Primary Detection Class.  We can change this value as well as enable a Secondary Detection Class within IoT Central.

While in the device view panel, select "Manage" to bring up the screen for updating the Module Twin for the Primary and Secondary Detection Class of NVIDIANanoIotCModule.  

Update the Primary Detection Class to "person" and the Secondary Detection Class to "vehicle" then select "Save".

![](./assets/updatemodule.png)

In the picture above, we are monitoring the output of the NVIDIANanoIotCModule with:
     
     iotedge logs -f NVIDIANanoIotCModule

Now that the secondary detector is enabled, and with a vehicle in very clear view of one of the cameras, we now get results for the Secondary Detection Class.

![](./assets/secondarydetectioncount.png)

## Configuring e-mail alert with IoT Central

To create a rule, for example to alert when an Occupancy Threshold has been reached, select "Rules" and create an entry with the following settings:

![](./assets/rule.png)

## Exporting data to Azure for use in additional Services

Data can be exported from IoT Central to an available Storage Blob, Event Hub, or Service Bus. For more information, consult the relevant [documentation](https://docs.microsoft.com/azure/iot-central/preview/howto-export-data?WT.mc_id=iot-0000-pdecarlo).
