# Lab 3 - Organising Things with Gateways

## Overview

In the previous lab, we secured the connection between the MQTT broker and the
clients. However, there is still possibilities for unintended inter-device cross-talk.
You can restrict this by designing good policies and this was covered in one of
the optional tasks of the last lab. If the intention is to organize Things
together in a logical fashion, e.g., at the customer level or at a level of
a floor of a building, then setting per customer and per device policy does not
scale. We need a better way of organizing them and in process make sure that
the devices of one customer cannot unintentionally connect with the devices of another customer. 

Gateways are one good way to achieve vertical hierarchy among things. Policies
are good way of managing horizontal hierarchy.  For instance, it is useful to
have a write only policy to a certain topic for all the data coming from all
temperature sensors sold to all customers, while the gateways prevent reading
one customer's data by another (malicious) customer. Gateways also act as 
edge devices and provide local computational infrastructure. 

There are several concerns with the gateway pattern. You need to design and
sell an additional piece of hardware. The Things and the gateway need to discover
each other and establish a secure connection to allow dynamic scaling of Things
infrastructure. A problem that is exacerbated when Things share or change
association between the gateways. The communication should not only be
coordinated between gateway and Things, but also between the Things and your
cloud server. The logic (software) for collecting and analysing data from
Things connected to each gateway may differ and this requires a scalable
solution to deploy and manage logic across all of the gateways.

In this lab, we will demonstrate a secure gateway deployment using AWS
Greengrass. AWS Greengrass is a software that extends cloud capabilities to
local devices, negating many of the concerns associated with the gateway
pattern expressed above. Among other packages, Greengrass software consists of
a MQTT broker that acts as a bridge between AWS IoT broker and Things, a
computing platform similar to cloud instances but on the gateway, and a PKI
based secure communication setup. Greengrass coordinates with AWS IoT service
and helps your Things to "automagically" discover the closest deployed gateway
and connect to it. Once the connection is established, the Things can collect
and analyse data closer to the source of information, react autonomously to
local events, and communicate securely with each other within local network
even when the gateway is cut off from the internet.

![](./figs/l3_gg_overview.jpg)

