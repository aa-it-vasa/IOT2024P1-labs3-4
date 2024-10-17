# System Architecture of IoT - Lab 3 report

**Description:** _Establish a MQTT connection that is safe from (unintended) snooping and is private using a gateway._

**Group members:** _Fill group member names here_

**Lab dates:** _Fill out the dates you did your labs here_

## Work description (4 p)

Provide a short description of the setup (hardware and software) and how the different components interact. Explain the steps you performed in the lab, e.g., using screenshots and photos including the received messages on the AWS dashboard.

Explain how the Thing connects to your gateway, including TLS mutual authentication. Use relevant diagrams. 

## Measure the latency of establishing the connection to the gateway (4 p)

Modify `pubsub.py` to measure the latency of establishing the connection to the gateway. Compare the results to pinging the Greengrass gateway from your laptop. 

## Simulate a malicious subscriber (2 p)

Describe what happens when you subscribe/publish from a Thing without a valid certificate.