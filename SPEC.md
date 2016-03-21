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
The incoming message consists of a word indicating the action the client wishes the server to perform and several key-value pairs that specifies the action. An incoming message can in general be represented as `<action>?{"user": <username>, "pwd": <SHA-1 of password>, "key1": <value1>, "key2": <value2>, ..., "keyn": <valuen>}`. 
### Incoming Message Samples
* `register?{"user": "cggong", "email": "cggong@uchicago.edu", "pwd": "329fjmsdjsdmfsd"}`
* `login?{"user": "cggong", "pwd": "23rujewfmis0&token=osfj0jf02imfeowfsd"`
* `postbookinfo?{"user": "cggong", "pwd": "sfdinu9i323", "isbn": "9783249237", "notes": 3, "price": 6.3, "notesdesc": "Some notes taken, but acceptable :)"}`
  * The `notesdesc` field is the string `"Some notes taken, but acceptable :)"`.

### Reply Message Format
The reply message consists of a JSON dictionary.

### Reply Message Samples
* `{"msg": "Error: username password mismatch"}`
  * Error messages begins with `Error`. 
* `{"msg": "getprop", "id": 150, "buyer": "bowen", "phone": "3123456789", "props": [{"date": "20160401", "time": "15:00", "place": "Outside Harper", "timedesc": "I have a class ending at 14:50, should be able to get there on time"}, {"date": "20160402", "time": "14:00", "place": "Bowen's shrine"}]}`

## Implementation of Actual Transportation
The client requests are HTTP POST requests like `http://serverdomain/register` with payload `{"user": "cggong", "email": "cggong@uchicago.edu", "pwd": "329fjmsdjsdmfsd"}`. The controller part is responsible for concatenating the request (with the `http://serverdomain/` part truncated away) with the payload separated by a `?`. The server replys are HTTP responses. 

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
  * `"id": an integer identifying the selling item`

#### Get Request Notification
Request notifications, etc, are communicated to the client in two
ways. The client can actively query the server for the new data if
any, like specified in this section. When new data is available for
the user (new request notifications, etc), the server also uses
[Apple Push Notification Service](https://developer.apple.com/library/ios/documentation/NetworkingInternet/Conceptual/RemoteNotificationsPG/Chapters/ApplePushService.html)
to push the data to the user. 
* Request: 
  * `<action>: getprop`
* Reply: the request can fail, e.g., because the user has not registered. In the case of failure, the reply dict only consists of the first entry. Request notifications for each purchase request are only sent once, i.e., if someone requested a user's book and that client asks the server "if anyone requested his books" for the first time, the server replies with the details of the request. When the client asks the server for the second time, the server does not reply with the same information again. 
  * `"msg": <error message if any, <action> of request if succeed>`
  * `"id": <the ID of the selling item>`
  * `"buyer": <user name of the buyer>`
  * `"phone"`
  * `"props"`: array of dictionaries representing meetup proposals. Each proposal has required fields of `"date"`, `"time"`, `"place"`, and optional fields of `"datedesc"`, `"timedesc"`, `"placedesc"`. 
* Optional Reply Fields: 
  * `"chat"`: chat message

#### Contact the Buyer
The contact messages of seller and buyer to negotiate a meetup time have the action type `propose`. After the seller receives a request notification from a buyer, the seller picks a proposal and put it in the action `propose`. If the seller likes none of the proposals, he can propose other proposals. For example, a procedure can be like: 
* buyer: time 1, time 2, time 3
* seller: time 4, time 5 = 15:00, time 6, desc: Sorry I'm available on none of the time slots! 
* buyer: time 5
* seller: time 5 // OK, see you then! 
* ... some time later, right before time 5
* buyer: time 5, chat: I'm almost there. 
  * `propose?{"id": 834, "buyer": "alice", "props":
    [{"date": "20160401", "time": "15:00", "place": "Reg"}], "chat":
    "I'm almost there."}]`
* seller: time 5, chat: I'm waiting. I have a book in my hand. 
* buyer: time 5, chat: I see you. 

In general, the proposal is determined iff the buyer and seller has only one time slot in their proposals in the last round of communication and their proposal agrees. The request has similar structure with the reply of `getprop`. 
* Request: 
  * `<action>: propose`
  * `<id>: <the ID of the selling item>`
  * `<buyer>: <user name of the buyer>`
  * `<phone>`
  * `"props"`: same as the reply of `getprop`
* Optional Request Fields: 
  * `"chat"`: string of chat message
  * `"control"`: array of strings represent control information. `completed` means transaction completed. `withdraw` means the transaction is withdrawn. 
* Reply: 
  * `"msg"`

### Buyer: 
#### Search Book
The process of searching a book can involve two types of actions. The first action provides a string indicating the book, like specifying its book name, author, ISBN, etc, and requests the detailed information of that book. 
* Request: 
  * `<action>: matchbook`
  * `<indicator>`: a string indicating the book. 
* Reply: 
  * `"msg"`
  * `"books"`: array of dictionaries representing books. Each book has fields of `"title"`, `"author"`, `"isbn"`. 

The second action provides the ISBN of a book, and requests selling information of such book. 
* Request: 
  * `<action>`: buysearch
  * `<isbn>`
* Reply: 
  * `"msg"`
  * `"items"`: array of dictionaries representing selling information. Each item has required fields of `"isbn"`, `"notes"`, `"paper"`, `"price"`, and optional fields of `"notesdesc"`, `"paperdesc"`, `"pricedesc"`, which is the same as the request fields in `postbookinfo`. 

#### Contact the Seller
See section "Contact the Buyer". 