The goal of this lab is to understand how scalable gateways can be deployed,
while managing a secure discovery and connection processes between Things. 
[Appendix](#appendix) gives a brief
explanation on various concepts that you will come across during the lab
exercises and walks you through discovery and connection management.

## Setup your local machine

You will need some software installed on your local machine.

Commands run in Raspberry Pi is preceded `rpi>` and in your local machine as `local>`.
These should be excluded when entering the commands. 

### SSH client
If you use a Mac or Linux machine, chances are very good that you already
have an ssh-client installed; just open a terminal and try the `ssh` command.

If you have a Windows machine you can, e.g., use [Putty](https://www.putty.org/)
or [Termius](https://termius.com/). The normal ssh client can also be available 
in you Windows terminal. 

### Python environment
You will need a working Python 3 environment on your laptop. You will need to set 
this up yourself.

In addition you will need the following Python packages installed `awscrt`, `awsiot`, 
`awsiotsdk`, and `AWSIoTPythonSDK`. These can be installed with
```
local> pip install awscrt awsiot awsiotsdk AWSIoTPythonSDK
```

### Clone the AWSIoTPythonSDK repository
We will need some example files available in the `AWSIoTPythonSDK-v2` repository on GitHub
https://github.com/aws/aws-iot-device-sdk-python-v2. These are included in the repository 
for the labs, or you can clone it separately
```
local> git clone https://github.com/aws/aws-iot-device-sdk-python-v2
```

## Setup the Raspberry PI

We will use a Raspberry Pi as the device gateway. The device is already
powered up and connected to the network. To connect to the Raspberry Pi,
you need its IP address. The IP address will be given to you during the lab.
To be able to reach your gateway from the VM, your laptop also needs to 
be conected to the same network using the same wireless network as in Lab 1. 

To connect to the gateway either use your graphical ssh client and use it to 
connect to the IP address of the Raspberry PI. If you are using the terminal-
based ssh client, the command is
```bash
rpi> ssh pi@IPADDRESS
```
The username is `pi` and password is `raspberry`.
  

3. Check that Greengrass is not installed and running on the RPI by issuing:
   ```
   rpi> sudo systemctl status greengrass.service
   ``` 
   If it is installed issue the following commands before continuing 
   ```
   rpi> sudo systemctl stop greengrass.service
   rpi> sudo systemctl disable greengrass.service
   rpi> sudo rm /etc/systemd/system/greengrass.service
   rpi> sudo systemctl daemon-reload && sudo systemctl reset-failed
   rpi> sudo rm -rf /greengrass/v2
   ```

## Setup Greengrass Core Device

To setup the Greengrass software on the RPI we will use the Greengrass device setup script provided by AWS. This script will
1. Configure your device and installs the AWS IoT Greengrass Core software on it.
2. Configure your cloud-based resources.

### Setup the AWS access key
First we will setup an access key. You only need to do this once per group (as long as you remember the key).
1. In the AWS Console press your user name in the upper right corner and select _Security credentials_. You do not have permissions to most of the settings on this page, but under _Access keys_ there is the option for you to create a new one. Press _Create access key_ select _Other_ and press _Next_. Provide a description, e.g., your group members and press _Create access key_. 
2. Save the _Access key_ and _Secret access key_ to a text file on the VM (you can for example select _Download .csv file_).
3. Press _Done_.

**Note:** If the _Create access key_ button is deactivated there might already be two assigned keys (which is the maximum). _Deactivate_ and _Delete_ the old keys before adding a new one! Check that the user is not using the access keys before deactivation.

### Setup the Greengrass software

**Very important:** Make sure that you are in the Frankfurt (eu-central-1) region on AWS! Otherwise you will run into trouble.

Perform the following steps to setup the Greengrass software:

1. In the AWS Console go to _Services > Internet of Things > IoT Greengrass_. 
2. Under _Manage_ select _Greengrass devices > Core devices_. 
3. Press the button _Set up one core device_. 
4. Give the core device a unique name that you will remember. For example _GreenGrassCore-groupname_. Write down the name of the core.
5. Select _Enter a new group name_ under _Thing group_. Give the group a unique name, for example, _GreengrassGroup-groupname_. Write down the name of the group.
6. Select _Linux_ under _Operating System_.
7. Some credentials are needed for the script to be able to communicate with the corresponding AWS services. These are the _Access key_ and _Secret access key_ generated in the previous step. It is easiest if these are saved as environmental variables in the shell you are using on the RPI. Execute the following commands on the RPI where the keys are replaced with their correct values. Also remember to remove the `pi>` statement:
   ```
   rpi> export AWS_ACCESS_KEY_ID=AKIAIOSFODNN7EXAMPLE
   rpi> export AWS_SECRET_ACCESS_KEY=wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
   ```
8. Download and start the script according to the instructions on the AWS Console. You can use `curl` to download the script.
9. Run the installer by executing the **specific** command given under _Run the installer_. Make sure you understand what the command does and that it executes successfully, i.e., that there are no error messages. It should output something similar to:
   ```
   Provisioning AWS IoT resources for the device with IoT Thing Name: [GreengrassQuickStartCore-andreas]...
   Found IoT policy "GreengrassV2IoTThingPolicy", reusing it
   Creating keys and certificate...
   Attaching policy to certificate...
   Creating IoT Thing "GreengrassQuickStartCore-andreas"...
   Attaching certificate to IoT thing...
   Successfully provisioned AWS IoT resources for the device with IoT Thing Name: [GreengrassQuickStartCore-andreas]!
   Adding IoT Thing [GreengrassQuickStartCore-andreas] into Thing Group: [GreengrassGroupAndreas]...
   Successfully added Thing into Thing Group: [GreengrassGroupAndreas]
   Setting up resources for aws.greengrass.TokenExchangeService ...
   Attaching TES role policy to IoT thing...
   No managed IAM policy found, looking for user defined policy...
   IAM policy named "GreengrassV2TokenExchangeRoleAccess" already exists. Please attach it to the IAM role if not already
   Configuring Nucleus with provisioned resource details...
   Downloading Root CA from "https://www.amazontrust.com/repository/AmazonRootCA1.pem"
   Created device configuration
   Successfully configured Nucleus with provisioned resource details!
   Creating a deployment for Greengrass first party components to the thing group
   Configured Nucleus to deploy aws.greengrass.Cli component
   Successfully set up Nucleus as a system service
   ```
   **Note:** If you get an error message about `<username> is not authorized to perform: iam:PassRole` then check that you are in the Frankfurt region (eu-central-1) in both the AWS console and in your command.
10. Press _View core devices_ to return to the list. Your device should now be visible in the list and its status should be _Healthy_. 



## Create Things

Unlike in previous labs, we will not use the Arduino WiFi board to connect to the
Greengrass Core.  We will instead create a simulated Thing in the terminal. This is because the
discovery process requires additional library that interacts with AWS IoT core
using REST APIs. In theory, we can do this with Arduino, but this will require
writing an additional library, something Arduino is lacking at this time.
Connecting Arduino WiFi to Greengrass is a nice little project of its own and
you are welcome to try it in your free time :)

The simulated Thing will be your terminal from your computer. To create a new
thing, we will create the key pair and register the public key with AWS IoT in
one step. 

1. Go to _Manage > All devices > Things_. Press _Create things_ and select _Create single thing_ and press _Next_.  
2. Enter a unique name for your thing, e.g., `simthing_GROUPNAME` and press _Next_. Remember this name!
3. We want AWS to generate the certificates and keys that will be used to authorize the communication from the thing, so use the default _Auto-generate a new certificate (recommended)_ and press _Next_. 
4. In the _Policies_ window, select the `GreengrassV2IoTThingPolicy` and just press _Next_. You can review what the policy contains by clicking on the name of it.
5. In the window _Download certificates and keys_, download the device certificate, public and private keys and store them in a folder
   `Publisher_Sim` on your machine. You can rename the certificate, public and private key as
   `publisher_sim.pem.crt`, `publisher_sim-public.pem.key` and
   `publisher_sim-private.pem.key`. Note that if you forget to save these at this stage, you will need to recreate the thing!
6. Finally, download the RSA 2048 root ca from the window. You can also download it from
   [here](https://www.amazontrust.com/repository/AmazonRootCA1.pem). Save it as `root_ca.pem` in the same folder as they previous certificate and keys. 
7. Make sure that the Thing you created is visible in the list of things. 
8. Open the Thing you created and verify that the certificate is active under the _Certificates_ tab.

Now we need to connect the thing we created to the Greengrass core device. 
1. Under _Manage > Greengrass devices > Core devices_ open the Greengrass core device you created in the previous step by clicking on its name. 
2. There are several tabs on the detail page of the core device (the RPI's Greengrass instance). 
   - Components: Here all the different software components deployed to the core device is listed. For example, we have an MQTT broker (Moquette) and a MQTT bridge deployed. It is also possible to deploy what components are deployed under _Manage > Greengrass devices > Deployments_. We will look at this in the next lab.
   - Client devices: here we can add local Things that will communicate through the Greengrass instance installed on the RPI. These can be, e.g., sensors or actuators (or our simulated thing in this case).
3. To setup discoverability for the things, select the tab _Client devices_ and press the button _Configure cloud discovery_. Select _Target type_ as _Core device_. The _Target name_ field should be prepopulated with the name of the Greengrass core device name you have created.
4. Press the button _Associate client devices_. Enter the name of your created thing into the _AWS IoT thing name_ and press _Add_. Then press _Associate_. Make sure that your thing is now in the list _Associated client devices_ on the _Client devices_ tab.
5. Under _Step 3_ the following items should be checked.

   ![](./figs/cloud_discover_step3.png)

6. Under _Client device auth_ press _Edit configuration_ and add the following under _Configuration to merge_:
   ```
   {
      "deviceGroups": {
         "formatVersion": "2021-03-05",
         "definitions": {
            "MyDeviceGroup": {
               "selectionRule": "thingName: *",
               "policyName": "MyClientDevicePolicy"
            }
         },
         "policies": {
            "MyClientDevicePolicy": {
               "AllowConnect": {
                  "statementDescription": "Allow client devices to connect.",
                  "operations": [
                     "mqtt:connect"
                  ],
                  "resources": [
                     "*"
                  ]
               },
               "AllowPublish": {
                  "statementDescription": "Allow client devices to publish to all topics.",
                  "operations": [
                     "mqtt:publish"
                  ],
                  "resources": [
                     "*"
                  ]
               },
               "AllowSubscribe": {
                  "statementDescription": "Allow client devices to subscribe to all topics.",
                  "operations": [
                     "mqtt:subscribe"
                  ],
                  "resources": [
                     "*"
                  ]
               }
            }
         }
      }
   }
   ```
   Press _Confirm_. 
7. Under _MQTT bridge_ press _Edit configuration_ and add the following under _Configuration to merge_:
   ```
   {
      "mqttTopicMapping": 
         {
         "GroupName_LocalToCloud": {
            "topic": "saiot/GROUPNAME/publish",
            "source": "LocalMqtt",
            "target": "IotCore"
         }
      }
   }
   ```
   Remeber to change GROUPNAME to something unique for your group. You need to write this down as well. Press _Confirm_. 
   
   This will forward MQTT messages arriving from the local MQTT broker running on the Greengrass instance on the RPI (_LocalMqtt_) to the cloud (_IoTCore_). In the next lab we will need to modify this to allow for bidirectional traffic, but this is fine for now.
8. Press _Review and deploy_. 
9. Press _Deploy_.

Now go back to _Manage > Greengrass devices > Deployments_. The deployment you modified will first have the status `Active` which indicates that the changes are being processed on the Greengrass instance on the RPI. After it is finished it will change to `Completed` (it might take over 10 minutes).

To troubleshoot you can view the Greengrass logs on the RPI by executing the following command
```
rpi> sudo cat /greengrass/v2/logs/greengrass.log
```
or 
```
rpi> sudo tail -f /greengrass/v2/logs/greengrass.log
```
to also print new lines that are printed in the log file. 

## Test the connection

1. From _AWS IoT_ -> _Test_ -> _MQTT test client_, setup a new subscriber to the topic
   _saiot/GROUPNAME/publish_. Select _Display payloads as strings (more
   accurate)_ option and then _Subscribe_.
2. On your machine, run the following commands
   
   ```
   local> python aws-iot-device-sdk-python-v2/samples/pubsub.py --endpoint ENDPOINT --key publisher_sim-private.pem.key --cert publisher_sim.pem.crt --client_id simthing_GROUPNAME --topic 'saiot/GROUPNAME/publish' --message 'Hello World From GroupName' 
   ```
   You can get the ENDPOINT from _AWS IoT_ -> _Domain configurations_. The value you are searching after is the _Domain name_. The names of topic must match with topic name in all the
   above steps. Client id option must match the name of the thing.

3. Verify in the _MQTT test client_ that your messages are received.

### Troubleshooting 

If the connection is not established, verify the endpoint is correct. Also verify that the certificate paths are correct. 

The client_id should be same as the Thing name under _Manage > All devices > Things_. 

Another thing to check is that the certificate for the thing is activated: Open your thing under _Manage > All devices > Things_. Under the _Certificates_ tab, there should be a certificate that is indicated as `Active`. 



## To do

1. Get a working understanding of the code in `pubsub.py`. The documentation can be found
   [here](https://github.com/aws/aws-iot-device-sdk-python-v2). 
2. Explain how the Thing connects to your gateway, including TLS mutual authentication. Use relevant diagrams. (3 p)
3. Modify `pubsub.py` to measure the latency of establishing the connection to the gateway. Use, for example, the `time`
   module and `perf_counter()`. See
   [documentation](https://docs.python.org/3/library/time.html#time.perf_counter). Compare the measured time with the ping value optained from your local machine to your gateway (`ping IPADRESS`). (7 p)
   
## Optional Task



### Verify privacy (3 points)

Repeat the previous steps to create another Thing. Call it _Snoopy_subscriber_GROUPNAME_. We will
try to simulate a malicious subscriber using this Thing. After creation, open the _Snoopy_subscriber_GROUPNAME_ thing (under _All Things_, _Things_). Then open the _Certificates_ tab and select the certificate. Now press _Detach_ to deactivate it.

Instead of using the `simthing_GROUPNAME`, use `Snoopy_Subscriber_GROUPNAME` and try
to publish or subscribe to the same topic. 

## Appendix

A Greengrass _Group_ consists of exactly one _Core_ along with a logical
collection of Things. The _Core_ manages connection between Things, gateway and
IoT services along with providing a platform to deploy compute instances from
the cloud. AWS calls this computation platform AWS Lambda and if time
permits, we will explore this in the next lab. 

A Thing gets associated with a Group when you associate Thing's certificate
with the Group. This is defined in the AWS IoT cloud interface. The Thing then
discovers the associated gateway through AWS IoT cloud service as shown in the
figure below. Information such as local IP address and Root CA is exchanged
between the Thing and AWS IoT service. The Thing can now contact the gateway
through the local IP address and after successful connection share data between
gateway and the AWS IoT service through gateway's MQTT broker.

![](./figs/l3_gg_interact.png)

The security setup is given in the figure below. 

(_A_) - AWS Greengrass and AWS IoT needs to have enough permission to access
each others services.

(_B_) - An X.509 certificate used to authenticate a specific gateway

(_C_) - An X.509 certificate used to authenticate a specific Thing

(_D_) - Group role. Similar to (_A_), but this time the Greengrass core running
on the gateway needs specific permissions to access other AWS cloud
infrastructure.

(_E_) - Group CA certificate. A root certificate used by Things to validate the
certificate presented by a gateway during TLS mutual authentication.

![](./figs/l3_gg_security.png)

The mutual authentication happens as follows: 

1. The Thing contacts the AWS IoT service via HTTPS. Mutual authentication
   between the Thing and the server occurs and authenticity of the AWS IoT server
   is established.
2. Similarly, the gateway also connects to the AWS IoT service via HTTPS. TLS
   mutual authentication is used to verify both AWS server and the gateway.
3. The AWS IoT server then sends all the groups a Thing is associated to, its
   respective IP address and their respective group CAs. Because we trust AWS
   IoT server (due to mutual authentication carried out in the previous step),
   the validity of group CA is also verified.
4. The Thing then connects to an interested gateway through the use of group
   CA. TLS mutual authentication happens between gateway and Thing now, and can
   be through MQTT. The local IP address is used to locate the MQTT broker, unlike
   the first and second step where an _endpoint_ was used.
5. Once connection is established, data now flows through the gateway to the
   server using the MQTT protocol.

## References

1. [AWS Greengrass Documentation](https://docs.aws.amazon.com/greengrass/latest/developerguide/what-is-gg.html)
2. [AWS Greengrass SDK Documentation](https://s3.amazonaws.com/aws-iot-device-sdk-python-docs/sphinx/html/index.html#module-AWSIoTPythonSDK.core.greengrass.discovery.providers)
3. [AWS IoT Python SDK and samples](https://github.com/aws/aws-iot-device-sdk-python-v2)
