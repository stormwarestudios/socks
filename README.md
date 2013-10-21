socks-factory
=============

socks-factory is a full implementation of the Socks 4, 4a, and 5 protocols in an easy to use node.js module.

### Why socks-factory?

As of this moment, there is not one other Socks proxy library on npm that supports all three varients of the Socks protocol. Nor are there any that support the BIND and associate features that some versions of the Socks protocol supports.

Key Features:
* Supports Socks 4, 4a, and 5 protocols
* Supports the connect method (simple tcp connections of Socks)  (Client -> Socks Server -> Target Server)
* Supports the BIND method (4, 4a, 5)
* Supports the associate (UDP forwarding) method (5)
* Simple and easy to use (one function call to make any type of socks connection)
 
## Installing:

`npm install socks-factory`

### Getting Started Example

For this example, say you wanted to grab the html of google's home page.

```javascript
var SocksFactory = require('socks-factory');

var options = {
    proxy: {
        ipaddress: "202.101.228.108", // Random public proxy
        port: 1080,
        type: 4 // type is REQUIRED. Valid types: [4, 5]  (note 4 also works for 4a)
    },
    target: {
        host: "google.com", // can be an ip address of domain (4a and 5 only)
        port: 80
    },
    command: 'connect'  // This defaults to connect, so it's optional if you're not using BIND or Associate.
};

SocksFactory.createConnection(options, function(err, socket, info) {
    if (err)
        console.log(err);
    else {
        // Connection has been established, we can start sending data now:
        socket.write("GET / HTTP/1.1\nHost: google.com\n\n");
        socket.on('data', function(data) {
            console.log(data.length);
            console.log(data);
        });
        
        // Remember to resume the stream.
        socket.resume();
        
        // 569
        // <Buffer 48 54 54 50 2f 31 2e 31 20 33 30 31 20 4d 6f 76 65 64 20 50 65...
    }
});
```

### BIND Example:

When sending the BIND command to a Socks proxy server, this will cause the proxy server to open up a new tcp port. Once this port is open you, or another client, application, or person can then connect to the socks proxy on that tcp port and communcations will be forwarded to each connection through the proxy itself.

```javascript
var options = {
    proxy: {
        ipaddress: "202.101.228.108",
        port: 1080,
        type: 4,
        command: "bind" // Since we are using bind, we must specify it here.
    },
    target: {
        host: "1.2.3.4", // When using bind, it's best to give an estimation of the ip that will be connecting to the newly opened tcp port on the proxy server. 
        port: 1080
    }
};

SocksFactory.createConnection(options, function(err, socket, info) {
    if (err)
        console.log(err);
    else {
        // BIND request has completed. 
        // info object contains the remote ip and newly opened tcp port to connect to.
        console.log(info);

        // { port: 1494, host: '202.101.228.108' }

        socket.on('data', function(data) {
            console.log(data.length);
            console.log(data);
        });

        // Remember to resume the stream.
        socket.resume();
    }
});

```
At this point, your original connection to the proxy server remains open, and no data will be received until a tcp connection is made to the given endpoint in the info object.

For an example, I am going to connect to the endpoint with telnet:

```
Joshs-MacBook-Pro:~ Josh$ telnet 202.101.228.108 1494
 Trying 202.101.228.108...
 Connected to 202.101.228.108.
 Escape character is '^]'.
 hello
 aaaaaaaaa
```

Note that this connection to the newly bound port does not need to go through the Socks handshake.

Back at our original connection we see that we have received some new data:

```
8
<Buffer 00 5a ca 61 43 a8 09 01>  // This first piece of information can be ignored.

7
<Buffer 68 65 6c 6c 6f 0d 0a> // Hello <\r\n (enter key)>

11
<Buffer 61 61 61 61 61 61 61 61 61 0d 0a> // aaaaaaaaa <\r\n (enter key)>
```

As you can see the data entered in the telnet terminal is routed through the Socks proxy and back to the original connection that was made to the proxy.

**Note** Please pay close attention to the first piece of data that was received. 

```
<Buffer 00 5a ca 61 43 a8 09 01>

        [005a] [PORT:2} [IP:4]
```

This piece of data is technically part of the Socks BIND specifications, but because of my design decisions that were made in an effort to keep this library simple to use, you will need to make sure to ignore and/or deal with this initial packet that is received when a connection is made to the newly opened port.

### Associate Example:

Please reference: http://www.ietf.org/rfc/rfc1928.txt

# Api Reference:

There is only one exported function that you will ever need to use.

###SocksFactory.createConnection( options, callback(err, socket, info)  )

Options:

```javascript
var options = {

    // Information about proxy server
    proxy: {
        // IP Address of Proxy (Required)
        ipaddress: "1.2.3.4",

        // TCP Port of Proxy (Required)
        port: 1080,

        // Proxy Type [4, 5] (Required)
        // Note: 4 works for both 4 and 4a.
        type: 4,

        // Socks Connection Type (Optional)
        // - defaults to 'connect'

        // 'connect'    - establishes a regular Socks connection to the target host. 
        // 'bind'       - establishes an open tcp port on the Socks for another client to connect to.
        // 'associate'  - establishes a udp association relay on the Socks server.
        command: "connect",


        // Socks 4 Specific:

        // UserId used when making a Socks 4/4a request. (Optional)
        userid: "someuserid",

        // Socks 5 Specific:

        // Authentication used for Socks 5 (when it's required) (Optional)
        authentication: {
            username: "Josh",
            password: "somepassword"
        }
    },

    // Information about target host and/or expected client of a bind association. (Required)
    target: {
        // When using 'connect':    IP Address or hostname (4a and 5 only) of a target to connect to.
        // When using 'bind':       IP Address of the expected client that will connect to the newly open tcp port.
        // When using 'associate':  IP Address and Port of the expected client that will send UDP packets to this UDP association.
        
        // Note:
        // When using Socks 4, only an ipv4 address can be used.
        // When using Socks 4a, an ipv4 address OR a hostname can be used.
        // When using Socks 5, ipv4, ipv6, or a hostname can be used.
        host: "1.2.3.4",

        // TCP port of target to connect to.
        port: 1080
    },

    // Amount of time to wait for a connection to be established. (Optional)
    // - defaults to 10000ms (10 seconds)
    timeout: 10000
};
```
Callback:

```javascript

// err:  If an error occurs, err will be an Error object, otherwise null.
// socket: Socket with established connection to your target host.
// info: If using BIND or associate, this will be the remote endpoint to use.

function(err, socket, info) {
  // Hopefully no errors :-)
}
```

# Further Reading:
Please read the Socks 5 specifications for more information on how to use BIND and Associate. 
http://www.ietf.org/rfc/rfc1928.txt

# License
This work is licensed under the [MIT license](http://en.wikipedia.org/wiki/MIT_License).