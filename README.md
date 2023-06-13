# Exalus integration for HomeAssistant
In this document you will find description on how to communicate with Exalus controllers.
## Communication with controllers
To see how to connect to controller you can use `Controller API.workbook` and download `Xamarin Workbooks` on your computer install it and run it.
Then you have to open the workbook file from this repository with `Xamarin Workbooks` and you will see the `C#` code that allows you to negotiate the server address to which the given controller is connected 
and then connect to packets broker where this code will authorize you to communicate with givrn controller and then login as an user.

The code provided uses SignalR which is a library for RPC (remote procedure call). On top of it we build our packets broker services that works kind of like packets proxy service. 
Because normally all controllers work in local - NAT networks - they don't have static and public IPs so you cannot easly connect to them from outside your home network. 
For that we build an online service to which all controllers connect and you can connect to controllers by providing controller serial number and access pin. 
When given controller is accessible from the cloud then you will be authorized and will have communication channel between you and controller.

### Packets abstraction layer
All APIs in controllers are JSON based packets that have common structure.
Exalus system has an abstraction layer for API communication. The data layer (packets sent between clients and controllers) is exactly the same no matter what kind of communication protocol is used to receive and send them.
So if you use for example SignalR for communication then you send and receive exactly the same data as when you use sockets to communicate in a local network.

### Packets structure
All packets have common structure. It is called `DataFrame<T>`.
`DataFrame` must have unique (GUID preferably) transaction ID. Transaction IDs are used to bind responses to given calls.
Because you can do multiple API calls simulatenosly and they can be processed out of order (some calls take a long time to finish, for example when you control devices) you have to understand which response belongs to which call.
You can see how this is handled in the workbook file.

You can run the code and experiment with it just by clicking "play" button under the code (it shows up at the bottom of code lines numbering).


They always have:
* `Resource` - it is resource ID you want to call, it is similar in structure as HTTP links. So you have for example "/users/user/login".
* `Status` - it's the response status for given call. For example you can receive Status.OK if the call succeded, ResourceDoesNotExists if the resource or Method is wrong or you have no permissions to call given resource etc.
* `Method` - it's similar to method in HTTP. You have Put, Get, Post, Delete methods and every call has a method. You can have similarly named resource but with different methods.
* `Transaction ID` - unique transaction ID that must be set by client for every call.
* `Data` - here can be any kind of data that you send or receive from controller.
  
When you send data to controller you will receive response that will have exactly the same transaction ID, Method and Resource but different status and data.

If you connect to controller then you have to login as an user, othervise you will not be able to do anything on controller.
In the workbook you have already controller serial, pin, user and it's password so it's enough to just run the code and you will be connected, authorized and logged in as default user.

### Handling packets
To send data to controller over our broker services you have to first authorize yourself to given controller.
For that first, you have to call our remote service `http://broker.tr7.pl/api/connections/broker/whichserver/{controllerId}` to get the last known packets broker server to which the controller was connected.
In the place of `{controllerId}` in this link you have to put upper cased controller serial number.
Then you will receive server address of signalR broker server to which you can connect.
To connect you can use one of already made SignalR client python libraries and connect to the server address received from call before.
You will need to connect to address `http://{receivedServerDomain}/broker`.
Then in signalR library you will have to implement methods for:
* `authorization to cloud` - for that you will have to create a callback in SignalR library that will act on the execution of your procedure named "Authorization" that will receive just one parameter as string with authoprization result  `true` or `false`. This will tell you if you are sucessfully authorized and have estabilished connection with controller or not. You can receive false in case of wrong serial and pin provided or when the controller is no longer connected to this broker. And when you register that client side RPC method then you can execute method on the server side called `AuthorizeTo` and provide as paremeters controller serial and pin.
* `handling errors` to handle errors for example when you are no longer authorized or there is other error you have to register client side method called `SendError` that have just one `string` parameter with error message. If the error message starts with `NotAuthorized:` it tells you that you are no longer authorized to given controller.
* `Pong` this client side methods allows you to receive confirmation that you are still connected to the server. If you want to check if you are still connected you have to execute `Ping` method on the server. (no need to implement this, you can just observe signalR connection status, there is a hearbeat too in the library). 
*  `Data` the most important client side RPC method that allows you to receive DataFrame packets from controllers. This method must have two parameters `sender` and `data`. Sender is a `string` parameter that tells you from what controller you received the response. `data` is an DataFrame JSON object that you have to parse and handle.

To send `DataFrame` to controller you have to execute RPC method on server called `SendTo` that has two parameters `controller serial` where you have to put the serial ID of the controller that you want to send the data to and serialized to string `DataFrame` that you want to send to controller.

There is always full-duplex (two-way) communication between clients and controllers. Packets can be send from controller to users without interaction from other end. For example when there is a device state change then controller will send a packet with device status update even, when you did not asked for that. It will make sure you are up-to-date and always will have real-time updates. Many things are synced this way, controllers send information that there was a change and library must handle that if it implements given functionality.
