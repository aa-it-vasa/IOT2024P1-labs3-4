# System Architecture of IoT - Lab 1 report

**Description:** _Get a working understanding of the MQTT protocol. Setup a rudimentary publisher and subscriber using a openly available broker._

**Group members:** Sebastian Horsch, Anna Vítová

**Lab dates:** 2nd and 15th October 2024

## Introduction (3 p)

Provide a short description of the setup (hardware and software) and how the different components interact. Explain the steps you performed in the lab, e.g., using screenshots and photos.

The lab was about setting up an Iot device with the Aduino MKR WiFi1010 board. A thermistor was used to read the room temperature and this info is then send to an MQTT broker over Wifi. 

Hardware Setup:
- Arduino MKR WiFi1010: The main microcontroller, providing WiFi connectivity and processing capabilities
- Thermistor (MCP9700/LMT87): Measures room temperature, connected to the analog pin of the Arduino for analog-to-digital conversion
- Breadboard and Jumper Wires: Used for prototyping, with connections for Vcc and GND.
- USB Cable: Provides power to the Arduino and enables programming.
- 
Software Setup:
- Arduino IDE: Programmed the Arduino board with the necessary code for temperature sensing and MQTT communication. Libraries like ArduinoMqttClient and WiFiNINA   
               were installed for MQTT and WiFi functionality.
- MQTT Broker (HiveMQ): Acts as a server to which the Arduino publishes temperature data. It also receives commands from the user to control onboard LEDs based on 
MQTT messages.
- MQTT Client (HiveMQ WebSocket client): Subscribes to topics published by the Arduino (recives the temperature and sends commands to the Arduino)

Interaction of Components:

- The Arduino reads temperature from the thermistor through its analog input pin. Then it converts the data and publishes it to the MQTT broker via WiFi. The MQTT broker stores the data and then all clients that subscribed to it will recive them as an update. The Arduino is also set up to receive MQTT messages on a specific topic, for example that it has an specific reponse to commands like ON, OFF or TEMP.

Steps performed in the Lab: 

1. Installed the Arduino IDE, added necessary libraries, and configured the board type and serial port
2. Assembled the MKR WiFi1010 board on the breadboard, then connected the thermistor according to the circuit diagram. After this the breadboard was contnected to the computer using USB
3. Configuration of the MQTT broker's address and topics and ensuring that the topic names are unique
4. Upload of the test sketches and checking if the MQTT broker is publishing and the subscription works. Then we checked the temperature readings
5. Analyse of the existing code
6. Implemention of the new logic for the commands ON, OFF and TEMP
7. Testing of the implementation and finalization of the code 

## How does the `getTemp` function work (1 p)
First in the function five variables are initialized. After that a loop is run through ten times. Each time first the method analogRead() reads the sensors raw voltage, which is then multiplied and divided by specific numbers. After that the temperature is calculated and save in the variable. Then the the program waits for 0.1 seconds and adds the temperature up into the variable ten_sample_sum. When the loop is finished the return parameter gives the added temperature divided by the amount of loop runs back.  

## What is the value returned by `mqttClient.messageQoS()`? What does it mean? (1 p)
The method mqttClient.messageQoS() returns an int value that is either 0, 1 or 2. 
0 = At most once (The message is delivered once, without acknowledgment / Delivery is not guaranteed, and the message could be lost)
1 = At least once (The message is delivered at least once, but it may be delivered many times if acknowledgment is lost)
2 = Exactly once (The message is guaranteed to be delivered exactly once)

## What is the value returned by `mqttClient.messageRetain()`? What does it mean? (1 p)
The value returned by this method is a boolean. So it is either true or false, when it is true it means that the message was stored by the broker as the "last known good" message for this topic. Furthermore, it is sent because of the subscription to that topic after it was retained. If it's false the message was sent in real time and was not retained by the broker.

## Command and response (8 p)

Implement the functionality required in the lab instructions and provide the code according to the instructions. Explain in your report how your code works and provide the block diagram requested.

![image](https://github.com/user-attachments/assets/edf2cb5e-2d16-4df1-a709-5b14dfe22770)

The changes made in the code begin with the declaration of one constant and two variables. The constant stores the name of the build in LED that should be switched on or off depending the command. The variable ledState is used to store the current status of the led that is either 0 or 1. The other one is used to store the command which was given to the aduino. Next in the code there is an while loop that reads every letter of the input command time after time. After the command is fully read into the command variable there is a switch chase that differentiates between ON, OFF and TEMP. If ON is the input command the status of the LED is set to 1 to turn it on. If the command was OFF, the status is set to 0. If the command is TEMP the temperature is read by the function getTemp() then it is displayed on the mqttClient. At the end of the adjustments there is the function digitalWrite() that sends the new or the default status to the LED of the Arduino board. Finally, there are two empty lines printed out and the the method onMqttMessage() ends. 

