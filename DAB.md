
# Device Automation Bus (DAB)
Version: 2.0  |  Last Updated: 2024-04-23

# Table of Contents:
1. [Introduction](#1-introduction)
2. [About the Specification](#2-about-the-specification)
3. [Requirements](#3-requirements)
4. [Protocol Basics](#4-protocol-basics)
5. [DAB Operations](#5-dab-operations)   
	5.1. [Supported operations](#51-supported-operations)   
	5.2. [Applications](#52-applications)  
	5.3. [System](#53-system)   
	5.4. [Input](#54-input)   
	5.5. [Output](#55-output)   
	5.6. [Device and Application Telemetry](#56-device-and-application-telemetry)   
	5.7. [Health Check](#57-health-check)   
	5.8. [General Notifications](#58-general-notifications)   
	5.9. [Voice](#59-voice)   
6. [Versioning of the protocol](#6-versioning-of-the-protocol)
7. [Appendix](#7-appendix)
8. [Deprecated APIs](#8-deprecated-apis)

# 1. Introduction
This document describes Device Automation Bus (DAB), a simple protocol for intuitively interacting with living room devices. Living room devices are consumer electronics in the living room that are connected to the Internet such as televisions and games consoles. DAB is a protocol that allows device manufacturers and application developers to programmatically communicate with living room devices to start and stop applications, send key presses to the device, restart the device, capture device metrics among other operations. Using DAB, device manufacturers and application developers can automate manual tasks such as application compatibility checks, cross application performance testing and monitoring device memory usage. Automating manual tasks provides reliable results and saves time and money.

The diagram below provides a high level view of how DAB works:  
![DAB diagram](https://i.ibb.co/tLn05Gj/DAB-Diagram.png)

# 2. About the Specification
This specification describes a message based protocol to support programmatically communicating with a device under test. This specification provides a description of the supported messages and their payloads as well as positive and negative response codes.

The DAB protocol is built on top of [MQTT 5](https://docs.oasis-open.org/mqtt/mqtt/v5.0/mqtt-v5.0.html). MQTT is a publish-subscribe messaging transport protocol for connecting to remote devices. For more details, see https://mqtt.org/

### Not covered in the specification
The following topics are not covered in this specification:
#### Security
Recommendations for securing DAB, as well as connecting to a device under test that supports DAB will be defined in a separate document.
#### Protocol Compliance
Protocol Compliance testing will be defined in a separate DAB certification guide.

# 3. Requirements

An endpoint is DAB compliant only if the following conditions are met:
- [HTTP/1.1](https://datatracker.ietf.org/doc/html/rfc7230) or above is supported for media operations
- 2 megabytes are dedicated to DAB for media operations

# 4. Protocol Basics
This section describes message payloads, request and response formats, message types and common status codes.

## Request / Response Model
DAB relies on the [MQTT request/response](https://docs.oasis-open.org/mqtt/mqtt/v5.0/os/mqtt-v5.0-os.html#_Toc3901252) model which is native to all MQTT 5 compliant brokers and clients.

To send requests and receive responses from DAB:
1. The responding client (device side) is running and subscribed to all topics corresponding to the supported DAB operations.
2. The requesting client (user side) determines to which topic responses should be sent and subscribes to this `responseTopic`. Response topics may be reused if `correlationData` is included, otherwise they must be unique.
3. The requesting client generates a unique binary request ID, referred to as `correlationData`, to aid in associating responses with requests. Use of correlation data is not required, but strongly encouraged.
4. The requesting client publishes the message to the request topic with the required `responseTopic` and optional `correlationData` message properties.
5. When the request is received by the responding client, an operation is performed, response message is formed, `correlationData` is copied from the request message property to the response (if it exists), and the message is published to the `responseTopic` property from the request.
6. The operation is complete when the requesting client receives a message on the `responseTopic` with matching `correlationData` (if applicable), or the client chooses to time out the request.

##### Example
1. The user client wants to send a message to the request topic `dab/<device-id>/health-check/get` and get responses at `dab/userClient-1/responses`. Note: The `responseTopic` doesn't have to be similar to `dab/userClient-1/responses`, it can be anything that the user client specifies. Device client uses the `responseTopic` field to send back the response.
2. A unique binary `correlationData` value is generated.
3. The user client subscribes to `dab/userClient-1/responses` and waits for a response message with the matching `correlationData` value to arrive.
4. The user client publishes a message to `dab/<device-id>/health-check/get` with request properties `correlationData` and `responseTopic: dab/userClient-1/responses`.
5. When a message sent by device client is received on the user client at `dab/userClient-1/responses` with matching `correlationData`, the message payload will be the health-check response and the operation is complete.

##### Sample MQTT Request Message:
```shell
   topic: "dab/<device-id>/health-check/get",
   correlationData: "abc",
   responseTopic: "dab/userClient-1/responses",
   payload: DabRequest{...}
```

##### Sample MQTT Response Message:
```shell
   topic: "dab/userClient-1/responses",
   correlationData: "abc",
   payload: DabResponse{...}
```

### Device ID Implementation Guidelines
The Device ID is an unique identifier that helps with DAB message routing across multiple device clients.

One should implement the Device ID as:
1. The device under test defines what its Device ID should be.
	2.1. It may or may not be baked into the device, but it should be globally unique.
	2.2. It may be configurable, but using transient values, such as current IP address, is not recommended as this can make reconnecting after restart difficult.
2. Device ID must be a string, and only include lower case alphanumeric characters and -_.

### Device Setup
When a device client first connected to the MQTT broker, it should do the following setup procedure:
1. Subscribe to all topics listed in this spec, like `dab/<device-id>/device/info` topic. 
2. Subscribe to `dab/discovery` topic.

### Device Discovery
These are the steps using which a user client can discover all the DAB device clients present in the current setup:

1. When a User client connects to the MQTT broker for the first time, there might be device clients already connected to the broker. User Client should publish a request to the discovery topic `dab/discovery` similar to the following:
```shell
   topic: "dab/discovery",
   correlationData: "abc",
   responseTopic: "dab/userClient-1/responses",
   payload: DabRequest{...}
```
2. All connected DAB devices will send a response back to the `responseTopic` as set by the user client. The user client can then extract the device-id from these responses.

### Message Payloads
DAB protocol message payloads are always JSON and the characters must use UTF-8 encoding.

This specification uses [typescript interfaces](https://www.typescriptlang.org/docs/handbook/interfaces.html) to define request and response schemas. Typescript interfaces are used in order to be descriptive, compact and concise with the request and response schema definitions.

As an example, the following JSON maps to the example Typescript interface (and vice versa):

```json
{
   "variableName": "Value",
}
```
maps to:

```typescript
interface ExampleResponse {
   variableName: string;
   optionallyIncludedVariable?: string;
}
```

For more information on TypeScript, please see the [TypeScript documentation](https://www.typescriptlang.org/docs).

### Common DAB Request Payload

All DAB requests must be valid JSON objects even if no parameters are required. All DAB requests must inherit from the  `DabRequest` format

#### Request Format

```typescript
interface DabRequest {
}
```
Sample Response
An empty request is equal to an empty JSON map

```json
{}
```

### Common DAB Response Payload

All DAB responses must inherit from the  `DabResponse` format. DAB responses must contain the `status` field and may contain the `error` field when an error occurs.

#### Response Format

```typescript
interface DabResponse {
   status: number;
   error?: string;
}
```

#### Response Parameters

Parameter |  Description
--- | ---
status | status code
error | optional explanation of the error

#### Common status codes

Status code | Description
--- | ---
200 | The request was successful.
400 | Bad request. The request was invalid or malformed. The explanation of the error must be included in the `error` field of the response.
500 | Internal error. The explanation of the error must be included in the `error` field of the response.
501 | Not implemented. The device must return this error when the requested functionality is not implemented.

An operation must respond with one of the above common status codes, unless the operation states otherwise.

#### Sample Responses

A response with the success status code

```json
{
   "status": 200
}
```

A response with the error status code

```json
{
   "status": 400,
   "error": "App parameter is required to launch the application"
}
```

## Unsolicited Messages
Unsolicited Messages are a feature of MQTT. An unsolicited message is a message that can be sent to communicate temporal messages like memory usage. In most cases, the client will subscribe to a topic where these messages are being sent. The client may or may not have to send a request for these messages to be enabled. Subscribing to the topic does not give the client a history or even the last published message - they will only get future messages.

## Data Types Definitions

|Friendly name|Type identifier|Description|Example|
|-|-|-|-|
|Language code|`rfc_5646_language_tag`|All language tags use the IETF language tags defined in [RFC 5646](https://tools.ietf.org/html/rfc5646). All language tags are string fields.|"en-US"|
|UNIX Time stamp|`unix_timestamp_ms`|UNIX Epoch time in milliseconds. The data type is a 64 bit unsigned integer.|1568815387403|
|Application Identifier|`appId`|The application identifier consists of characters. For resiliency, uppercase letters are equivalent to lowercase (e.g., "netflix" is the same as "Netflix"). The minimum number of characters is 1 and the maximum is 64.|"PrimeVideo"|
|Uniform Resource Locator|`url`|MQTT has a max play load size of 256mb for binary data. HTTP is used for pushing and pulling resources from devices to support larger media files. URLs are defined in [RFC 3986](https://datatracker.ietf.org/doc/html/rfc3986).|http://192.168.0.1/file|

### Standardized Application and Voice System Identifiers in Appendix

In order to refer to applications and voice systems in a standardized manner, key application identifiers and voice system names are provided within [Secton 7 -- Appendix](#7-appendix). DAB implementations must use these appId and voice system names, and not any internal codenames. This is necessary for test automation to use uniform names across all DAB-capable platforms.

# 5. DAB Operations

## 5.1 Supported operations
*Operation model: Request / Response*

### Operation: Listing all the supported operations
The operation lists all the DAB operations supported by the device. All devices are required to implement this operation. The operations are identified by the name of the topic without the `dab` prefix. The result should not include the list operations operation itself.

#### Request Topic

`dab/<device-id>/operations/list`

#### Request Format

```typescript
type ListSupportedOperationRequest = DabRequest
```

#### Response Format

```typescript
interface ListSupportedOperationResponse extends DabResponse {
   operations: string [];
}
```
#### Response Parameters

Parameter | Description
--- | ---
operations | a list of operations supported by the DAB device client

#### Sample Response

```json
{
   "status": 200,
   "operations": [
      "applications/list",
      "applications/launch",
      "applications/launch-with-content",
      "applications/get-state",
      "applications/exit",
      "device/info",
      "system/restart",
      "system/settings/list",
      "system/settings/get",
      "system/settings/set",
      "input/key/list",
      "input/key-press",
      "input/long-key-press",
      "output/image",
      "device-telemetry/start",
      "device-telemetry/stop",
      "device-telemetry/metrics",
      "app-telemetry/start",
      "app-telemetry/stop",
      "app-telemetry/metrics",
      "health-check/get",
      "messages",
      "voice/list",
      "voice/send-audio",
      "voice/send-text",
   ]
}
```

## 5.2. Applications

### Application states

An application can be in one of the following states:

State | Description
--- | ---
STOPPED | There is no instance of the application running on the device.
FOREGROUND | The application is visible, active and accepting input.
BACKGROUND | There is a running instance of the application on the device but it is not visible and not accepting input.

Your device may support more states, choose the one that best represents the described state

### Operation: Listing all the Applications
*Operation model: Request / Response*

This operation lists the applications installed on the device. This list can be limited to applications that are known to leverage DAB.

#### Request Topic

`dab/<device-id>/applications/list`

#### Request Format

```typescript
type ListApplicationsRequest = DabRequest
```

#### Response Format

```typescript
interface Application {
   appId: string;
   friendlyName?: string;
   version?: string;
}
```

Parameter | Description
--- | ---
appId | Application Identifier, as [defined in the DAB Registry](#7-appendix)
friendlyName | *optional* application friendly name
version | *optional* application version

```typescript
interface ListApplicationsResponse extends DabResponse {
   applications: Application [];
}
```

#### Response Parameters

Parameter | Description
--- | ---
applications | a list of applications

#### Sample Response
```json
{
   "status": 200,
   "applications": [
      {
         "appId": "Netflix",
         "friendlyName": "Netflix",
         "version": "2.0.0"
      },
      {
         "appId": "PrimeVideo",
         "friendlyName": "Amazon Prime Video",
         "version": "2.0.0"
      },
      {
         "appId": "YouTube",
         "friendlyName": "YouTube",
         "version": "2.0.0"
      }
   ]
}
```

### Operation: Launching an Application
*Operation model: Request / Response*

This operation launches an instance of the application into the FOREGROUND state. There can only be one instance of an application running in the system and the instance is referenced by the application identifier. The response should only be sent after the application state has changed on the platform.

* If the application is in the STOPPED state, this operation will launch a new instance into the FOREGROUND state. 
* If the application is in the BACKGROUND state, this operation will bring the instance of the application into the FOREGROUND state. 
* If the application is in the FOREGROUND state, this operation will not change the state of the application.


#### Request Topic

`dab/<device-id>/applications/launch`

#### Request Format

```typescript
interface LaunchApplicationRequest extends DabRequest {
   appId: string;
   parameters?: string [];
}
```

#### Request Parameters

Parameter | Default |  Description
--- | --- | ---
appId | - | Application Identifier
parameters | `[]` | URL encoded list of parameters to be passed to the application being launched (usually directly to the app executable). The technique to pass parameters may be app specific. Please work with [app partners in DAB Registry](#7-appendix) for how the parameters are expected to be passed.

#### Sample Request

```json
{
   "appId": "Netflix",
   "parameters": [
    "-KEY",
    "https%3A%2F%2Fwww.example-value.com%2F",
    "-STANDALONE_PARAM",
    "%22param-with-quotes%22",
   ]
}
```

In the above sample, since `parameters` is defined, DAB implementations must URL decode each parameter and pass them in order. As an example, for Netflix it would look like `./netflix -KEY https://www.example-value.com -STANDALONE_PARAM "param-with-quotes"`. Note how the order is preserved and parameters containing quotes and other miscellaneous characters are passed without issue due to URL encoding / decoding.

#### Response Format

```typescript
type LaunchApplicationResponse = DabResponse
```

### Operation: Launching an application to the specified content
*Operation model: Request / Response*

This operation launches the application into FOREGROUND state with the specified content (also known as deep linking). There can only be one instance of an application running in the system and the instance is referenced by the application identifier. The response should only be sent after the application state has changed on the platform.

* If the application is in the STOPPED state, this operation will launch a new instance into the FOREGROUND state with the specified content
* If the application is in the BACKGROUND state, this operation will bring the instance of the application into the FOREGROUND and pass the specified content to the instance
* If the application is in the FOREGROUND state, this operation will pass the content to the instance


#### Request Topic

`dab/<device-id>/applications/launch-with-content`

#### Request format

```typescript
interface LaunchApplicationWithContentRequest extends DabRequest {
   appId: string;
   contentId: string;
   parameters?: string [];
}
```

#### Request Parameters

Parameter | Default | Description
--- | --- | ---
appId | - | Application Identifier
contentId | - | Application specific content identifier
parameters | `[]` | URL encoded list of parameters to be passed to the application being launched (usually directly to the app executable). The technique to pass parameters may be app specific. Please work with [app partners in DAB Registry](#7-appendix) for how the parameters are expected to be passed.

#### Sample Request

```json
{
   "appId": "YouTube",
   "contentId": "XqZsoesa55w"
}
```

#### Response Format

```typescript
type LaunchApplicationWithContentResponse = DabResponse
```

### Operation: Get the application state
*Operation model: Request / Response*

This operation returns the state the application is in.

#### Request Topic

`dab/<device-id>/applications/get-state`

#### Request Format

```typescript
interface GetApplicationStateRequest extends DabRequest {
   appId: string;
}
```

#### Request Parameters

Parameter | Default|  Description
--- | --- | ---
appId | - | Application Identifier

#### Sample Request

```json
{
   "appId": "YouTube"
}
```

#### Response Format

```typescript
interface GetApplicationStateResponse extends DabResponse {
   state: string;
}
```
#### Response Parameters

Parameter | Description
--- | ---
state | state of the application at the time of the request, one of `STOPPED`, `BACKGROUND` or `FOREGROUND`

#### Sample Response
```json
{
   "status": 200,
   "state": "BACKGROUND"
}
```

### Operation: Exiting the application
*Operation model: Request / Response*

This operation exits the application. 
* If the optional *background* parameter is set, this operation attempts to move the application to the BACKGROUND state.
* If the *background* parameter is omitted or is set to false, this operation attempts to completely stop the application and move it to a STOPPED state.

When using the exit operation without the background flag, the intention is to stop/kill the corresponding application process.
When using the exit operation with the background flag, the intention is to put the application in a background state, similarly to when the user presses  the `home` key.
The response should only be sent after the application state has changed on the platform.

#### Request Topic

`dab/<device-id>/applications/exit`

#### Request Format

```typescript
interface ExitApplicationRequest extends DabRequest {
   appId: string;
   background?: boolean;
}
```

#### Request Parameters

Parameter | Default|  Description
--- | --- | ---
appId | - | Application Identifier
background | false | decides whether to background or completely stop the application

#### Sample Request

```json
{
   "appId": "PrimeVideo",
   "background": false
}
```

#### Response Format

```typescript
interface ExitApplicationResponse extends DabResponse {
   state: string;
}
```
#### Response Parameters

Parameter |  Description
--- | --- 
state | state of the application after the exit operation


#### Sample Response

```json
{
   "status": 200,
   "state": "STOPPED"
}
```

## 5.3. System

### Operation: Device Information
*Operation model: Request / Response*

#### Request Topic

`dab/<device-id>/device/info`

#### Request Format

```typescript
type GetDeviceInformationRequest = DabRequest
```

#### Response Format

```typescript
enum NetworkInterfaceType {
   Ethernet = "Ethernet",
   Wifi = "Wifi",
   Bluetooth = "Bluetooth",
   Coax = "Coax",
   Other = "Other",
}

enum displayType {
   // Device has native display capability
   Native = "Native",
   // Device cannot display natively and needs an external display connected
   External = "External",
}

interface NetworkInterface {
   connected: boolean;
   macAddress: string;
   ipAddress?: string;
   dns?: string [];
   type: NetworkInterfaceType;
}

interface GetDeviceInformationResponse extends DabResponse {
   manufacturer: string;
   model: string;
   serialNumber: string;
   chipset: string;
   firmwareVersion: string;
   firmwareBuild: string;
   networkInterfaces: NetworkInterface [];
   displayType: DisplayType;
   screenWidthPixels: number;
   screenHeightPixels: number;
   uptimeSince: unix_timestamp_ms;
   deviceId: string;
}
```
#### Response Parameters

Parameter | Description
--- | ---
manufacturer | The device manufacturer e.g. Samsung
model | The device model e.g. UE32F4500
serialNumber | String used by the manufacturer to uniquely identify this device
chipset | Device chipset e.g. X12
firmwareVersion | Device OS or firmware version e.g. 7.00
firmwareBuild | Device OS or firmware build number e.g. 60129
networkInterfaces | One or more network interfaces, indicating the type (e.g. WiFi, Ethernet, Bluetooth), connection status, MAC address, and IP address (if applicable)
displayType | Type of display used by the device e.g. Native display like TVs, Monitors, Projector etc or an external display like HDMI/DisplayPort used by STBs, Streaming Sticks, Soundbars etc
screenWidthPixels | Current screen resolution width measured in pixels (e.g. 1920). Should not be present when no screen is attached.
screenHeightPixels | Current screen resolution height measured in pixels (e.g. 1080). Should not be present  when no screen is attached.
uptimeSince | The unix timestamp of when the device was last booted (e.g. 1615205137769 would denote the device was last booted on Mon Mar 08 2021 12:05:37, UTC)
deviceId | The unique identifier of this device. Should be same as <device-id> in the topic.

If any of the parameters are not applicable/supported, the whole parameter should be omitted from the response.

### Operation: Restarting the device
*Operation model: Request / Response*

This operation will start the device restart sequence. After the device is restarted, DAB should stay enabled; DAB remaining enabled is the only expectation of the specification.

#### Request Topic

`dab/<device-id>/system/restart`

#### Request Format

```typescript
type RestartDeviceRequest = DabRequest
```

#### Response Format
```typescript
type RestartDeviceResponse = DabResponse
```

Success:
```json
{
   "status": 200
}
```

Error:
```json
{
   "status": 403,
   "error": "insufficient privileges"
}
```

### Operation: List system settings
*Operation model: Request / Response*

This operation is used to retrieve the possible values that can be assigned to each system setting. 

Implementations must consider the current operating environment and surface only the setting values that the device believes is possible to set.

#### Request Topic

`dab/<device-id>/system/settings/list`

#### Request Format

```typescript
type ListSupportedSystemSettingsRequest = DabRequest
```

#### Response Format

```typescript
interface OutputResolution {
   width: int;
   height: int;
   frequency: number;
}

enum MatchContentFrameRate {
    // Match Content Frame Rate is Always Allowed even if the transition is not Seamless
    EnabledAlways = "EnabledAlways",
    // Match Content Frame Rate is Only Allowed when the transition is Seamless
    EnabledSeamlessOnly = "EnabledSeamlessOnly",
    // Match Content Frame Rate is Disabled
    Disabled = "Disabled",
}

enum HdrOutputMode {
    // Always output HDMI signal in HDR Format
    AlwaysHdr = "AlwaysHdr",
    // Output HDMI signal in HDR Format Only when playing HDR content
    HdrOnPlayback = "HdrOnPlayback",
    // Never Output HDMI Signal in HDR Mode
    DisableHdr = "DisableHdr",
}

enum PictureMode {
    // Standard Mode or Normal Mode
    Standard = "Standard",
    // Dynamic or Vivid Mode
    Dynamic = "Dynamic",
    // Movie or Cinema Mode
    Movie = "Movie",
    // Sports Mode
    Sports = "Sports",
    // Filmmaker Mode
    FilmMaker = "FilmMaker",
    // Game Mode, Lowest Latency Option
    Game = "Game",
    // Automatically adjust the picture mode to the best option for current content
    Auto = "Auto",
}

enum AudioOutputMode {
    // 2 Channel PCM
    Stereo = "Stereo",
    // Multichannel PCM e.g. 5.1 PCM
    MultichannelPcm = "MultichannelPcm",
    // Bitstreams sent in passthrough mode
    PassThrough = "PassThrough",
    // Audio Output Mode Recommended by the device partner in current setup
    Auto = "Auto",
}

enum AudioOutputSource {
    // Audio Output on Native Speakers of the device
    NativeSpeaker = "NativeSpeaker",
    // Audio Output on ARC Port
    Arc = "Arc",
    // Audio Output on E-ARC Port
    EArc = "EArc",
    // Audio Output on Optical Port
    Optical = "Optical",
    // Audio Output on Auxilliary Port
    Aux = "Aux",
    // Audio Output on connected Bluetooth Speaker
    Bluetooth = "Bluetooth",
    // Audio Output in the default mode recommended by the device partner in current setup
    Auto = "Auto",
    // HDMI, This is only applicable for Source devices
    HDMI = "HDMI",
}

enum VideoInputSource {
    // TV Tuner / Antenna
    Tuner = "Tuner",
    // First HDMI Input Port
    HDMI1 = "HDMI1",
    // Second HDMI Input Port
    HDMI2 = "HDMI2",
    // Third HDMI Input Port
    HDMI3 = "HDMI3",
    // Fourth HDMI Input Port
    HDMI4 = "HDMI4",
    // Composite input port
    Composite = "Composite",
    // Component input port
    Component = "Component",
    // TV Home Screen
    Home = "Home",
    // Wireless Casting
    Cast = "Cast",
}

interface ListSupportedSystemSettingsResponse extends DabResponse {
   language: rfc_5646_language_tag [];
   outputResolution: OutputResolution [];
   memc: boolean;
   cec: boolean;
   lowLatencyMode: boolean;
   matchContentFrameRate: MatchContentFrameRate [];
   hdrOutputMode: HdrOutputMode [];
   pictureMode: PictureMode [];
   audioOutputMode: AudioOutputMode [];
   audioOutputSource: AudioOutputSource [];
   videoInputSource: VideoInputSource [];
   audioVolume: {
      min: int;
      max: int;
   },
   mute: boolean;
   textToSpeech: boolean;
}
```
#### Response Parameters

Setting               | Description
--------------------- | -----------
language              | All possible language settings per RFC 5646 language tag
outputResolution      | All possible Video output resolutions for the device
memc                  | Motion estimation and compensation setting support
cec                   | CEC feature support on the device
lowLatencyMode        | Low Latency Mode Setting support
matchContentFrameRate | Match Content FrameRate selection options
hdrOutputMode         | HDR Output Mode options on the Source Device
pictureMode           | Picture modes supported by the device
audioOutputMode       | Audio Output modes supported by the device
audioOutputSource     | Audio output source selection options
videoInputSource      | Video input source options on TV
audioVolume           | Audio output volume control range. If it cannot be controlled by device, set both min and max equal to the current volume level.
mute                  | Volume mute operation support by the device
textToSpeech          | Text to Speech support on the device

### Response Types

Type                  |  Sample Value | Description
--------------------- | ------------- | -----------
list                  |   ["a", "b"]  |  list of all the possible values. Empty list means it's not supported. 
boolean               |      true     |  This is a on/off only setting. `true` means there exists at-least one circumstance where the feature is supported, `false` if not.
number                |       3       |  Usually goes by range with min max pair. Having min>max means it's not supported.

### Sample Response
```json
{
   "status": 200,
   "language": ["en-US", "es"],
   "outputResolution": [{
      "width": 3840,
      "height": 2160,
      "frequency": 60
   },
   {
      "width": 1920,
      "height": 1080,
      "frequency": 60
   }],
   "memc": false,
   "cec": true,
   "lowLatencyMode": true,
   "matchContentFrameRate": ["EnabledAlways", "EnabledSeamlessOnly","Disabled"],
   "hdrOutputMode": ["AlwaysHdr", "HdrOnPlayback", "DisableHdr"],
   "pictureMode": ["Standard", "Dynamic", "Movie"],
   "audioOutputMode": ["Stereo", "Auto"],
   "audioOutputSource": ["HDMI", "Optical"],
   "videoInputSource": [],
   "audioVolume": {
      "min": 0,
      "max": 100
   },
   "mute": false,
   "textToSpeech": true
}
```
### Operation: Getting system settings
*Operation model: Request / Response*

This operation is used to retrieve the current settings on the device.

#### Request Topic

`dab/<device-id>/system/settings/get`

#### Request Format

```typescript
type GetCurrentSystemSettingsRequest = DabRequest
```

#### Response Format
```typescript
interface GetCurrentSystemSettingsResponse extends DabResponse {
   language: rfc_5646_language_tag;
   outputResolution: OutputResolution;
   memc: boolean;
   cec: boolean;
   lowLatencyMode: boolean;
   matchContentFrameRate: MatchContentFrameRate;
   hdrOutputMode: HdrOutputMode;
   pictureMode: PictureMode;
   audioOutputMode: AudioOutputMode;
   audioOutputSource: AudioOutputSource;
   videoInputSource: VideoInputSource;
   audioVolume: int;
   mute: boolean;
   textToSpeech: boolean;
}
```

#### Response Parameters

Setting               | Description
--------------------- | -----------
language              | Current language setting per RFC 5646 language tag
outputResolution      | Current Video output resolution for the device
memc                  | Current Motion estimation and compensation setting used
cec                   | Current CEC feature state on the device
lowLatencyMode        | Current Low Latency Mode Setting used on the device
matchContentFrameRate | Current Match Content FrameRate selected option
hdrOutputMode         | HDR Output Mode as selected on the Source Device
pictureMode           | Picture modes currently used by the device
audioOutputMode       | Audio Output mode currently used by the device
audioOutputSource     | Current Audio output source selected on the device
videoInputSource      | Current Video input source selected on TV
audioVolume           | Current audio output volume
mute                  | Current Volume mute state
textToSpeech          | Current Text to Speech state

#### Sample Response

```json
{
   "status": 200,
   "language": "en-US",
   "outputResolution": {
      "width": 3840,
      "height": 2160,
      "frequency": 60
   },
   "memc": false,
   "cec": true,
   "lowLatencyMode": true,
   "matchContentFrameRate": "EnabledSeamlessOnly",
   "hdrOutputMode": "AlwaysHdr",
   "pictureMode": "Other",
   "audioOutputMode": "Auto",
   "audioOutputSource": "HDMI",
   "videoInputSource": "Other",
   "audioVolume": 20,
   "mute": false,
   "textToSpeech": true
}
```

### Operation: Setting system settings
*Operation model: Request / Response*

This operation is used to update a discreet settings on the device.

#### Request Topic

`dab/<device-id>/system/settings/set`

#### Request Format

```typescript
interface SetSystemSettingsRequest extends DabRequest {
   [system_setting_key: keyof SystemSettings]: [value: any];
}
```

#### Sample Request

```json
{
   "outputResolution": {
      "width": 3840,
      "height": 2160,
      "frequency": 60
   }
}
```

#### Response Format

```typescript
interface SetSystemSettingsResponse extends DabResponse {
   [system_setting_key: keyof SystemSettings]: [value: any];
}
```

#### Sample Response
Success:
```json
{
   "status": 200,
   "outputResolution": {
      "width": 3840,
      "height": 2160,
      "frequency": 60
   }
}
```

Error:
```json
{
   "status": 400,
   "error": "Setting outputResolution does not support value { width: 3840, height: 2160, frequency: 60}"
}
```

## 5.4. Input

### Common key codes

Below is a list of common key codes applicable to all Input operations in this section.

Key Code | Description
--- | ---
KEY_POWER | power on / power off the device
KEY_HOME | go to home screen
KEY_VOLUME_UP | increase the volume
KEY_VOLUME_DOWN | decrease the volume
KEY_MUTE | mute audio
KEY_CHANNEL_UP | next channel
KEY_CHANNEL_DOWN | previous channel
KEY_MENU | go to main menu
KEY_EXIT | exit from current screen
KEY_INFO | display program information
KEY_GUIDE | display guide screen
KEY_CAPTIONS | toggle closed captions/subtitles
KEY_UP | navigation: go up
KEY_PAGE_UP | navigation: page up
KEY_PAGE_DOWN | navigation: page down
KEY_RIGHT | navigation: go right
KEY_DOWN | navigation: go down
KEY_LEFT | navigation: go left
KEY_ENTER | confirm current selection
KEY_BACK | go to the previous screen
KEY_PLAY | start playback
KEY_PLAY_PAUSE | start or pause playback
KEY_PAUSE | pause playback
KEY_RECORD | start recording
KEY_STOP | stop recording
KEY_REWIND | rewind playback (continuous)
KEY_FAST_FORWARD | fast forward playback (continuous)
KEY_SKIP_REWIND | rewind/skip (set increments)
KEY_SKIP_FAST_FORWARD | fast forward (set increments)
KEY_0 | 0 numeric key
KEY_1 | 1 numeric key
KEY_2 | 2 numeric key
KEY_3 | 3 numeric key
KEY_4 | 4 numeric key
KEY_5 | 5 numeric key
KEY_6 | 6 numeric key
KEY_7 | 7 numeric key
KEY_8 | 8 numeric key
KEY_9 | 9 numeric key
KEY_RED | red function button
KEY_GREEN | green function button
KEY_YELLOW | yellow function button
KEY_BLUE | blue function button

If an action is available on the device, it must be mapped to one of the specified key codes.

### Custom keys

The device may choose to accept key codes not specified in the Common key codes section above if they are supported (e.g. vendor buttons). Custom keys must start with `KEY_CUSTOM_` prefix.

### Operation: Get available input keys
*Operation model: Request / Response*

This operation retrieves the list of all the input key codes supported by the device.

#### Request Topic

`dab/<device-id>/input/key/list`

#### Request Format

```typescript
type ListSupportedKeysRequest = DabRequest
```

#### Response Format

```typescript
interface ListSupportedKeysResponse extends DabResponse {
   keyCodes: string [];
}
```
#### Response Parameters

Parameter |  Description
--- | ---
keyCodes | list of key codes supported by the device

#### Sample Response

```json
{
   "status": 200,
   "keyCodes": [
      "KEY_POWER",
      "KEY_HOME",
      "KEY_MENU",
      "KEY_EXIT",
      "KEY_INFO",
      "KEY_UP",
      "KEY_RIGHT",
      "KEY_DOWN",
      "KEY_LEFT",
      "KEY_ENTER",
      "KEY_BACK"
   ]
}
```

### Operation: Key press
*Operation model: Request / Response*

This operation sends a key press to the device. Key press is an action that can be associated with the key press on the remote control. The operational should mimic the same behavior as a key press and release of a remote control. A key code represents button name or function name typically found on the remote control.

#### Request Topic

`dab/<device-id>/input/key-press`

#### Request Format

```typescript
interface SendKeyPressRequest extends DabRequest {
   keyCode: string;
}
```

#### Request Parameters

Parameter | Description
--- | ---
keyCode | string literal, prefixed with `KEY_`, representing the button name / function name, e.g. `KEY_PLAY`. A valid key code consists of a sequence of alpha-numeric characters and/or `_` (underscore) characters. The minimum number of characters is 1 and the maximum is 64.

#### Sample Requests

To navigate left in the menu
```json
{
   "keyCode": "KEY_LEFT"
}
```

To press `Vendor 1` key on the remote. `Vendor 1` is a custom key and its key code starts with `KEY_CUSTOM_`
```json
{
   "keyCode": "KEY_CUSTOM_VENDOR_1"
}
```

#### Response Format

```typescript
type SendKeyPressResponse = DabResponse
```

### Operation:  Long press
*Operation model: Request / Response*

This operation mimics the action of pressing a key for a specified duration.

#### Request Topic

`dab/<device-id>/input/long-key-press`

#### Request Format

```typescript
interface SendLongKeyPressRequest extends DabRequest {
   keyCode: string;
   durationMs: int;
}
```

#### Request Parameters

Parameter | Description
--- | ---
keyCode | string literal, prefixed with `KEY_`, representing the button name / function name, e.g. `KEY_PLAY`. A valid key code consists of a sequence of alpha-numeric characters and/or `_` (underscore) characters. The minimum number of characters is 1 and the maximum is 64.
durationMs | Duration in milliseconds for which the key specified in keyCode must be pressed

#### Sample Request

To press the left key for 3 seconds
```json
{
   "keyCode": "KEY_LEFT",
   "durationMs": 3000
}
```

#### Response Format

```typescript
type SendLongKeyPressResponse = DabResponse
```


## 5.5. Output

Retrieves outputs from the device (such as images, audio, etc.). Images are the only outputs currently supported.

### Operation: Capture image
*Operation model: Request / Response*

This operation captures the current output of the device and returns an image that is base64 encoded. The image must include both graphics plane content and clear (non-DRM) stream video. DRM content and any video that passes through the device's secure video pipeline or trusted execution environment shall not be in the captured image. The image data must be in PNG format.

#### Request Topic

`dab/<device-id>/output/image`

#### Request Format

```typescript
interface CaptureScreenshotRequest = DabRequest
```

#### Response Format

```typescript
interface CaptureScreenshotResponse extends DabResponse {
   outputImage: string;
}
```

#### Response Parameters

Parameter |  Description
--- | ---
outputImage | base64 encoded PNG image in a [data URL](https://www.rfc-editor.org/rfc/rfc2397)

#### Sample Response

```json
{
   "outputImage": "data:image/png;base64,iVBORw0KGgo..."
}
```

## 5.6. *Optional* Device and Application Telemetry

Device and application telemetry allows the connected clients to gather metrics about the device and the applications running on the device.
To start or stop the telemetry, either for the device or for the specific application, a request must be made to the correct request topic.

Once the telemetry is started the device will start publishing metrics to the assigned telemetry notification topic until requested to stop. Telemetry is stateless, and it is expected that start telemetry will have to be requested again after a device restart.


### Operation: Starting Device Telemetry
*Operation model: Request / Response*

#### Request Topic

`dab/<device-id>/device-telemetry/start`

#### Request Format

```typescript
interface StartDeviceTelemetryRequest extends DabRequest {
   duration: number;
}
```

#### Request Parameters

Parameter | Default|  Description
--- | --- | ---
duration | - | desired pull duration for telemetry updates in milliseconds

#### Sample Request

A request to start telemetry for the device with the pull duration of 500 milliseconds
```json
{
   "duration": 500
}
```

#### Response Format

```typescript
interface StartDeviceTelemetryResponse extends DabResponse {
   duration: number;
}
```

#### Response Parameters

Parameter |  Description
--- | ---
duration | actual duration between pull requests device is capable of for telemetry updates

#### Sample Response

Telemetry started with the actual duration of 1 second
```json
{
   "status": 200,
   "duration": 1000
}
```

### Operation: Stopping Device Telemetry
*Operation model: Request / Response*

#### Request Topic

`dab/<device-id>/device-telemetry/stop`

#### Request Format

```typescript
interface StopDeviceTelemetryRequest extends DabRequest {
}
```

#### Response Format

```typescript
type StopDeviceTelemetryResponse = DabResponse
```


### Operation: Starting Application Telemetry
*Operation model: Request / Response*

#### Request Topic

`dab/<device-id>/app-telemetry/start`

#### Request Format

```typescript
interface StartApplicationTelemetryRequest extends DabRequest {
   appId: string;
   duration: number;
}
```

#### Request Parameters

Parameter | Default|  Description
--- | --- | ---
appId | - | application identifier
duration | - | desired duration for telemetry updates in milliseconds

#### Sample Request

A request to start telemetry for the application `media-player-1` with the duration of 5 seconds
```json
{
   "appId": "media-player-1",
   "duration": 5000
}
```

#### Response Format

```typescript
interface StartApplicationTelemetryResponse extends DabResponse {
   duration: number;
}
```

#### Response Parameters

Parameter |  Description
--- | ---
duration | actual duration device is capable of for telemetry updates

#### Example

Telemetry started with the actual duration of 1 second
```json
{
   "status": 200,
   "duration": 1000
}
```

### Operation: Stopping Application Telemetry
*Operation model: Request / Response*

#### Request Topic

``dab/<device-id>/app-telemetry/stop``

#### Request Format

```typescript
interface StopApplicationTelemetryRequest extends DabRequest {
   appId: string;
}
```

#### Request Parameters

Parameter | Description
--- | ---
appId | Application identifier

#### Sample Request

To stop the telemetry for the application `media-player-2`:
```json
{
   "appId": "media-player-2"
}
```

#### Response Format

```typescript
type StopApplicationTelemetryResponse = DabResponse
```

### Telemetry Notification Topics

Scope | Notification Topic | Description
--- | --- | ---
Device | `dab/<device-id>/device-telemetry/metrics` | -
Application | `dab/<device-id>/app-telemetry/metrics/<appId>` | where `<appId>` is the application ID

#### Example

The application identified in the system by `media-player-1` will have its metrics delivered to the notification topic `dab/app-telemetry/metrics/media-player-1`

### Telemetry Metrics

List of supported metrics

| Metric  | Unit | Description | Example
|---|---|---|---|
| cpu | percentage expressed as a number | *optional* Overall CPU usage scaled to the range of 0% to 100%| 26
| memory | kilobytes | *optional* Memory usage | 43008

If an application is not running and the app specific telemetry collection is ongoing or gets started, the device should return 0 as the value for both cpu and memory metric.

### Message Format

```typescript
interface TelemetryMessage {
   timestamp: unix_timestamp_ms;
   metric: string;
   value: number;
}
```

### Message Parameters

Parameter | Description
--- | ---
timestamp | UNIX Epoch time in milliseconds
metric | metric name
value | value of the metric

### Sample Messages

```json
{
   "timestamp": 1568815387403,
   "metric": "cpu",
   "value": 33.5
}
```

```json
{
   "timestamp": 1568815390414,
   "metric": "memory",
   "value": 44040192
}
```

## 5.7. Health Check

### Operation: Run a health check
*Operation model: Request / Response*

This operation checks whether DAB is enabled and operational on the device.

#### Request Topic

`dab/<device-id>/health-check/get`

#### Request Format

```typescript
type CheckDeviceHealthRequest = DabRequest
```

#### Response Format

```typescript
interface CheckDeviceHealthResponse extends DabResponse {
   healthy: boolean;
   message?: string;
}
```

#### Response Parameters

Parameter | Description
--- | ---
healthy | `true` when DAB is enabled and operational, `false` otherwise
message | optional message to indicate any problems with the DAB protocol

#### Sample Responses

```json
{
   "status": 200,
   "healthy": true
}
```

```json
{
   "status": 200,
   "healthy": false,
   "message": "Internal DAB error"
}
```

## 5.8 *Optional* General Notifications

The device may publish any errors, warnings, general information to the general notification topic. These messages provide developers an internal observability into the state of DAB on the device and help debug potential issues on the platform. While not required, it is highly suggested to at-least provide retained MQTT messages informing clients about the state of DAB as shown in the examples below.

### Notification Topic

`dab/<device-id>/messages`

### Notification Format

```typescript
type NotificationLevel = "info" | "warn" | "debug" | "trace" | "error";

interface Notification {
   timestamp: unix_timestamp_ms;
   level: NotificationLevel;
   message: string;
}
```

### Notification Levels

Notification Level | Description
--- | ---
`info` | A message at coarse-grained level
`warn` | A message indicating a warning, potentially problematic event
`debug` | A message intended to help with the debugging of the problem
`trace` | A message at fine-grained level, low-level details
`error` | A message indicating an error event

### Notification Parameters
Parameter | Description
--- | ---
timestamp | UNIX Epoch time in milliseconds
level | Notification level
message | Message

#### Suggested Notifications

```json
{
   "timestamp": 1610704569349,
   "level": "info",
   "message": "DAB started successfully and is online."
}
```

```json
{
   "timestamp": 1610704569349,
   "level": "error",
   "message": "DAB faced an exception. Terminating abnormally."
}
```

## 5.9. Voice

This section applies to all devices where users may interact with the device using a microphone. 

### Voice Systems

A voice system can be any platform assistant or speech recognizer that accepts audio as input.

### Operation: Listing all available voice systems
*Operation model: Request / Response*

This operation lists the available voice systems installed on the device.

#### Request Topic

`dab/<device-id>/voice/list`

#### Request Format

```typescript
type ListSupportedVoiceSystemsRequest = DabRequest
```

#### Response Format

```typescript
interface VoiceSystem {
   name: string;
   enabled: boolean;
}
```

Parameter | Description
--- | ---
name | Name of the voice system
enabled | 'true' when the voice system is enabled on the platform, 'false' otherwise

```typescript
interface ListSupportedVoiceSystemsResponse extends DabResponse {
   voiceSystems: VoiceSystem [];
}
```
#### Response Parameters

Parameter | Description
--- | ---
voiceSystems | a list of available voice systems supported by the device, as [defined in the DAB Voice System Registry](#voice-system-name-registry)

#### Sample Response
```json
{
   "status": 200,
   "voiceSystems": [
      {
         "name": "AmazonAlexa",
         "enabled": true
      },
      {
         "name": "GoogleAssistant",
         "enabled": false
      },
      {
         "name": "Siri",
         "enabled": false
      }
   ]
}
```

### Operation: Setting enabled voice system(s)
*Operation model: Request / Response*

This operation is used to change the currently enabled voice system(s). The platform shall determine whether multiple voice systems may be enabled concurrently.

#### Request Topic

`dab/<device-id>/voice/set`

#### Request Format

```typescript
interface SetVoiceSystemRequest extends DabRequest {
   voiceSystem: VoiceSystem;
}
```

#### Sample Request

```json
{
   "voiceSystem": {
      "name": "GoogleAssistant",
      "enabled": true
   },
}
```

#### Response Format

```typescript
interface SetVoiceSystemResponse extends DabResponse {
   voiceSystem: VoiceSystem;
}
```

#### Sample Response
Success:
```json
{
   "status": 200,
   "voiceSystem": {
      "name": "GoogleAssistant",
      "enabled": true
   },
}
```

Error:
```json
{
   "status": 400,
   "error": "Setting voiceSystem failed"
}
```

### Operation: Sending audio to voice system
*Operation model: Request / Response*

This operation provides an HTTP URL identifying an audio file of MIME type `audio/wav` to be used for a voice request. The platform must issue a GET request to the URL and appropriately route the retrieved audio through the platform as if it came from the device's microphone. 

Audio must be a WAV format file containing 16-bit linear PCM, 16-kHz sample rate, single channel, little-endian byte order format. The sample rate must be contained in the header.

File size must not exceed 4MB. 

#### Request Topic

`dab/<device-id>/voice/send-audio`

#### Request Format

```typescript
interface SendVoiceAsAudioRequest extends DabRequest {
   fileLocation: url;
   voiceSystem?: string;
}
```

#### Request Parameters

Parameter | Default|  Description
--- | --- | ---
fileLocation | - | HTTP URL identifying the location of the audio resource for the platform to download.
voiceSystem | - | _Optional_ Specific voice system to direct the audio to. Voice system must be enabled first. As [defined in the DAB Voice System Registry](#voice-system-name-registry)

#### Sample Request

```json
{
   "fileLocation": "http://192.168.1.8:80/utterance_1.wav",
   "voiceSystem": "AmazonAlexa"
}
```

#### Response Format

```typescript
type SendVoiceAsAudioResponse = DabResponse
```

#### Sample Responses

```json
{
   "status": 200
}
```

```json
{
   "status": 400,
   "error": "File not available at fileLocation"
}
```

### Operation: Sending text to voice system
*Operation model: Request / Response*

This operation provides a text string to be used for a simulated voice request. The platform must treat the text string as if it were the transcribed output of a traditional, microphone-initiated voice request.

#### Request Topic

`dab/<device-id>/voice/send-text`

#### Request Format

```typescript
interface SendVoiceAsTextRequest extends DabRequest {
   requestText: string;
   voiceSystem: string;
}
```

#### Request Parameters

Parameter | Default|  Description
--- | --- | ---
requestText | - | The string to be passed as output of a voice system
voiceSystem | - | Specific voice system to direct the audio to.  Voice system must be enabled first.

#### Sample Request

```json
{
   "requestText": "Play Ed Sheeran on YouTube",
   "voiceSystem": "GoogleAssistant"
}
```

#### Response Format

```typescript
type SendVoiceAsTextResponse = DabResponse
```

#### Sample Responses

```json
{
   "status": 200
}
```

```json
{
   "status": 501,
   "error": "Not implemented"
}
```

### Operation: Discover existing active device
*Operation model: Request / Response*

This operation provides a way for user client to discover devices under the same MQTT broker. For each DAB capable device a device discovery response must be provided.

#### Request Topic

`dab/discovery`

#### Request Format

```typescript
type DiscoverDevicesRequest = DabRequest
```

#### Response Format

```typescript
interface DiscoverDevicesResponse extends DabResponse {
   ip: string;
   deviceId: string;
}
```

#### Response Parameters

Parameter | Description
--- | ---
ip | IP Address of the device.
deviceId | The unique identifier of this device. Should be same as <device-id> in other topics.

#### Sample Responses

```json
{
   "status": 200,
   "ip": "192.168.1.119",
   "deviceId": "device3"
}
```

# 6. Versioning of the protocol
*Operation model: Request / Response*

This operation allows the use clients to know the DAB version implemented on a device client. It is possible for a device to implement more than one version of the DAB protocol.

## Operation: Get Protocol Version

A protocol version consists of the major version and the minor version delimited by a full stop character `.`
Major and minor versions are non-negative integers.

Examples:

Valid versions:
`1.0`, `2.3`, `0.48`

Invalid versions
`1.a`,  `x.5`

### Request Topic

`dab/<device-id>/version`

#### Request Format

```typescript
type GetDabVersionRequest = DabRequest
```

### Response Format

```typescript
interface GetDabVersionResponse extends DabResponse {
   versions: string [];
}
```
#### Response Parameters

Parameter | Description
--- | ---
versions | a list of versions of the protocol implemented on the device


#### Sample Response

```json
{
   "status": 200,
   "versions": ["2.0"]
}
```


# 7. Appendix

## Application ID Registry
| Application        | DAB appId         |
| :----------------: | :---------------: |
| Netflix            | Netflix           |
| YouTube            | YouTube           |
| Amazon Prime Video | PrimeVideo        |
| BBC iPlayer        | uk.co.bbc.iplayer |
| YouTube TV         | YouTubeTV         |

## Voice System Name Registry

| Voice System       |  DAB Voice System Name  |
| :---------------:  |  :-------:              |
| Google Assistant   |   GoogleAssistant       |
| Amazon Alexa       |   AmazonAlexa           |
| LG ThinQ           |   LGThinQ               |
| Yandex Alice       |   YandexAlice           |
| TiVo Conversation  |   TiVoConversation      |
| WhaleAIVoice       |   WhaleAIVoice          |

### Process for Contributing to Registry

 If you are an application or voice system owner and your identifier is not found in the registry, you can register it by following these steps:

1. Fork this specification repository
2. Add and commit your application or voice system identifier in your repo
3. Create a pull request to merge your additions to this DAB specification
4. A DAB consortium member will review your pull request, approve, and merge promptly

You can see more instructions on how to [create a pull request from a forked repository on GitHub using these instructions.](https://docs.github.com/en/pull-requests/collaborating-with-pull-requests/proposing-changes-to-your-work-with-pull-requests/creating-a-pull-request-from-a-fork)


# 8. Deprecated APIs

### Operation: Listing available languages

This operation lists the available languages on the device.

All language operations apply to the device language. The device language is defined as the language setting that will be provided to applications as the device locale. All language codes used are defined in the [RFC 5646](https://tools.ietf.org/html/rfc5646) specification.

#### Request Topic

`dab/system/language/list`

#### Request Format

```typescript
type GetAvailableLanguagesRequest = DabRequest
```

#### Response Format
```typescript
interface GetAvailableLanguagesResponse extends DabResponse {
	languages: rfc_5646_language_tag [];
}
```

#### Sample Response

```json
{
   "status": 200,
   "languages": [
      "en-GB",
      "en-US",
      "fr"
   ]
}
```

### Operation: Getting the currently set language

This operation returns the currently set language on the device.

#### Request Topic

`dab/system/language/get`

#### Request Format

```typescript
type GetCurrentLanguageRequest = DabRequest
```

#### Response Format

```typescript
interface GetCurrentLanguageResponse extends DabResponse {
   language: rfc_5646_language_tag;
}
```

Parameter | Description
--- | ---
language | language that is currently set on the device

#### Sample Response

Success:

```json
{
   "status": 200,
   "language": "en-US"
}
```

### Operation: Setting the language

This operation is used to update the language on the device. The language tag being set by the set topic must match a language tag in the list returned by the `dab/system/language/list` topic.

#### Request Topic

`dab/system/language/set`

#### Request Format

```typescript
interface SetLanguageRequest extends DabRequest {
   language: rfc_5646_language_tag;
}
```
#### Response Format

```typescript
type SetLanguageResponse = DabResponse
```

#### Sample Response

```json
{
   "status": 200
}
```
