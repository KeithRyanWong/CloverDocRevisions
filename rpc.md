# Cloud Based Semi-integration: Clover Connector Tutorial
## **The Clover Connector SDK**

The Clover Connector Browser SDK provides an interface that enables your point-of-sale (POS) software to interact with Clover's customer-facing payment devices through the cloud to complete actions such as: processing credit cards, displaying order details, etc.

> ### Prerequisites
> This tutorial assumes you have already:
> * Read the [Overview of the Clover Platform][2], including the [Developer Guidelines][3].
> * [Set up a developer account][4] (including your test merchant settings).
> * [Ordered a Clover Developer Kit (DevKit)][5] and [set it up][6].
> * [Created a Remote App ID][7] (`remoteApplicationID`) for your semi-integration POS. This is different from your App ID.
> * Installed the Secure Network Pay Display app or Cloud Pay Display app on the Clover device. The Secure Network Pay Display app is currently in Beta. If you’re interested in using a local network connection, please contact Clover’s Developer Relations team at semi-integrations@clover.com.
> * Installed NPM on your computer.


The SDK can be broken down into two main structures: the **CloverConnector** and the **CloverConnectorListener**. Complete documentation can be found [here][1].

---

## The CloverConnector
The CloverConnector is the interface which the developer will use to issue commands to the Clover device. 

To set up this interface, the SDK provides a factory method that takes in a configuration object and returns a CloverConnector. The first thing that we need to do is create the configuration object, (there's a little bit of a disconnect between the configuration settings and the interfaces of the clover device the names correspond to the actual application but aren't really descriptive of the connection, mostly because of the display part) which will specify one of two ways the connector will use to connect to the Clover device:

> ### a. Secure Network Pay Display (Direct Connection through Local Network)
> (Can the adv and disadbv sections be collapsible? if so, then maybe having an extra bit of explanation here would solve the problem mentioned above)
> > #### Advantages 
> > - Performance – A direct connection will perform better than a cloud connection, which proxies requests through Clover’s servers.
> > - Simple connection configuration – The configuration for a direct connection is less complex.
> > #### Disadvantages
> > - Your POS users will be required to install and trust the Clover device server certificate in each browser that runs the POS.

> ### b. Cloud Pay Display (Connection through Clover Servers) 
> > #### Advantages
> > - Flexibility – Cloud connections support connecting to devices that are not on the same network as the POS.
> > - No certificate requirements – The cloud connects to the device through Clover’s servers, which eliminates the need for browsers running the POS to install and trust the Clover device server certificate.
> > #### Disadvantages
> > - Performance – Because the cloud proxies requests through Clover’s servers, it may not perform as well as a direct connection to the Clover device.
> > - More complex connection configuration – The connection configuration for the cloud is more complex than the configuration for a direct connection.

### Step 1. Specify a configuration type and its parameters
Use the CloverDeviceConfiguration constructor to create a configuration object. It takes in a number of different parameters(specific to connection type) and it might be best to get them sorted out before jumping into creating the configuration.
```javascript 
const clover = require ("remote-pay-cloud");
var connectorConfiguration = new clover.WebSocketPairedCloverDeviceConfiguration([params...]);
```
a. Secure Network Pay Display params and setup
> (I think it would be great if all the steps could be collapsed, giving a higher level overview of the process)
> #### Parameters
> 1. `Application ID` – The ID that uniquely identifies the POS (e.g. com.company.MyPOS:2.3.1). To retrieve your Application ID, see step 5 on the Create Your Remote App ID page.
> 2. `webSocketFactory` – A function that returns an instance of the CloverWebSocketInterface that will be used when connecting. For browser implementations, this can be defined as BrowserWebSocketImpl.createInstance. For Node.JS implementations, define the function as heartbeatInterval: number = 1000;.
> 3. `imageUtil` – A utility for translating images into base64 strings.
> 4. `heartbeatInterval` – How long to wait for a PING before disconnecting, in milliseconds.
> 5. `reconnectDelay` – How long to wait before attempting to reconnect, in milliseconds.
> 6. `endpoint` – The endpoint of the Clover device (e.g. ws://192.168.1.15:12345/remote_pay). This endpoint is displayed on the screen that appears when you first launch the Secure Network Pay Display app.
> 7. `posName` – The POS name the Secure Network Pay Display app shows on the Clover device during the pairing process (e.g. MyPOS).
> 8. `serialNumber` – The serial number/identifier for the POS, as displayed in the Secure Network Pay Display app. Note: This is not the same as the Clover device’s serial number.
> 9. `authToken` – The authToken retrieved from a previous pairing response, passed as an argument to onPairingSuccess. This will be null for the first connection. Store and use the authToken in subsequent requests to skip pairing.
> const clover = require ("remote-pay-cloud");
> ```javascript
> var connectorConfiguration = new clover.WebSocketPairedCloverDeviceConfiguration(
>    "wss://192.168.0.20:12345/remote_pay", // endpoint
>        "com.cloverconnector.javascript.simple.sample:1.4", // applicationId
>        "Javascript Simple Sample", // posName
>        "Register_1", // serialNumber
>        null, // authToken
>        clover.BrowserWebSocketImpl.createInstance, // webSocketFactory
>        new clover.ImageUtil(), // imageUtil
>        1000, // heartbeatInterval
>        3000); // reconnectDelay
>
>
> networkConfiguration = Object.assign(connectorConfiguration, {
>    onPairingCode: function (pairingCode) {
>         console.log(`Pairing code is  ${pairingCode}`);
>    },
>    onPairingSuccess: function (authToken) {
>        console.log(` Pairing succeeded, authToken is : ${authToken}`);
>    }
> });
>```


[1]: https://clover.github.io/remote-pay-java/1.4.0/docs/index.html?com/clover/remote/client/WebSocketCloverDeviceConfiguration.html
[2]: https://docs.clover.com/build/architecture/
[3]: https://docs.clover.com/build/developer-guidelines/
[4]: https://www.clover.com/developers
[5]: https://cloverdevkit.com/
[6]: https://docs.clover.com/build/devkit/
[7]: https://docs.clover.com/build/create-your-remote-app-id/