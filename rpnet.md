# Windows-based .NET Semi-integration: Clover Connector Tutorial
## **The Clover Connector SDK**

The Clover Connector Browser SDK provides an interface that enables your point-of-sale (POS) software to interact with Clover's customer-facing payment devices through the cloud to complete actions such as: processing credit cards, displaying order details, etc.

### Prerequisite steps
This tutorial assumes you have already:
* Read the [Overview of the Clover Platform][2], including the [Developer Guidelines][3].
* [Set up a developer account][4] (including your test merchant settings).
* [Ordered a Clover Developer Kit (DevKit)][5] and [set it up][6].
* [Created a Remote App ID][7] (`remoteApplicationID`) for your semi-integration POS. This is different from your App ID.
* Installed the [USB Pay Display app][11] on the Clover device. You can also use the [Secure Network Pay Display app][12] for a local network connection (currently available for Clover Mini and Clover Mobile).
* Installed [Visual Studio][13] on your computer.

> ## **Installing and Using the Clover Connector SDK in your project**
>     Note: NET 4.5 supports both wss (secured endpoints) and ws (unsecured). NET 4.0 only supports ws.
> 1. Download the latest version of CloverSDKSetup.exe
>     - NOTE: downloadable files for the SDK are found here: https://github.com/clover/remote-pay-windows/releases
>     - Executable may be in a zipped file
> 2. Run the setup, it will install the necessart .DLL's on your computer, keep track of their location
> 3. Add the libraries to your project references.
>     - Libraries needed for connecting: remotepay.transport, remotepay.sdk
> > ```C
> > using com.clover.remotepay.transport;
> > using com.clover.remotepay.sdk;
> > ```

The SDK can be broken down into two main structures: the **ICloverConnector** and the **ICloverConnectorListener**. Complete documentation can be found [here][1].

---

## The ICloverConnector
The ICloverConnector is the interface which the developer will use to issue commands to the Clover device. This interface can be initiated with one of two configurations that specifies which Clover application it will use to launch payments on the Clover device. The two applications and their corresponding configurations are: USB Pay Display (USBCloverDeviceConfiguration) and Secure Network Pay Display (WebSocketCloverDeviceConfiguration).

## The ICloverConnectorListener
The ICloverConnectorListener is an object that attaches to the ICloverConnector and allows a program to respond to actions and events from the Clover Device. It is basically a collection of callback functions.

## **Option A: Steps for connecting to a Clover through the USB Pay Display App**
> ### **Step A.1. Build a CloverConnector**
> > An ICloverConnector can be instantiated from its constructor that takes in a USBCloverDeviceConfiguration object. The two important parameters of the config object are the device ID of the Clover you are targeting and the remote application ID you made for your application.
> > > Note: The CloverConnector has logging capabiltiies that may be useful for debugging. You can turn this feature on or off by passing a boolean into your device config object.
> > ``` C
> >var app_id = "com.AOELabs.netpay";
> >var device_id = "c2226a83-d37d-3957-0d54-4470dbc0275b";
> > 
> >//Turn on logging by setting USBCloverDeviceConfiguration(_, _, true, _)
> >var usbConfiguration = new USBCloverDeviceConfiguration(device_id, app_id, false, 1);
> > 
> >CloverConnector cloverConnector = new CloverConnector(usbConfiguration);
> > ```
> ### **Step A.2. Build a ICloverConnectorListener**
> > An ICloverConnector can be created by extending the DefaultCloverConnectorListener.
> > Define the following class and fill it in with whatever you want!
> > ``` C
> > public class ExampleCloverConnectionListener : DefaultCloverConnectorListener
> >        {
> >            public ExampleCloverConnectionListener(ICloverConnector cloverConnector) : base(cloverConnector)
> >            {
> >
> >            }
> >            public override void OnDeviceReady(MerchantInfo merchantInfo)
> >            {
> >                base.OnDeviceReady(merchantInfo);
> >                Console.WriteLine("We're ready!");
> >            }
> >
> >            public override void OnDeviceConnected()
> >            {
> >                base.OnDeviceConnected();
> >                Console.WriteLine("We're Connected, still waiting for ready though!");
> >            }
> >
> >            public override void OnDeviceDisconnected()
> >            {
> >                base.OnDeviceDisconnected();
> >                Console.WriteLine("We're gone!");
> >            }
> >
> >            public override void OnConfirmPaymentRequest(ConfirmPaymentRequest request)
> >            {
> >                throw new NotImplementedException();
> >            }
> >
> >            public override void OnSaleResponse(SaleResponse response)
> >            {
> >                if (response.Result == ResponseCode.FAIL)
> >                {
> >                    Console.WriteLine($"Error: {response.Message}");
> >                    Thread.Sleep(10000);
> >                }
> >                base.OnSaleResponse(response);
> >            }
> >        }
> > ```
> ### **Step A.3. Make an instance of your listener and attach it to the ICloverConnector**
> > ``` C
> > var ccl = new ExampleCloverConnectionListener(cloverConnector);
> >
> > cloverConnector.AddCloverConnectorListener(ccl);
> > ```
> ### **Step A.4. Initialize the connection**
> > You must initialize the connection before making any requests. 
> > ``` C
> > cloverConnector.InitializeConnection();
> > ```

