# System Architecture of IoT - Lab 1 report

**Description:** _Get a working understanding of the MQTT protocol. Setup a rudimentary publisher and subscriber using a openly available broker._

**Group members:** Sebastian Horsch, Anna Vítová

**Lab dates:** 2nd and 15th October 2024

## Introduction (3 p)

Provide a short description of the setup (hardware and software) and how the different components interact. Explain the steps you performed in the lab, e.g., using screenshots and photos.

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
