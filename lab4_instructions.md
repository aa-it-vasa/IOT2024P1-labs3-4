# Lab 4 - Edge computing using Greengrass

## Overview

In the previous lab, we organised Things using gateways and ensured that there were no unintended cross-talk between devices of different groups. In this lab, we will extend the gateway's usefulness to perform close-to-sensor computation. Close-to-sensor computation, or edge computation, is, for example, used to manage high-bandwidth data emerging from the sensors. By processing the data at the edge, either completely or partially, we reduce the communication burden on the network infrastructure. Also, for a latency-critical application a gateway reduces the response time between events and manages the risk related to network outages. Edge computing can also be used when we are considering sensitive data that we do not want to send over the internet but rather process locally.

There are several concerns with edge computation. Unlike the situation in the lab, you can not always rely on the local network to SSH into the device. So a remote deployment and management scheme is often necessary. Also, the logic may vary from one gateway to another. So we need a solution that scales and handles diverse computational needs. 

Webservers on gateways and a version control should meet most of the concerns expressed above. Coupled with a dashboard to manage all the gateways, this solution would scale as more gateways are added. One option is to use the services included in AWS.
One such solution is AWS Lambda. As a developer, you mwould write the logic using one of the supported languages and deploy them on the gateway along with the device certificates, policies etc. 

The programming model of Lambda is event driven, similar to the subscription mechanism of MQTT. Every time a message formatted as JSON document is sent to the lambda either through MQTT or HTTPS, an event is triggered and the function associated with the event is spawned in a virtual container. This on-demand event driven function runs without the need of setting up a server. A side-effect is that in the case of multiple events, multiple independent Lambdas are invoked and managed. AWS calls this pattern as "event-driven serverless computing". If you are interested in running a single Lambda for all the events, then Lambda can also be configured to run in this fashion. 

AWS Lambda is suitable for running small on-demand applications that are reactive in nature. For a heavy duty logic that process multiple streams from different sources, you may want to forward the data to the cloud and process them there.

In this lab, we will not directly utilize AWS functionality for deploying Lambda functions, however, everything we do here could be done utilizing the web based functionality as well. Instead we will develop the Python code locally and deploy it to the local Raspberry PI using the Greengrass CLI. 

Here we will configure a core device to interact with local IoT devices, called client devices, that connect to the core device over MQTT. This is done by configuring AWS IoT things to use cloud discovery to connect to the core device as client devices. When you configure cloud discovery, a client device can send a request to the AWS IoT Greengrass cloud service to discover core devices. The response from AWS IoT Greengrass includes connectivity information and certificates for the core devices that you configure the client device to discover. Then, the client device can use this information to connect to an available core device where it can communicate over MQTT.

## Verify that Greengrass is working

Make sure Greengrass core is running on your Raspberry PI. For example check the AWS Console under _AWS IoT > Manage > Greengrass devices > Core devices_. It the status is not `Healthy` and the status reported is less than the amount of time since you rebooted the device, then check the status of the Greengrass service using:
```bash
rpi> sudo systemctl status greengrass
```
If it is stopped start it with 
```bash
rpi> sudo systemctl start greengrass
```
You can also check the Greengrass log files on the PI with 
```bash
rpi> sudo cat /greengrass/v2/logs/greengrass.log
```

If you cannot get it working, check with the lab assistant or recreate the Greengrass core device as you did in Lab 3.

## Enable client device support

For a client device to use cloud discovery to connect to a core device, you must associate the devices. When you associate a client device to a core device, you enable that client device to retrieve the core device's IP addresses and certificates to use to connect.

1. Navigate to the AWS IoT Greengrass console.
2. In the left navigation menu, choose _Core devices_.
3. On the _Core devices_ page, choose the core device where you want to enable client device support. This is the same thing you created in Lab 3, i.e., `simthing_GROUPNAME`.
4. On the core device details page, choose the _Client devices_ tab.
5. On the _Client devices_ tab, choose _Configure cloud discovery_. 

   The Configure core device discovery page opens. On this page, you can associate client devices to a core device and deploy client device components. This page selects the core device for you in Step 1: Select target core devices.

6. In _Step 2: Associate client devices_, associate the client device's AWS IoT thing to the core device. This enables the client device to use cloud discovery to retrieve the core device's connectivity information and certificates. Do the following:

    1. Choose _Associate client devices_.

    2. In the _Associate client devices with core device_ modal, enter the name of the AWS IoT thing to associate.

    3. Choose _Add_.

    4. Choose _Associate_.

