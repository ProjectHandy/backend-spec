# Handy Backend API Specification (Experimental)

## Architecture
The server has two processes. 
* The first process listens on HTTP,
processes requests from clients, accesses the database, sends replies
to clients, _and sends
[APNs](https://developer.apple.com/library/ios/documentation/NetworkingInternet/Conceptual/RemoteNotificationsPG/Chapters/ApplePushService.html)
notifications to the clients via the second
process_. This process consists of a controller part and a core part. The core part can be thought of as a Haskell function of type `(String, Database) -> (String, Database, Maybe Notification)`. The controller part listens on requests from clients. When the client sends a request, the controller receives a string, and calls the core part with the received string and the current database as input. When the core function returns, it sends the first return value to the user, updates the database with the second return value, and if the third return value is not `Nothing`, it requests the second process to send the notification. 
* The second process listens for requests to send notifications. A
  notification is identified by two strings: the content of the
  message and the device token. To implement this process, we can use
  [a library](https://github.com/notnoop/java-apns) for sending APNs
  notifications. 


## Message Format
### Incoming Message Format
The incoming message consists of a word indicating the action the client wishes the server to perform and several key-value pairs that specifies the action. An incoming message can in general be represented as `<action>?<user>=<username>&<pwd>=<SHA-1 of password>&<key1>=<value1>&<key2>=<value2>&...&<keyn>=<valuen>`. To make sure the keys and values does not contain '?' and '=', we escape them in the messages and the server needs to [unescape](http://hackage.haskell.org/package/network-2.1.0.0/docs/Network-URI.html#v%3AunEscapeString) them. 
### Incoming Message Samples
* `register?user=cggong&email=cggong%40uchicago.edu&pwd=329fjmsdjsdmfsd`
* `login?user=cggong&pwd=23rujewfmis0&token=osfj0jf02imfeowfsd`
* `postbookinfo?user=cggong&pwd=sfdinu9i323&isbn=9783249237&notes=3&price=6.3&notesdesc=Some+notes+taken%2C+but+acceptable+%3A%29`
  * The `notesdesc` field is the string `"Some notes taken, but acceptable :)"`.

### Reply Message Format
The reply message consists of a JSON dictionary.

### Reply Message Samples
* `{"msg": "Error: username password mismatch"}`
* `{"msg": "reqnotif", "user": "bowen", "phone": "3123456789", "nprop": 2, "prop1": {"date": "20160401", "time": "15:00", "place": "Outside Harper", "timedesc": "I have a class ending at 14:50, should be able to get there on time"}, "prop2": {"date": "20160402", "time": "14:00", "place": "Bowen's shrine"}}`

## Structure Of The Remaining Document
The remaining document specifies each specific type of requests and replies in the order of the wireframe (Jan 30 version, pdf in dropbox). 

## Must Haves
### Registration
* Request: 
  * `<action>: register`
  * `<user>: <user name>`
  * `<email>`
  * `<pwd>: <SHA-2 of password>`
* Reply: 
  * `"msg": <error message if any, <action> of request if succeeds>`
 
### Log In: 
Client sends this message to the server so that the server knows its device token and will be able to send notifications to this device. One user account can be associated with several device tokens. 
* Request: 
  * `<action>: login`
  * `<token>`
* Reply: 
  * `"msg"`

### Seller: 
#### Post Book Information
* Request: 
  * `<action>: postbookinfo`
  * `<key1>, <value1>: "isbn", ISBN`
  * `<key2>: notes`
  * `<value2>: 0 lowest - 9 highest`
  * `<key3>: paper`
  * `<value3>: 0 - 9`
  * `<key4>: price`
  * `<value4>: decimal number`
* Optional Request Fields: 
  * `<key5>: notesdesc`
  * `<value5>: description of notes taken, string`
  * `<key6>: paperdesc`
  * `<value6>: description of paper quality, string`
  * `<key7>: pricedesc`
  * `<value7>: description of price offer, string`
* Reply fields: 
  * `"msg": <error message if any, <action> of request if succeed>`

#### Get Request Notification
Request notifications, etc, are communicated to the client in two ways. The client can actively query the server for the new data if any, like specified in this section. When new data is available for the user (new request notifications, etc), the server also uses [Apple Push Notification Service](https://developer.apple.com/library/ios/documentation/NetworkingInternet/Conceptual/RemoteNotificationsPG/Chapters/ApplePushService.html) to push the data to the user, which I didn't expect and will figure out this more. 
* Request: 
  * `<action>: reqnotif`
* Reply: the request can fail, e.g., because the user has not registered. In the case of failure, the reply dict only consists of the first entry. Request notifications for each purchase request are only sent once, i.e., if someone requested a user's book and the that client asks the server for if anyone requested his books for the first time, the server replies with the details of the request. When the client asks the server for the second time, the server does not reply with the same information again. 
  * `"msg": <error message if any, <action> of request if succeed>`
  * `"user": <user name of the request>`
  * `"phone"`
  * `"nprop": <number of meetup proposals>`
  * `"prop1", "prop2", ...`: dictionaries representing meetup proposals. Each proposal has required fields of `"date"`, `"time"`, `"place"`, and optional fields of `"datedesc"`, `"timedesc"`, `"placedesc"`. 

#### Contact the Buyer
TBD

### Buyer: 
#### Search Book
TBD
#### Contact the Seller
TBD
