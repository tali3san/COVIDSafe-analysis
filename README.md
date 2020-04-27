# COVIDSafe App Analysis
Analysis of the COVIDSafe Android app based on simple techniques and publicly available tools for analysing Android apps.

To be clear, my analysis is mostly based on looking at the app from the standpoint of an Android/iOS Bluetooth developer, and not focussing on cryptographic security.

## The TLDR
The decision to release the app now and not to wait until Google/Apple have their protocol/APIs ready is astounding, especially given these APIs should be avilable in the next few days.

As far as I can tell, the app does what it says it does on the phone (and does not CURRENTLY use lattitude/longitude based location services). However giving the permission to do this means that it is open to abuse in future versions of the app.

I cannot make any claim for the data retention or the security of the network services. 

The Australian Government needs to commit publicly to working with Google and Apple to retrofit the app when they introduce their new APIs in a few weeks (I've yet seen a single article to indicate this).

The requirement to run a foreground service on Android (with a notification that can't be removed) and to 
allow location permissions (required on Android to scan), may limit uptake (see permission related messages). 

On iOS side the workarounds necessary to keep the app running (user interaction) may result in the app not running when it actually needs to. 

Add that it is unclear, with the requirement for an actual Bluetooth connection to be made to exchange data how this will scale in more crowded environments. 

While it could be better than nothing, it could also induce a false sense of security, especially if the government gets the take up it wants and opens things up more, resulting in larger groups of people congregating (obeying social distancing rules). 

Also the requirement to submit data to the app before a diagnosis is a requirement imposed by this app and protocol and if you look at the Google/Apple proposal should only be necessary when it has been determined that you have been in close contact with someone who has tested positive and you consent to provide details at that time.

## Introduction

The Bluetooth tracking part of the app appears to be based on OpenTrace:

https://github.com/opentrace-community/opentrace-android/tree/master/app/src/main/java/io/bluetrace/opentrace/services

This is an open source implementation of the BlueTrace protocol developed by the Singapore government.
As far as I know this is not the same as the protocol being worked on by Google and Apple.  I need to give both
white papers a read to see how they match up.

https://bluetrace.io/

This protocol requires phones to initiate a Bluetooth connection with each other and exchange data, whereas the Google and Apple protocol
does not (https://covid19-static.cdn-apple.com/applications/covid19/current/static/contact-tracing/pdf/ExposureNotification-BluetoothSpecificationv1.1.pdf).
This is down to the fact this is not backed by the operating system or vendor libraries and on Apple in particular
an app cannot initiate advertising and attach service data or manufacturer data. This means the anonymized token cannot be exchanged
merely in the advertising data.

Specifically the Google/Apple protocol states:

```text
During the Bluetooth broadcast, advertisements are to be non-connectable undirected of type ADV_NONCONN_IND
(Section 2.3.1.3 of 5.2 Core Spec).
```

I believe this would limit the effectiveness of the app, if for example you were in a moderately busy place. Making a Bluetooth connection to a
"new" device requires multiple data packets to be exchanged to read the service information before you can use that service.

The requirement to connect also brings with it many problems inherent with Bluetooth connections, even BLE ones.

The Google/Apple proposal is way more efficient as the information is exchanged by virtue of receiving the scan data.

The reference Android implementation uses Google's Firebase. The COVIDSafe app uses AWS.

## Specific Bluetooth Issues

### Connecting in General (Android/iOS)
The requirement to connect can means that the app has to receive an advertisement packet, process it and then initiate a connection 
request to repeat the exchange. There are many, many ways for the device to become difficult or impossible to connect to between these two 
points in time.  

### Direct Connections (Android)
On Android, when you attempt to connect to a remote device, you can do so in one of two modes, directly or when the remote device becomes available.
The direct method is quicker, but runs into problems if the device is not in range at the time you initiate the connection.

Only one direct connection attempt can be made at any time. If multiple connection attempts are made, each will be blocked until the preceding one
has connected or timed out. On earlier versions of Android the direct connection timeout was 30 seconds and this connection attempt CANNOT be cancelled.
This blocking behaviour extends across the Bluetooth stack to any app trying to connect. On newer and different manufacturer 
versions of Android that timeout is reduced, but it still exists.

This could mean that if the user has another app installed that utilised direct connections (or a malicious app, 
no user requested permission is required to make an attempt to connect to another device), it could block or significantly slow 
down the app from exchanging data with other phones. 

### Service Discovery (Android/iOS)
Connecting to a NEW device takes time as the phone is required to perform service discovery before it can use the GATT service to exchange
data with the other phone. This spans an exchange of multiple data packets and means connecting takes longer than it would for a non new device.

In the situation here, EVERY device is new to each other, so the additional service discovery step will be required for almost every connection.

### Multiple Connections
In a situation where there are multiple devices and people moving around each other the requirement for each pair of devices to connect to
exchange information with each other creates scalability issues, particularly in the above case where a connection times out. 

### Transport Mechanism Selection (Android)
The app doesn't use transport selection when creating the BLE connection. While developing an app that connected to an 
iOS device running an app to simulate a peripheral, it was noted that some Android devices required using the transport selection introduced in
Android 6 to connect properly with iOS devices (`BluetoothDevice.TRANSPORT_LE`). This issue may have been iOS related and resolved by now, but
adding the transport selection merely indicates the default. 

Existing code:
```java
BluetoothGatt bluetoothGatt = this.device.connectGatt(paramContext, false, paramStreetPassGattCallback);
```
May be more reliable when connecting to iOS as:
```java
BluetoothGatt bluetoothGatt = this.device.connectGatt(paramContext, false, paramStreetPassGattCallback, BluetoothDevice.TRANSPORT_LE);
```

### Foreground Service and Boot Permissions

To be able to scan continuously with only normal developer permissions, the app requires a non dismissable notification and a foreground service. The app can thus only be killed via the preferences app and the "Force Stop" option. This can lead to issues if the app malfunctions in a way that requires a restart. The app is also restarted automatically on boot or on install of an updated. 

### RSSI (Android)
The protocol exchanges the phone model in an attempt to account for phone model based variations in broadcasting 
strength and receive sensitivity. RSSI is only a very rough indication of signal strength and in this case one or both phones
may be in bags, pockets etc. Where the RSSI would be significantly affected by the phone orientation and what other 
material is adjacent to it (e.g. peoples bodies (which quite effectively block/absorb 2.4Ghz), other devices (iPad etc.), keys, wallet etc. etc.).

### Denial of Service
The requirement to connect makes denial of service attacks much easier to carry out.  

## Libraries Used

### GUI
* https://github.com/airbnb/lottie-android
* https://github.com/razir/ProgressButton
### Feedback
* Atlassian's JIRA mobile connect: https://developer.atlassian.com/server/jira/platform/jira-mobile-connect-4653058/
### Utility
* Permissions: https://github.com/googlesamples/easypermissions
* Reactive programming: http://reactivex.io/
* HTTP/S request wrapping: https://square.github.io/retrofit/
* JSON encoding/decoding: https://github.com/google/gson
* Google's Tink encryption libraries:  https://github.com/google/tink
### Android APIs
* Room persistence API: https://developer.android.com/topic/libraries/architecture/room
* SQLite: https://www.sqlite.org/index.html

## Permissions Required

### Permissions
* android.permission.INTERNET
* android.permission.BLUETOOTH
* android.permission.BLUETOOTH_ADMIN
* android.permission.ACCESS_FINE_LOCATION
* android.permission.RECEIVE_BOOT_COMPLETED
* android.permission.FOREGROUND_SERVICE
* android.permission.REQUEST_IGNORE_BATTERY_OPTIMIZATIONS
* android.permission.ACCESS_NETWORK_STATE

### Related communication
The app contains the messages:

```text
Android requires location access to enable Bluetooth functions for COVIDSafe. COVIDSafe cannot work properly without it.
```

and:

```text
COVIDSafe needs Bluetooth® and notifications enabled to work.

Select ‘Proceed’ to enable:
1. Bluetooth®
2. Location Permissions
3. Battery Optimiser
```
