---
title: Intercepting and modifying Fitbit Charge 5 messages
description: >-
  How I walked 1337 steps from the comfort of my chair.
author: ivanv
date: 2024-03-13 22:45:00 +0200
categories: [Android, Mobile]
tags: [golden gate, frida, fitbit, reverse engineering]
pin: true
media_subpath: '/posts/20250828'
---
# What is Golden Gate

> Disclaimer: the best source for information about what is Golden Gate, how it works and how to use it is the [source code](https://github.com/Fitbit/golden-gate) and the [documentation](https://fitbit.github.io/golden-gate/). I have decided to *copy* very few parts of the documentation as they are useful for what will be explained later, but they might be outdated now so please refer to the live documentation.

Golden Gate is a framework to connect wearables and other IoT devices to mobile phones, tablets and PCs with an IP-based protocol stack over Bluetooth Low Energy, and in this article I will explain what Golden Gate is, where it's used, how it works and what I learned about it.

The goal of the Golden Gate project is to allow implementers of embedded devices to communicate with remote (typically mobile) applications in a way that looks and feels like the way they would do it in a traditional networked environment, where the de-facto standard is to use web technologies like TCP/IP for transport, TLS for security, HTTP for applications, and other familiar protocols.

Notably the Golden Gate framework is what Fitbit devices uses to handle the communication between the wearable the device (eg. Fitbit Charge 5) and the Fitbit Android App.
In this article I will explain how I reverse engineered the functionality of the Fitbit App and how it uses the Golden Gate framework to communicate with Fitbit devices. Ultimately, to demonstrate the understanding the communication I will showcase how I was able to intercept and modify messages sent between the app and the device.

## Context

My research about Golden Gate done in early 2024 while the project was still in active development, but at the time of writing the GitHub repository is archived.
This project was carried under the supervision of Marco Casagrande, you can find more about his work about Android and Bluetooth on his [Google Scholar page](https://scholar.google.com/citations?user=H2VzxLwAAAAJ).

In this article I will discuss the research done on:

- Fitbit Charge 5 Firmware: 57.20001.194.91
- Fitbit App: Fitbit_4.03.fitbit-mobile-40349031-576203982 (SHA256: 6e1c36e6135f8b28d3d29c46920f60c180e40765ca7cf26bda76cd773b9ae15f, [Download Link](https://apkcombo.com/fitbit/com.fitbit.FitbitMobile/download/phone-4.03.fitbit-mobile-40349031-576203982-apk))

## Stack diagram

Notably, the Golden Gate framework is what Fitbit devices use to handle the communication between the wearable (e.g., Fitbit Charge 5) and the Fitbit Android App.

To achieve this seamless integration, Golden Gate implements a core library that offers a complete network stack with cleanly separated layers. This architecture leverages well-established standards to bridge the gap between low-power hardware and modern web environments.

On top of the stack are the applications and services that make or receive requests. For application requests, the first supported protocol is CoAP. CoAP is a lightweight application protocol optimized for constrained devices.
The requests transit through the network stack, all the way to the bottom of the stack, where a physical transport layer is used to communicate with a remote, peer application or service. On the peer is an equivalent stack that the requests traverses in the reverse direction, all the way to that peer's own set of applications and services. The responses take the reverse route all the way back to the original client application.

![The Stack Diagram](/StackDiagram.png){: width="700" height="400" }
*The Stack Diagram*

## CoAP

CoAP is an internet technology standard defined in [RFC7252](https://tools.ietf.org/html/rfc7252). It offers a request/response protocol very similar to HTTP, but with a lower processing overhead and the ability to be transported over UDP instead of TCP. It has builtin support for reliability (even though UDP itself does not offer that), and well-defined semantics that are very close to what developers may be used to with HTTP, such as `GET`, `PUT`, `POST` and `DELETE` methods, request and response headers ("options" in CoAP terminology), numeric response status codes that map 1-1 with HTTP response status codes, and more.

### Getting started with CoAP

Before getting started and learning how the Fitbit App communicates with the Fitbit Charge 5 it's useful to get a feeling of how CoAP works in practice and how it looks like.
In the following sections I will show how it's is possible to setup a CoAP server and connect to it with an CoAP client, and inspect the communication between the two using Wireshark.

#### Setup CoAP

For this project I will be using WSL2 (Ubuntu 22.04.3 LTS).

Clone the project (commit SHA: `94e3f1d7e7daef11bd4cc46df79d798796c63fff`):
```sh
git clone https://github.com/Fitbit/golden-gate.git
cd golden-gate
git submodule update --init --recursive
```

Install [miniconda](https://docs.conda.io/projects/miniconda/en/latest/) (`wget https://repo.anaconda.com/miniconda/Miniconda3-latest-Linux-x86_64.sh`):
```sh
wget https://repo.anaconda.com/miniconda/Miniconda3-latest-Linux-x86_64.sh
chmod +x Miniconda3-latest-Linux-x86_64.sh
./Miniconda3-latest-Linux-x86_64.sh
export PATH="/home/user/miniconda3/bin:$PATH"
```

Create an environment, go inside the golden-gate directory and run:
```sh
conda env create -n gg
source activate gg
```

#### Building and running a CoAP application

Inside the `golden-gate` directory and with the virtual environment enabled run the following command to build the binaries:
```sh
(gg) $ inv native.build
```

You will find the binaries:

- `xp/build/cmake/native/apps/coap-client/gg-coap-client`
- `xp/build/cmake/native/apps/coap-server/gg-coap-server`
- `xp/build/cmake/native/apps/stack-tool/gg-stack-tool`

Start a server on the default port:

```sh
(gg) $ xp/build/cmake/native/apps/coap-server/gg-coap-server
```

Get the `helloworld` resource of the server:

```sh
(gg) $ xp/build/cmake/native/apps/coap-client/gg-coap-client get coap://127.0.0.1/helloworld
```

You should see the server replying with the message `Hello, World`.

If you want to create a new endpoint you will have to edit the file: `xp/apps/coap-server/gg_coap_server.c`. After editing the file a rebuild is required `inv native.build`.

#### Wireshark packet inspection

Download [wireshark](https://www.wireshark.org/download.html). If you are using Windows choose to install the USB drivers (USBPcap) for Android at installation time, you will need them later.
To see the traffic in Wireshark, open the loopback interface and apply the filter `udp.port == 5683`.

![Wireshark screenshot showcasing the GET request sent from the CoAP Client to the CoAP Server on the /helloworld endpoint.](/Wireshark_1.png)
*Wireshark screenshot showcasing the GET request sent from the CoAP Client to the CoAP Server on the /helloworld endpoint.*

Congratulations, you are seeing CoAP in action!

### Building and modifying the sample Android Application

In the Golden Gate repository they provide a sample Android Application. The app is located at `golden-gate/platform/android/goldengate/`. The app defines a resource handler for a GATT server, which is the piece of code resposible for handling GET requests received on the main endpoint.
Generic Attribute Profile (GATT) is a general specification for sending and receiving short pieces of data known as "attributes" over a BLE link. All current BLE application profiles are based on GATT. In GATT, once a connection is established, one device acts as the server, ther other as the client. The device in the client role sends requests for data, and the device in the server role fulfills them.

The original file is at [`platform/android/goldengate/app/src/main/kotlin/com/fitbit/goldengatehost/coap/handler/HelloWorldResourceHandler.kt`](https://github.com/Fitbit/golden-gate/blob/94e3f1d7e7daef11bd4cc46df79d798796c63fff/platform/android/goldengate/app/src/main/kotlin/com/fitbit/goldengatehost/coap/handler/HelloWorldResourceHandler.kt), but I've mofidied to return "Hello Ivan!", instead of "Hello world!".

```java
// Copyright 2017-2020 Fitbit, Inc
// SPDX-License-Identifier: Apache-2.0

package com.fitbit.goldengatehost.coap.handler

import com.fitbit.goldengate.bindings.coap.data.ContentFormatOption
// [...] 
import io.reactivex.Single

/**
 * Coap resource handler that will return "Hello world!"
 */
class HelloWorldResourceHandler : BaseResourceHandler() {

    override fun onGet(request: IncomingRequest, responseBuilder: OutgoingResponseBuilder): Single<OutgoingResponse> {
        return Single.just(helloWordResponse(responseBuilder))
    }

    private fun helloWordResponse(responseBuilder: OutgoingResponseBuilder) =
            responseBuilder
                    .option(ContentFormatOption(FormatOptionValue.TEXT_PLAIN))
                    .body("Hello Ivan!".toByteArray())
                    .build()
}
```

Once a GET request is received on the `/helloworld` endpoint, the server will reply with the string: `Hello Ivan!`.
Once the app is recompiled, and running, you can install it on two devices and let them talk via CoAP. On one device click the "SEND COAP REQUEST OVER UDP" and on the second device you should see something similar to: 

![The Modified Golden Gate App](/GoldenGateHost_4.jpg){: width="400" height="400" }
*The Modified Golden Gate App*

### CoAP interception over BLE in Wireshark

I won't go over the details on how to see BLE traffic Wireshark, but while the app is running, this is the BLE traffic sent during a simple GET request:

![The request](/Wireshark_3.png){: width="700" height="400" }
*The request to GET /helloworld*


And the reponse:

![The response](/Wireshark_4.png){: width="700" height="400" }
*The response to GET /helloworld*


The traffic can be seen in clear text! The GoldenGate app offers the ability to use a stack configuration with encryption thanks to TLS. Of course if you use that Stack Configuration, in Wireshark you will only see encrypted data:

![The encrypted request](/Wireshark_5.png){: width="700" height="400" }
*The encrypted to GET /helloworld*

Encryption is one of the key features of Golden Gate. Encryption is needed because as explained in the [Android BLE Guide](https://developer.android.com/develop/connectivity/bluetooth/ble/ble-overview): 
> When a user pairs their device with another device using BLE, the data that's communicated between the two devices is accessible to all apps on the user's device.  For this reason, if your app captures sensitive data, you should implement app-layer security to protect the privacy of that data.