7. For the _aws.greengrass.clientdevices.Auth_ component, choose _Edit configuration_. 

    In the _Edit configuration_ modal for the client device auth component, configure an authorization policy that allows client devices to publish and subscribe to the MQTT broker on the core device. Do the following:

8. Under _Configuration_, in the _Configuration to merge_ code block, enter the following configuration, which contains a client device authorization policy. Each device group authorization policy specifies a set of actions and the resources on which a client device can perform those actions. Remember to update the name of your Thing under _thingName_.
   ```json
   {
   "deviceGroups": {
      "formatVersion": "2021-03-05",
      "definitions": {
         "MyDeviceGroup": {
         "selectionRule": "thingName: simthing_GROUPNAME",
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

9. Choose _Confirm_.

10. For the _aws.greengrass.clientdevices.mqtt.Bridge_ component, choose _Edit configuration_.

    In the _Edit configuration_ modal for the MQTT bridge component, configure a topic mapping that relays MQTT messages from client devices to AWS IoT Core. Do the following:

    1. Under _Configuration_, in the _Configuration to merge_ code block, enter the following configuration. This configuration specifies to relay MQTT messages on the `GROUPNAME/+/hello/world` topic filter from client devices to the AWS IoT Core cloud service:

         ```json
         { 
            "mqttTopicMapping": {
               "HelloWorldIotCoreMapping": {
                  "topic": "GROUPNAME/+/hello/world",
                  "source": "LocalMqtt",
                  "target": "IotCore"
               }
            }
         }
         ```

         For more information, see [MQTT bridge component configuration](https://docs.aws.amazon.com/greengrass/v2/developerguide/mqtt-bridge-component.html#mqtt-bridge-component-configuration).
      
      2. Choose _Confirm_.

11. Choose _Review and deploy_ to review the deployment that this page creates for you.

12. On the _Review_ page, choose _Deploy_ to start the deployment to the core device.

13. Verify that the deployment succeeds by checking that the _Status_ in the _Deployments_ list change to _Completed_ (this might take some time). 

## Connect the client device

On the terminal on your local machine, navigate to the samples subfolder of the _aws-iot-device-sdk-python-v2_ folder used in Lab 3.

Run the sample Greengrass discovery application. This application expects arguments that specify the client device thing name, the MQTT topic and message to use, and the certificates that authenticate and secure the connection. The following example sends a Hello World message to the GROUPNAME/MyClientDevice1/hello/world topic.

```bash
local> python3 basic_discovery.py \
  --thing_name simthing_GROUPNAME \
  --topic 'GROUPNAME/MyClientDevice1/hello/world' \
  --message 'Hello World!' \
  --ca_file root_ca.pem \
  --cert publisher_sim.pem.crt \
  --key publisher_sim-private.pem.crt \
  --region eu-central-1 \
  --verbosity Warn