## **Option B: Steps for connecting to a Clover through the Secure Network Pay Display (SNPD) App**
Connecting to a Clover through SNPD involves an initial pairing process that can be confusing to go through. So first, why don't we go through a quick run-through of the process. Read through it but don't try to implement anything just yet, as the steps will make more sense once we've actually built out your application.
> Note: These steps assume that you've correctly built the application to connect using the SDK. The order of these steps matters, proceed chronically.

> Note: NET 4.5 supports both wss (secured endpoints) and ws (unsecured). NET 4.0 only supports ws.
### <a id="dpoverview"></a>**Device Pairing overview** (**IMPORTANT - read before proceding**)
1. Download SNPD onto your device and open it up. It should pull down an additional application (Device Server), which is a dependency of SNPD.
    - a. Turn secure endpoints on or off depending on the NET framework used in your application.
    - b. Turn on Enable Developer Certficates
    - c. Configure and Restart your device server 
2. Configure your application to use the **endpoint** specified on the home screen of SNPD.
3. Click 'Start' on the homescreen of SNPD
4. Run your application/Initialize the connection to the Clover device
5. The Clover device will ask for a pairing code, which will have been received from within your application. Type it in.
    - Once you've done that, your application will have received an authtoken to use in the future, which will allow you to bypass this whole pairing process.

With that process in mind, let's get to building out your application.
> ### **Step B.1. Prepare your application to receive pairing codes**
> As mentioned, SNPD invloves a pairing process where a pairing code is generated from within your application and typed into a Clover device. The first thing we want to do is make sure that we can retrieve that code.
> >1. Make two callback functions, one to handle the pairingCode and one to handle the pairingAuthToken
> >```C
> >string AuthToken;
> >public static void OnPairingCode(string pairingCode) 
> >{
> >    Console.WriteLine("Enter this pairing code into your device: " + pairingCode);
> >}
> >public static void OnPairingSuccess(string pairingAuthToken) 
> >{
> >    //store this token somewhere to use it in later runs
> >    AuthToken = pairingAuthToken;     
> >}
> >```
> ### **Step B.2. Setup WebsocketCloverDeviceConfiguration**
> The CloverConnector constructor takes in a configuration object specific to the method of connection. Let's look at the WebsocketCloverDeviceConfiguration object, which is the config we'll use to connect through SNPD. Its constructor takes in the following parameters:
> #### Params
> 1. `string endpoint` - The address that specifies which Clover device you're connecting to. This address will be gotten from the Clover device's SNPD app's [homescreen](#dpoverview).
> 2. `string remoteApplicationID` - the [remoteApplicationID][7] you created in your application settings.
> 3. `bool enableLogging` - The CloverConnector has logging capabiltiies that may be useful for debugging. You can turn this feature on or off by passing a boolean into your device config object.
> 4. `int pingSleepSeconds` - How long you want to wait inbetween requests
> 5. `string posName` - any identifier you want to give to your POS
> 6. `string serialNumber` - any identifier you want to give to your device (doesn't have to be the device ID)
> 7. `string pairingAuthToken` - pass in `null` the first time, on subsequent runs we'll pass in the token returned from the pairing process that will allow us to bypass having to pair devices again.
> 8. `PairingDeviceConfiguration.OnPairingCodeHandler pairingCodeHandler` - an object that we'll construct by passing in the callbacks we made in Step B.1
> 9. `PairingDeviceConfiguration.OnPairingSuccessHandler pairingSuccessHandler` - an object that we'll construct by passing in the callbacks we made in Step B.1
> ### **Code**
> >```C
> > public static WebSocketCloverDeviceConfiguration getNetworkConnection()
> > {
> >     var app_id = "com.example.app";
> >     var endpoint = "ws://get.this.from.snpd:12345/remote_pay";
> >     
> >     var config = new WebSocketCloverDeviceConfiguration(
> >         endpoint,
> >         app_id,
> >         false,
> >         1,
> >         "RemotePayConsole",
> >         "Storefront Mini",
> >         null,
> >         new PairingDeviceConfiguration.OnPairingCodeHandler(OnPairingCode),
> >         new PairingDeviceConfiguration.OnPairingSuccessHandler(OnPairingSuccess)
> >         );
> > 
> >     return config;
> > }
> >```


---

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
[11]: https://www.clover.com/appmarket/apps/737JZG428NQ2J?clientCountry=US
[12]: https://docs.clover.com/build/secure-network-pay-display/ 
[13]: https://www.visualstudio.com/
[14]: https://www.microsoft.com/net/download