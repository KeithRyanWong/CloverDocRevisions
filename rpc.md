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


The SDK can be broken down into two main structures: the **ICloverConnector** and the **ICloverConnectorListener**. Complete documentation can be found [here][1].

---

## The ICloverConnector
The ICloverConnector is the interface which the developer will use to issue commands to the Clover device. 

To set up this interface, the SDK provides a factory method that takes in a configuration object and returns a CloverConnector. The first thing that we need to do is create the configuration object, (there's a little bit of a disconnect between the configuration settings and the interfaces of the clover device the names correspond to the actual application but aren't really descriptive of the connection, mostly because of the display part) which will specify one of two ways the connector will use to connect to the Clover device:

> ### a. Secure Network Pay Display (Direct Connection through Local Network)
> (Can the adv and disadbv sections be collapsible? If so, then maybe having an extra bit of explanation here would solve the problem mentioned above)
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
Use the `WebSocketPairedCloverDeviceConfiguration` constructor to create a configuration object. It takes in a number of different parameters(specific to connection type) and it might be best to get them sorted out before jumping into creating the configuration.
```javascript 
const clover = require ("remote-pay-cloud");
var connectorConfiguration = new clover.WebSocketPairedCloverDeviceConfiguration([params...]);
```
a. Secure Network Pay Display params and setup
> (I think it would be great if all the steps could be collapsed, giving a higher level overview of the process)
> #### Parameters
> 1. `Application ID` – The ID that uniquely identifies the POS (e.g. com.company.MyPOS:2.3.1). To retrieve your Application ID, see step 5 on the Create Your Remote App ID page.
> 2. `webSocketFactory` – A function that returns an instance of the CloverWebSocketInterface that will be used when connecting. For browser implementations, this can be defined as BrowserWebSocketImpl.createInstance. For Node.JS implementations, you will need to implement an object that extends the [CloverWebSocketInterface][9]. [Example can be found here][8].
> 3. `imageUtil` – A utility for translating images into base64 strings.
> 4. `heartbeatInterval` – How long to wait for a PING before disconnecting, in milliseconds.
> 5. `reconnectDelay` – How long to wait before attempting to reconnect, in milliseconds.
> 6. `endpoint` – The endpoint of the Clover device (e.g. ws://192.168.1.15:12345/remote_pay). This endpoint is displayed on the screen that appears when you first launch the Secure Network Pay Display app.
> 7. `posName` – The POS name the Secure Network Pay Display app shows on the Clover device during the pairing process (e.g. MyPOS).
> 8. `serialNumber` – The serial number/identifier for the POS, as displayed in the Secure Network Pay Display app. Note: This is not the same as the Clover device’s serial number.
> 9. `authToken` – The authToken retrieved from a previous pairing response, passed as an argument to onPairingSuccess. This will be null for the first connection. Store and use the authToken in subsequent requests to skip pairing.
> #### Setup
> ```javascript
> const clover = require ("remote-pay-cloud");
> //...
> var connectorConfiguration = new clover.WebSocketPairedCloverDeviceConfiguration(
>        "wss://192.168.0.20:12345/remote_pay", // endpoint
>        "com.cloverconnector.javascript.simple.sample:1.4", // applicationId
>        "Javascript Simple Sample", // posName
>        "Register_1", // serialNumber
>        null, // authToken
>        clover.BrowserWebSocketImpl.createInstance, // webSocketFactory
>        new clover.ImageUtil(), // imageUtil
>        1000, // heartbeatInterval
>        3000 // reconnectDelay
> ); 
>
>
> var SNPDConfiguration = Object.assign(connectorConfiguration, {
>    onPairingCode: function (pairingCode) {
>         console.log(`Pairing code is  ${pairingCode}`);
>    },
>    onPairingSuccess: function (authToken) {
>        console.log(` Pairing succeeded, authToken is : ${authToken}`);
>    }
> });
>```
b. Cloud Pay Display params and setup
> #### Parameters
> 1. `Application ID` – The ID that uniquely identifies the POS (e.g. com.company.MyPOS:2.3.1). To retrieve your Application ID, see step 5 on the Create Your Remote App ID page.
> 2. `webSocketFactory` – A function that returns an instance of the CloverWebSocketInterface that will be used when connecting. For browser implementations, this can be defined as BrowserWebSocketImpl.createInstance. For Node.JS implementations, you will need to implement an object that extends the [CloverWebSocketInterface][9]. [Example can be found here][8].
> 3. `imageUtil` – A utility for translating images into base64 strings.
> 4. `heartbeatInterval` – How long to wait for a PING before disconnecting, in milliseconds.
> 5. `reconnectDelay` – How long to wait before attempting to reconnect, in milliseconds.
> 6. cloverServer – The base URL for the Clover server used in the cloud connection (e.g. https://www.clover.com, https://sandbox.dev.clover.com/).
> 7. accessToken – The steps for obtaining your accessToken are available on the OAuth 2.0 page.
> 8. httpSupport – The helper object used when making  HTTP requests.
> 9. merchantId – The steps for finding your merchantId are available on the Merchant ID and API Token for Development page.
> - 9a. deviceId – To obtain the deviceId, you must first retrieve an accessToken and your merchantId.
> - 9b. Then, make the following GET request to the Clover API: https://{cloverServer}/v3/merchants/{merchantId}/devices?access_token={accessToken}
> 10. friendlyId – An identifier for the specific terminal connected to this device. This ID is used in debugging, and may be sent to other clients if they attempt to connect to the same device. Clover will also send it to other clients that are currently connected if the device does a forceConnect.
> 11. forceConnect – If true, overtakes any existing connection.
> #### Setup
> ```javascript
> const clover = require ("remote-pay-cloud");
> //...
> var connectorConfiguration = new clover.WebSocketCloudCloverDeviceConfiguration(
>    "com.mycompany.app.remoteApplicationId:0.0.1", // applicationId      
>    clover.BrowserWebSocketImpl.createInstance, // webSocketFactory  
>    new clover.ImageUtil(), // imageUtil
>    "https://sandboxdev.dev.clover.com/", // clover server     
>    "7aacbd26-d900-8e6a-80f0-ae55f5d1cae2", // accessToken
>    new clover.HttpSupport(XMLHttpRequest), // httpSupport
>    "VKYQ0RVGMYHRR", // merchantId
>    "f0f00afd4bc9c55ef0733e7cb8c3da97", // deviceId
>    "testTerminal1_connection1", // friendlyId
>    true, // forceConnect
>    1000, // heartbeatInterval
>    3000 // reconnectDelay
> );
> ```

### Step 2. Specify a Factory

```javascript
const clover = require ("remote-pay-cloud");
//...
var builderConfiguration = {};
builderConfiguration[clover.CloverConnectorFactoryBuilder.FACTORY_VERSION] = clover.CloverConnectorFactoryBuilder.VERSION_12;

var cloverConnectorFactory = clover.CloverConnectorFactoryBuilder.createICloverConnectorFactory(builderConfiguration);
```
### Step 3. Build a ICloverConnector
```javascript
//...
var cloverConnector = cloverConnectorFactory.createICloverConnector(connectorConfiguration);
```

### Step 4. Build a ICloverConnectorListener
So now we've got a CloverConnector, through which we can issue commands to the Clover device, such as `cloverConnector.initializeConnection();`, but how can we build a program that reacts to anything that it does or signals? For that, we'll need to build a CloverConnectorListener, which is really just a collection of callback functions.

We'll overwrite some of the functions from the `clover.remotepay.ICloverConnectorListener.prototype` to make our own custom listener. A complete list of events can be found [here][10]. For now, we'll just get ready to connect.
```javascript
const clover = require ("remote-pay-cloud");
//...

var defaultCloverConnectorListener = Object.assign({}, clover.remotepay.ICloverConnectorListener.prototype, {

   onDeviceReady: function (merchantInfo) {
       updateStatus("Pairing successfully completed, your Clover device is ready to process requests.");
       console.log({message: "Device Ready to process requests!", merchantInfo: merchantInfo});
   },

   onDeviceDisconnected: function () {
       console.log({message: "Disconnected"});
   },

   onDeviceConnected: function () {
       console.log({message: "Connected, but not available to process requests"});
   }

});
```
> NOTE: 
> onDeviceReady() may be called multiple times.  For this reason, we do not recommended initiating sales or other transactions from onDeviceReady().

### Step 5. Attach the ICloverConnectorListener to the ICloverConnector

```javascript
cloverConnector.addCloverConnectorListener(defaultCloverConnectorListener);
```

### Step 6. Initialize the connection
The connection has been all set up, but before calling any methods on the ICloverConnector, we need to call the following method to initialize the connection:

```javascript
cloverConnector.initializeConnection();
```

### You're all set! 
Test out the connection with the following code:
```javascript
cloverConnector.showMessage("Welcome to Clover Connector!");
```
Happy coding!

[1]: https://clover.github.io/remote-pay-java/1.4.0/docs/index.html?com/clover/remote/client/WebSocketCloverDeviceConfiguration.html
[2]: https://docs.clover.com/build/architecture/
[3]: https://docs.clover.com/build/developer-guidelines/
[4]: https://www.clover.com/developers
[5]: https://cloverdevkit.com/
[6]: https://docs.clover.com/build/devkit/
[7]: https://docs.clover.com/build/create-your-remote-app-id/
[8]: https://github.com/clover/remote-pay-cloud-examples/blob/master/remote-pay-cloud-nodejs-example/lib/support/ExampleWebSocketFactory.js
[9]: http://clover.github.io/remote-pay-cloud/1.4.1/classes/_websocket_cloverwebsocketinterface_.cloverwebsocketinterface.html
[10]: https://clover.github.io/remote-pay-cloud-api/1.4.1/remotepay.ICloverConnectorListener.html