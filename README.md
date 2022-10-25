# Unreal Engine Project "Heartbeat"

* Author: Copyright 2022 Roland Bruggmann aka brugr9
* Profile on UE Marketplace: [https://www.unrealengine.com/marketplace/profile/brugr9](https://www.unrealengine.com/marketplace/profile/brugr9)
* Profile on Epic Developer Community: [https://dev.epicgames.com/community/profile/PQBq/brugr9](https://dev.epicgames.com/community/profile/PQBq/brugr9)

---

Proof of Concept for Receiving MQTT Data using UE Plugin "Integration Tool"

## Description

Use Case: Polar&reg; H10 to Unreal&reg; Engine Integration

* Index Terms: Sports Performance, Physiological Measuring, Heart Rate HR, Electrocardiogram ECG, Accelerometer ACC, Integration, Messaging, Machine-to-Machine M2M, Internet of Things IoT
* Technology: Polar&reg; H10 HR Sensor with Chest Strap, Bluetooth&reg;, Polar Sensor Logger PSL, MQTT, NanoMQ&trade;, NNG&trade;, Unreal&reg; Engine

---

<div style='page-break-after: always'></div>

## Table of Contents

<!-- Start Document Outline -->

* [1. Concept](#1-concept)
* [2. Unreal Engine](#2-unreal-engine)
  * [2.1. Install Plugin](#21-install-plugin)
  * [2.2. NNG Subscription](#22-nng-subscription)
* [3. NanoMQ](#3-nanomq)
  * [3.1. Configure Firewall](#31-configure-firewall)
  * [3.2. Setup NanoMQ](#32-setup-nanomq)
  * [3.3. Forward Messages](#33-forward-messages)
* [4. Polar Sensor Logger](#4-polar-sensor-logger)
  * [4.1. Setup Android Debug Bridge](#41-setup-android-debug-bridge)
  * [4.2. Setup Polar Sensor Logger](#42-setup-polar-sensor-logger)
* [5. Monitor Messages](#5-monitor-messages)
* [6. Data Visualisation](#6-data-visualisation)
* [A. Attribution](#a-attribution)
* [B. References](#b-references)
* [C. Readings](#c-readings)
* [D. Citation](#d-citation)

<!-- End Document Outline -->

<div style='page-break-after: always'></div>

## 1. Concept

The concept for receiving MQTT data in the Unreal Engine is based on a general data flow: A data producer sends data via MQTT to a broker which forwards the data via NNG/SP to a NNG-Socket.

*Listing 1.1.: General Data Flow*
> **Data Producer** &mdash;(*MQTT*)&rarr; **Broker** &mdash;(*NNG/SP*)&rarr; **NNG-Socket**

We use system components as follows (for the specific data flow see Listing 1.2.):

* Data Producer:
  * Polar&reg; H10 Heart Rate Sensor with Chest Strap
  * Android&trade; App **Polar Sensor Logger**, with Accelerometer (ACC) Sampling Rate of $25 Hz$
* Broker: EMQ&trade;'s **NanoMQ&trade;** MQTT Broker as a Proxy
* NNG-Socket: **NNG SUB-Socket** from Unreal&reg; Engine Plugin "Integration Tool"

*Listing 1.2.: Specific Data Flow*
> Polar H10 &ndash;(*Polar BLE SDK*)&rarr; **Polar Sensor Logger** &ndash;(*MQTT*)&rarr; **NanoMQ**  &ndash;(*NNG/SP*)&rarr; **NNG SUB-Socket** from UE Plugin "Integration Tool"

The following shows the setup in reverse order of the data flow: Unreal Engine, NanoMQ and Polar Sensor Logger. Finally we monitor the messages using Wireshark&trade; and visualise the data in the Unreal Editor.

<div style='page-break-after: always'></div>

## 2. Unreal Engine

### 2.1. Install Plugin

1. Purchase UE Plugin "Integration Tool"
2. Open a new UE Project named, e.g., "Heartbeat"
3. Activate Plugin "Integration Tool"
4. Restart Project "Heartbeat"

![ScreenshotPlugin](https://user-images.githubusercontent.com/1733987/183462214-39703236-7147-4c56-bdab-faaaad848877.jpg)
*Fig. 2.1.: Unreal Engine Editor with Plugin "Integration Tool"*

### 2.2. NNG Subscription

That's how the map `Map_PSL_Demo` and the actor `BP_PSL_Demo` were created:

1. From `Engine > Plugins > Integration Tool Content > Demo > Blueprints` copy `BP_CubeGreen` to the project's Content folder, rename the Blueprint to `BP_PSL_Demo` and `Edit ...` the same:
   * 1. Change Actor-Component `Cube > Materials > Element 0` to `WidgetMaterial_X`
   * 2. Change Actor-Component `TextRender > Text > Text Render Color` to red
   * 3. Delete Actor-Component `Subscriber_C` and its related nodes in the `Event Graph`
   * 4. Rename Actor-Component `Subscriber_Y` to `Subscriber`, set variable "Topic" to `psl` and ensure to have "Starts With" checked

![NNGSubscription-BP_PSL_Demo](https://user-images.githubusercontent.com/1733987/184128474-6b491449-3991-4938-8fb5-a0aff018bb27.jpg)
*Fig. 2.2.: BP_PSL_Demo Actor-Components and Graph Editor*

2. From `Engine > Plugins > Integration Tool Content > Demo > Maps` copy Level `Map_PubSub_Demo` to the project's Content folder. Rename the Level to `Map_PSL_Demo` and `Edit ...` the same:
   * 1. In the Level Blueprint ...
     * 1. Delete Variable `PUB-Socket` and its related nodes in the `Event Graph`
     * 2. OnOpen (SUB-Socket): Replace function node `Connect` with function node `Bind`
     * 3. Loop calling the function `Receive` with a little more than double the sampling rate (Nyquist theorem): $2.1 \times 25 Hz = 52.5 Hz = 1 / 52.5  s = 0.019 s$
   * 2. In the Outliner ...
     * 1. Delete the Bluprint instances of `BP_CubeBlue`, `BP_CubeYellow` and `BP_CubeGreen`
     * 2. Delete the NNG PUB-Socket Actor instance `PubSocketActor1`
     * 3. Add an instance of Blueprint `BP_PSL_Demo` and in its parameter 'Socket' drop-down choose the NNG SUB-Socket Actor instance `SubSocketActor1`

Reminder: The NNG SUB-Socket Actor instance `SubSocketActor1` transport is set to `tcp://127.0.0.1:5555` (default values).

![UEProjectHeartbeat-NNGSubscription-Map_PSL_Demo_-_LevelBP](https://user-images.githubusercontent.com/1733987/190147063-b392f2a6-0bac-4eb1-8776-949f05b945bd.jpg)
*Fig. 2.3.: Map_PSL_Demo Level Blueprint*

![NNGSubscription-Map_PSL_Demo](https://user-images.githubusercontent.com/1733987/184128515-2d5a377e-3c6d-45e7-9245-ff8b9d17acef.jpg)
*Fig. 2.4.: Map_PSL_Demo Outliner*

<div style='page-break-after: always'></div>

## 3. NanoMQ

### 3.1. Configure Firewall

Allow TCP port 1883:

```PowerShell
New-NetFirewallRule -DisplayName "ALLOW TCP PORT 1883" -Direction inbound -Profile Any -Action Allow -LocalPort 1883 -Protocol TCP
```

Allow TCP port 5555:

```PowerShell
New-NetFirewallRule -DisplayName "ALLOW TCP PORT 5555" -Direction inbound -Profile Any -Action Allow -LocalPort 5555 -Protocol TCP
```

### 3.2. Setup NanoMQ

Using, e.g., a PowerShell pull the NanoMQ docker container from EMQ (see [https://www.emqx.com/en/try?product=nanomq](https://www.emqx.com/en/try?product=nanomq))

```PowerShell
docker pull emqx/nanomq:0.12.2
```

and run the same using:

```PowerShell
docker run -d --name nanomq -p 1883:1883 -p 5555:5555 emqx/nanomq:0.12.2
```

### 3.3. Forward Messages

Open the same in a terminal, e.g., by using the "Open in Terminal" button in Docker Desktop:

![NanoMQDocker Container](https://user-images.githubusercontent.com/1733987/189970626-4b1d9df6-e8b4-476d-99d6-0fdcf33ef122.jpg)
*Fig. 3.1.: Docker Desktop with NanoMQ Container and Button "Open in Terminal"*

In the terminal type `nanomq_cli` and check for having `nngproxy` available:

*Listing 3.1.: Output of nanomq_cli*
```bash
# nanomq_cli
nanomq_cli { pub | sub | conn | nngproxy | nngcat } [--help]

available tools:
   * pub
   * sub
   * conn
   * nngproxy
   * nngcat

Copyright 2022 EMQ Edge Computing Team
```

Forward messages received on endpoint `mqtt://localhost:1883` with any topic to `tcp://localhost:5555` using NanoMQ NNG-proxy with Quality of Service (QoS) level 0&mdash;which is a guarantee of delivery for a specific message "At most once" (cp. [https://github.com/emqx/nanomq#quick-start](https://github.com/emqx/nanomq#quick-start), section *NanoMQ nng message proxy*):

```bash
nanomq_cli nngproxy pub0 --mqtt_url "mqtt://localhost:1883" --dial "tcp://localhost:5555" --topic '#' --qos 0
nanomq_cli nngcat --sub --listen="tcp://localhost:5555" -v  --quoted
```

<div style='page-break-after: always'></div>

## 4. Polar Sensor Logger

### 4.1. Setup Android Debug Bridge

(cp. Skanda Hazarika: *How to Install ADB on Windows, macOS, and Linux*. July 28, 2021. In: XDA Developers. URL: [https://www.xda-developers.com/install-adb-windows-macos-linux](https://www.xda-developers.com/install-adb-windows-macos-linux))

On the Android enable USB Debugging mode:

1. Launch the `Settings` application.
2. Tap the `About Phone` option (generally found near the bottom of the list).
3. Then tap the `Build Number` option _7 times_ to enable _Developer Mode_. You will see a toast message when it is done.
4. Now go back to the main `Settings` screen and you should see a new `Developer Options` menu you can access.
5. Go in there and enable the `USB Debugging` mode option.
6. Allow `USB Debugging prompt` on Android

On the PC:

Ensure you have setup _Android Debug Bridge_, e.g., using 

```ps
choco install adb
```

Using an _administrative PowerShell_ start the Android Debug Bridge (cp. Tushar Sadhwani: *Connecting Android Apps to localhost, Simplified*. 17 Apr 2021. In: DEV Community, URL: [https://dev.to/tusharsadhwani/connecting-android-apps-to-localhost-simplified-57lm](https://dev.to/tusharsadhwani/connecting-android-apps-to-localhost-simplified-57lm)):

```ps
adb reverse tcp:1883 tcp:1883
```

### 4.2. Setup Polar Sensor Logger

1. On the Android device install the "Polar Sensor Logger" (PSL) App
2. Mount the Polar H10 sensor on the chest strap and wear the same.
4. Connect the Android device by USB to PC, prompt "Allow USB Debugging" > OK
5. On the Android device ...
   * 1. Activate Bluetooth
   * 2. Activate Location Service
   * 3. Open "Polar Sensor Logger" App
     * 1. Under `SDK data select`: Check `ACC` solely
     * 2. Under `Settings`: Check `MQTT` solely, in the pop-up configure `MQTT broker address` with IP `127.0.0.1`, hit `OK`
     * 3. Hit `Seek Sensor`, select listed sensor `Polar H10 12345678` (ID will differ), hit `OK`
     * 4. In the pop-up select logging ACC data with a sampling rate $25 Hz$ (default), hit `OK`

![Screenshot_20220808-190056](https://user-images.githubusercontent.com/1733987/183487653-f9ea33fc-2299-489d-854f-22f751cfa2df.png) | ![Screenshot_20220808-190046](https://user-images.githubusercontent.com/1733987/183487651-5ff9136c-0cc4-4938-86b6-3bd400f2781d.png) | ![Screenshot_20220808-191014](https://user-images.githubusercontent.com/1733987/183488379-4e3fe7b6-465e-4126-8102-7804c34bd58e.png) | ![Screenshot_20220811-135008](https://user-images.githubusercontent.com/1733987/184129307-3c32ab79-c81d-4498-81c5-eb7d5217d8a3.png) | ![Screenshot_20220906-181704](https://user-images.githubusercontent.com/1733987/188705729-ffe28e57-070a-4102-85b3-4fdaa3814627.png)
:-------------------------:|:-------------------------:|:-------------------------:|:-------------------------:|:-------------------------:
*Fig. 4.1.: PSL, Main Tab* | *Fig. 4.2.: PSL, Dialogue "MQTT Settings"* | *Fig. 4.3.: PSL, Dialogue "Seek Sensor"* | *Fig. 4.4.: PSL, Dialogue "ACC Settings"* | *Fig. 4.5.: PSL, Status Connected*

<div style='page-break-after: always'></div>

## 5. Monitor Messages

Use Wireshark&reg; to monitor PSL for sending its messages over port 1883 and NanoMQ for forwarding the messages to port 5555 (see also *Abhinaya Balaji: Dissecting MQTT using Wireshark*. In: Blog Post, July 6, 2017, Catchpoint Systems, Inc. URL: [https://www.catchpoint.com/blog/wireshark-mqtt](https://www.catchpoint.com/blog/wireshark-mqtt)).

*Listing 5.1.: Wireshar Filter TCP Port 1883*
```
tcp.port == 1883
```

TODO: PrintScreens
![Screenshot-Wireshark-1883-1]()
![Screenshot-Wireshark-1883-2]()
*Fig. 5.1.: Wireshark Dissecting Port 1883*

*Listing 5.2.: Wireshar Filter TCP Port 5555*
```
tcp.port == 5555
```

TODO: PrintScreens
![Screenshot-Wireshark-5555-1]()
![Screenshot-Wireshark-5555-2]()
*Fig. 5.2.: Wireshark Dissecting Port 5555*

*Listing 5.3.: Wireshar Filter TCP Ports 1883 and 5555*
```
tcp.port in {1883 5555}
```

<div style='page-break-after: always'></div>

## 6. Data Visualisation

In Unreal Editor with Level `Map_PSL_Demo` open, click the `Play` button &#9658; in the level editor to start Play-in-Editor PIE. The plugin writes to the output log with the custom log category LogNextGenMsg.

*Listing 6.1.: Output Log of Map_PSL_Demo starting PIE*
```log
[...]
LogWorld: Bringing World /Game/UEDPIE_0_Map_PSL_Demo.Map_PSL_Demo up for play (max tick rate 60) at 2022.09.14-13.46.08
LogWorld: Bringing up level for play took: 0.000768
LogOnline: OSS: Created online subsystem instance for: :Context_3
LogNextGenMsg: SubSocketActor1_2: Open socket ...
LogNextGenMsg: NngSocketObject_0: Socket successfully opened.
LogNextGenMsg: SubSocketActor1_2: Open socket done.
LogNextGenMsg: SubSocketActor1_2: Bind tcp://127.0.0.1:5555 ...
LogNextGenMsg: NngSocketObject_0: Successfully listening to tcp://127.0.0.1:5555
LogNextGenMsg: SubSocketActor1_2: Bind done.
LogNextGenMsg: BP_AccDemo_C_3.Subscriber Subscribe topic psl ...
LogNextGenMsg: SubSocketActor1_2 Subscribe topic 'psl' ...
LogNextGenMsg: NngSocketObject_0: Topic successfully subscribed.
PIE: Server logged in
PIE: Play in editor total start time 0.152 seconds.
[...]
```

TODO: GIF
*Fig. 6.1.: Animation Screenshot of Map_PSL_Demo PIE*

<div style='page-break-after: always'></div>

## A. Attribution

* The word mark Unreal and its logo are Epic Games, Inc. trademarks or registered trademarks in the US and elsewhere (cp. Branding Guidelines and Trademark Usage, URL: [https://www.unrealengine.com/en-US/branding](https://www.unrealengine.com/en-US/branding)).
* The word marks nanomsg and NNG and its logos are trademarks of Garrett D'Amore, used with permission (cp. Trademark Policy, URL: [https://nanomsg.org/trademarks.html](https://nanomsg.org/trademarks.html)).
* Docker and the Docker logo are trademarks or registered trademarks of Docker, Inc. in the United States and/or other countries. Docker, Inc. and other parties may also have trademark rights in other terms used herein (cp. Trademark Guidelines, URL: [https://www.docker.com/legal/trademark-guidelines/](https://www.docker.com/legal/trademark-guidelines/)).
* The word marks EMQ, EMQX and NanoMQ and its logos are trademarks of EMQ Technologies Co., Ltd.
* The word mark Polar and its logos are trademarks of Polar Electro Oy.
* Android is a trademark of Google LLC.
* The Bluetooth word mark and logos are registered trademarks owned by Bluetooth SIG, Inc.
* PowerShell and Windows are registered trademarks of Microsoft Corporation.
* Wireshark and the "fin" logo are registered trademarks of the Wireshark Foundation (cp. Legal Information,  URL: [https://www.wireshark.org/about.html](https://www.wireshark.org/about.html)).

## B. References

* *Unreal Engine Plugin: Integration Tool* by Roland Bruggmann aka brugr9 on Unreal Marketplace: [https://www.unrealengine.com/marketplace/en-US/product/integration-tool](https://www.unrealengine.com/marketplace/en-US/product/integration-tool)
* *NanoMQ* MQTT Broker by EMQ: [https://nanomq.io/](https://nanomq.io/)
  * Discussion on GitHub: "New module NNG proxy is finished.but requires docs & demo & configuration files #646", URL: [https://github.com/emqx/nanomq/discussions/646](https://github.com/emqx/nanomq/discussions/646)
  * See also: Daniel Dörig: *Getting started with MQTT/C++*. 01. June, 2018, In: Noser Blog, URL: [https://blog.noser.com/getting-started-with-mqtt-c-plus-plus/](https://blog.noser.com/getting-started-with-mqtt-c-plus-plus/)
* *Polar Sensor Logger* App by Jukka Happonen on Google Play: [https://play.google.com/store/apps/details?id=com.j_ware.polarsensorlogger](https://play.google.com/store/apps/details?id=com.j_ware.polarsensorlogger)
* *Polar H10* Heart Rate Sensor with Chest Strap: [https://www.polar.com/en/sensors/h10-heart-rate-sensor](https://www.polar.com/en/sensors/h10-heart-rate-sensor)
* OASIS Message Queuing Telemetry Transport (MQTT) TC: [https://www.oasis-open.org/committees/tc_home.php?wg_abbrev=mqtt](https://www.oasis-open.org/committees/tc_home.php?wg_abbrev=mqtt)
* *Wireshark*, Display Filter Reference: MQ Telemetry Transport Protocol: [https://www.wireshark.org/docs/dfref/m/mqtt.html](https://www.wireshark.org/docs/dfref/m/mqtt.html)

## C. Readings

* Ch&#281;&cacute;, A.; Olczak, D.; Fernandes, T. and Ferreira, H. (2015). **Physiological Computing Gaming - Use of Electrocardiogram as an Input for Video Gaming**. In Proceedings of the 2nd International Conference on Physiological Computing Systems - PhyCS, ISBN 978-989-758-085-7; ISSN 2184-321X, pages 157-163. DOI: [10.5220/0005244401570163](http://dx.doi.org/10.5220/0005244401570163)

## D. Citation

To acknowledge this work, please cite

> Bruggmann, R. (2022). Unreal Engine Project 'Heartbeat' (Version 1.0.0) [Computer software]. https://github.com/brugr9/heartbeat

```bibtex
@software{Bruggmann_Unreal_Engine_Project_2022,
  author = {Bruggmann, Roland},
  year = {2022},
  month = {10},
  title = {{Unreal Engine Project 'Heartbeat'}},
  version = {1.0.0},
  url = {https://github.com/brugr9/heartbeat}
}
```

---
<!-- Footer -->

[![Creative Commons Attribution-ShareAlike 4.0 International License](https://i.creativecommons.org/l/by-sa/4.0/88x31.png)](https://creativecommons.org/licenses/by-sa/4.0/)

*Unreal&reg; Engine Project "Heartbeat"* &copy; 2022 by [Roland Bruggmann](https://about.me/rbruggmann) is licensed under [Creative Commons Attribution-ShareAlike 4.0 International](http://creativecommons.org/licenses/by-sa/4.0/)