```
The discovery sample application sends the message 10 times and disconnects. It also subscribes to the same topic where it publishes messages. If the output indicates that the application received MQTT messages on the topic, the client device can successfully communicate with the core device.

Verify that the MQTT bridge relays the messages from the client device to AWS IoT Core by using the MQTT test client in the AWS IoT Core console to subscribe to an MQTT topic filter. Do the following:
1. Navigate to the _AWS IoT console_.
2. In the left navigation menu, under _Test_, choose _MQTT test client_.
3. On the _Subscribe to a topic_ tab, for _Topic filter_, enter `GROUPNAME/+/hello/world` to subscribe to client device messages from the core device.
4. Choose _Subscribe_.
5. Run the discovery application on the client device again.

The MQTT test client displays the messages that you send from the client device on topics that match this topic filter.

## Develop a component that communicates with client devices

You can develop Greengrass components that communicate with client devices. Components use interprocess communication (IPC) and the local publish/subscribe interface to communicate on a core device. To interact with client devices, configure the MQTT bridge component to relay messages between client devices and the local publish/subscribe interface.

In this section, you update the MQTT bridge component to relay messages from client devices to the local publish/subscribe interface. Then, you develop a component that subscribes to these messages and prints the messages when it receives them.

### Setup AWS IoT Core

Revise the deployment to the core device and configure the MQTT bridge component to relay messages from client devices to local publish/subscribe. Do the following:

1. In the left navigation menu in the _AWS IoT Greengrass console_, choose _Core devices_.

2. On the _Core devices_ page, choose the core device that you are using for this tutorial.

3. On the core device details page, choose the _Client devices_ tab.

4. On the _Client devices_ tab, choose _Configure cloud discovery_.

5. The _Configure core device discovery_ page opens. On this page, you can change or configure which client device components deploy to the core device.

6. In _Step 3_, for the _aws.greengrass.clientdevices.mqtt.Bridge_ component, choose _Edit configuration_.

7. In the _Edit configuration_ modal for the MQTT bridge component, configure a topic mapping that relays MQTT messages from client devices to the local publish/subscribe interface. 

   Under _Configuration_, in the _Configuration to merge_ code block, enter the following configuration. This configuration specifies to relay MQTT messages on topics that match the `GROUPNAME/+/hello/world` topic filter from client devices to the AWS IoT Core cloud service and the local Greengrass publish/subscribe broker.

   ```json
   {
      "mqttTopicMapping": {
         "HelloWorldIotCoreMapping": {
            "topic": "GROUPNAME/+/hello/world",
            "source": "LocalMqtt",
            "target": "IotCore"
         },
         "HelloWorldPubsubMapping": {
            "topic": "GROUPNAME/+/hello/world",
            "source": "LocalMqtt",
            "target": "Pubsub"
         }
      }
   }
   ```

8. Choose _Confirm_ and then _Review and deploy_ to review the deployment that this page creates for you.

9. On the _Review_ page, choose _Deploy_ to start the deployment to the core device.

10. To verify that the deployment succeeds, check the status of the deployment, and check the logs on the core device. 

### Setup the local component

We will now develop and deploy a Greengrass component that subscribes to Hello World messages from client devices. Do the following:

1. Create folders for recipes and artifacts on the Raspberry PI, i.e., the core device.

    1. Create folders for recipes and artifacts on the core device.
       ```bash
       rpi> mkdir component
       rpi> cd component
       rpi> mkdir recipes
       rpi> mkdir -p artifacts/MyHelloWorldSubscriber/1.0.0
       ```
       You must use the format `artifacts/componentName/componentVersion/` for the artifact folder path. Include the component name and version that you specify in the recipe.

   2. Use a text editor to create a component recipe with the following contents. This recipe specifies to install the AWS IoT Device SDK v2 for Python and run a script that subscribes to the topic and prints messages. For example, on a Linux-based system, you can run the following command to use GNU nano to create the file.
       ```bash
       rpi> nano recipes/MyHelloWorldSubscriber-1.0.0.json
       ```
      Copy the following recipe into the file. Make sure you understand what this policy does.
      ```json
      {
      "RecipeFormatVersion": "2020-01-25",
      "ComponentName": "MyHelloWorldSubscriber",
      "ComponentVersion": "1.0.0",
      "ComponentDescription": "A component that subscribes to Hello World messages from client devices.",
      "ComponentPublisher": "GROUPNAME",
      "ComponentConfiguration": {
         "DefaultConfiguration": {
            "accessControl": {
            "aws.greengrass.ipc.pubsub": {
               "MyHelloWorldSubscriber:pubsub:1": {
                  "policyDescription": "Allows access to subscribe to all topics.",
                  "operations": [
                  "aws.greengrass#SubscribeToTopic"
                  ],
                  "resources": [
                  "*"
                  ]
               }
            }
            }
         }
      },
      "Manifests": [
         {
            "Platform": {
            "os": "linux"
            },
            "Lifecycle": {
            "install": "python3 -m pip install --user awsiotsdk",
            "run": "python3 -u {artifacts:path}/hello_world_subscriber.py"
            }
         }
      ]
      }
      ```

   3. Use a text editor to create a Python script artifact named `hello_world_subscriber.py` with the contents specified below. This application uses the publish/subscribe IPC service to subscribe to the `GROUPNAME/+/hello/world` topic and print messages that it receives.
       ```bash
       rpi> nano artifacts/MyHelloWorldSubscriber/1.0.0/hello_world_subscriber.py
       ```
   
   4. Copy the following Python code into the file:
       ```python
      import awsiot.greengrasscoreipc
      from awsiot.greengrasscoreipc.model import (PublishToTopicRequest, PublishMessage, BinaryMessage, UnauthorizedError)

      local_topic = 'GROUPNAME/+/localtocloud'
      cloud_topic = 'GROUPNAME/+/greengrasstocloud'
      client_topic = 'GROUPNAME/+/greengrasstolocal'
      command_topic = 'GROUPNAME/+/cloudtogreengrass'

      my_counter = 0
      my_platform = platform.platform()

      def on_local_topic(event):
         try:
            global my_counter
            my_counter = my_counter+1

            # The message received through MQTT
            recv_message = str(event.binary_message.message, 'utf-8')

            # The platform identifier
            my_platform = platform.platform()

            # Publish and send the message
            message ="Sent from Greengrass Core running on platform {}. Invocation Count {}. Activation message {} ".format(my_platform, my_counter, recv_message)
            binary_message = BinaryMessage(message=bytes(message, 'utf-8'))
            publish_message = PublishMessage(binary_message=binary_message)
            ipc_client = GreengrassCoreIPCClientV2()
            ipc_client.publish_to_topic(topic=cloud_topic, publish_message=publish_message)
            print('Successfully published to topic: ' + cloud_topic)
            ipc_client.close()

         except:
            traceback.print_exc()

      def on_command_topic(event):
         try:
            recv_message = str(event.binary_message.message, 'utf-8')
            message ="Cloud message received: {} ".format(recv_message)
            binary_message = BinaryMessage(message=bytes(pub_message, 'utf-8'))
            publish_message = PublishMessage(binary_message=binary_message)
            ipc_client = GreengrassCoreIPCClientV2()
            ipc_client.publish_to_topic(topic=client_topic, publish_message=publish_message)
            print('Successfully published to topic: ' + client_topic)
            ipc_client.close()

         except:
            traceback.print_exc()

      try:
         ipc_client = GreengrassCoreIPCClientV2()

         # SubscribeToTopic returns a tuple with the response and the operation.
         _, operation = ipc_client.subscribe_to_topic(topic=local_topic, on_stream_event=on_local_topic)
         print('Successfully subscribed to topic: %s' % local_topic)

         _, operation = ipc_client.subscribe_to_topic(topic=command_topic, on_stream_event=on_command_topic)
         print('Successfully subscribed to command topic: %s' % command_topic)

         # Keep the main thread alive, or the process will exit.
         try:
            while True:
                  time.sleep(10)
         except InterruptedError:
            print('Subscribe interrupted.')

         operation.close()
      except Exception:
         print('Exception occurred when using IPC.', file=sys.stderr)
         traceback.print_exc()
         exit(1)
       ```
       This component uses the IPC client V2 in the [AWS IoT Device SDK v2](https://github.com/aws/aws-iot-device-sdk-python-v2) for Python to communicate with the AWS IoT Greengrass Core software.

   5. Use the Greengrass CLI to deploy the component:
      ```bash
      rpi> sudo /greengrass/v2/bin/greengrass-cli deployment create \
      --recipeDir recipes \
      --artifactDir artifacts \
      --merge "MyHelloWorldSubscriber=1.0.0"
      ```

   6. View the component logs to verify that the component installs successfully and subscribes to the topic:
      ```bash
      rpi> sudo tail -f /greengrass/v2/logs/MyHelloWorldSubscriber.log
      ```
      You can keep the log feed open to verify that the core device receives messages.

   7. On the client device, run the sample Greengrass discovery application again to send messages to the core device:
      ```bash
      python3 basic_discovery.py \
      --thing_name simthing_GROUPNAME \
      --topic 'GROUPNAME/MyClientDevice1/hello/world' \
      --message 'Hello World!' \
      --ca_file root_ca.pem \
      --cert publisher_sim.pem.crt \
      --key publisher_sim-private.pem.crt \
      --region eu-central-1\
      --verbosity Warn
      ```

   8. View the component logs again to verify that the component receives and prints the messages from the client device.
      ```bash
      rpi> sudo tail -f /greengrass/v2/logs/MyHelloWorldSubscriber.log
      ```


## To do
 
1. Understand the `greengrassHelloWorld.py` code that was packaged into the Lambda function above.
2. Connect the simulated device that you created in the last lab, with the Lambda function. The setup you are looking at is given below. You can use `pubsub.py` to publish messages from your device (see Lab 3).
    1. Device -> Lambda. Device sends message on topic `saiot/GROUPNAME/localtolambda`. Tip: you will need to update some of the settings for the component and deployment above. 
    2. Lambda -> IoT Cloud. Lambda parses the message sent by the device and forwards it to IoT Cloud on topic `saiot/GROUPNAME/localtocloud`.
3. In your report describe the achieved architecture and behavior of your application. Use figures to illustrate your description. (5p)
4. In your report describe the behavior differences between a long-lived Lambda function and an on-demand Lambda function deployed on a gateway. Illustrate your response taking a simple application example and provide the corresponding sequence (UML) diagrams for each. (5p)