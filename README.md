# ExalusIntegrationForHomeAssistant

To connect to controller you have to use Controller API.workbook and download Xamarin Workbooks on your computer install it and run.
Then you have to open this file from Xamarin Workbooks and you will see the C# code that allows you to negotiate the server address to which the given controller is connected 
and then connect to packets broker where this code will authorize you to communicate with givrn controller and then login as an user.

Exalus system has an abstraction layer for API communication. The data layer (packets sent between clients and controllers) is exactly the same no matter what kind of communication protocol is used to receive and send them.
So if you use for example SignalR for communication then you send and receive exactly the same data when you use sockets to communicate in local network.

All packets have common structure. It is called DataFrame<T>.
DataFrame must have unique (GUID preferably) transaction ID. Transaction IDs are used to bind responses to given calls.
Because you can do multiple API calls simulatenosly and they can be processed out of order (some calls take a long time to finish, for example when you control devices) you have to understand which response belongs to which call.
You can see how this is handled in the workbook file.

You can run the code and experiment with it just by clicking "play" button under the code (it shows up at the bottom of code lines numbering).

The code provided uses SignalR which is a library for RPC (remote procedure call). On top of it we build our packets broker services that work kind of like packets proxy service. 
Because normally all controllers work in NAT network and they don't have static and public IPs you cannot easly connect to them from outside your home network. 
For that we build an online service to which all controllers connect and you can connect to controllers by providing controller serial number and access pin. 
When given controller is accessible from the cloud then you will be authorized and will have communication channel betwen you and controller.
All APIs in controllers are JSON based packets that have common structure.
They always have:
  - Resource - it is resource ID you want to call, it is similar in structure as HTTP links. So you have for example "/users/user/login".
  - Status - it's the response status for given call. For example you can receive Status.OK if the call succeded, ResourceDoesNotExists if the resource or Method is wrong or you have no permissions to call given resource etc.
  - Method - it's similar to method in HTTP. You have Put, Get, Post, Delete methods and every call has a method. You can have similarly named resource but with different methods.
  - Transaction ID - unique transaction ID that must be set by client for every call.
  - Data - here can be any kind of data that you send or receive from controller.
  
When you send data to controller you will receive response that will have exactly the same transaction ID, Method and Resource but different status and data.

If you connect to controller then you have to login as an user, othervise you will not be able to do anything on controller.
In the workbook you have already controller serial, pin, user and it's password so it's enough to just run the code and you will be connected, authorized and logged in as default user.
