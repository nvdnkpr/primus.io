# Primus.IO

[![Build Status](https://travis-ci.org/cayasso/primus.io.png?branch=master)](https://travis-ci.org/cayasso/primus.io)
[![NPM version](https://badge.fury.io/js/primus.io.png)](http://badge.fury.io/js/primus.io)

Primus.IO makes working with [Primus](https://github.com/3rd-Eden/primus) a little slicker, it ads some hight level features like:

- Emit-style `emit()` w/ arguments.
- Client & server side "ack" callbacks.
- Multiplexing via channels.
- Rooms.
- Serves `/primus.io.js`.

Primus.IO combines the core [Primus](https://github.com/3rd-Eden/primus) with [primus-rooms](https://github.com/cayasso/primus-rooms), [primus-emitter](https://github.com/cayasso/primus-emitter) and [primus-multiplex](https://github.com/cayasso/primus-multiplex) plugins to provide an easy and still powerfull way of developing real time applications.

### Instalation

```bash
$ npm install primus.io
```

### Quick Start

#### On the Server

```javascript
var Primus = require('primus.io');
var server = require('http').createServer();

var primus = new Primus(server, { transformer: 'websockets', parser: 'JSON' });

primus.on('connection', function (spark) {

  // listen to hi events
  spark.on('hi', function (msg) {
    
    console.log(msg); //-> hello world

    // send back the hello to client
    spark.emit('hello', 'hello from the server');

  });

});

server.listen(8080);
```

#### On the Client

```javascript
var primus = Primus.connect('ws://localhost:8080');

primus.on('open', function () {

  // Send request to join the news room
  primus.emit('hi', 'hello world');

  // listen to hello events
  primus.on('hello', function (msg) {

    console.log(msg); //-> hello from the server

  });

});

```

Check the examples for more use cases.

## How to use

### Using with Node HTTP server

#### Server

```javascript
  var Primus = require('primus')
    , http = require('http')
    , fs = require('fs');

  // serve index.html
  var server = http.createServer(function server(req, res) {
    res.setHeader('Content-Type', 'text/html');
    fs.createReadStream(__dirname + '/index.html').pipe(res);
  });

  // Primus server
  var primus = new Primus(server, { transformer: 'websockets', parser: 'JSON' });

  primus.on('connection', function (spark) {
    spark.emit('news', { hello: 'world' });
    spark.on('my other event', function (data) {
      console.log(data);
    });
  });

  server.listen(8080);
```

#### Client

```javascript
  var primus = new Primus('http://localhost:8080/');

  primus.on('news', function (data) {
    console.log(data);
    primus.emit('my other event', { my: 'data' });
  });
```


### Using with Express

Express requires that you instantiate a `http.Server` to attach socket.io to first:

#### Server

```javascript
  var express = require('express')
    , Primus = require('primus.io')
    , http = require('http')
    , app = express()
    , server = http.createServer(app);

  // Primus server
  var primus = new Primus(server, { transformer: 'websockets', parser: 'JSON' });

  primus.on('connection', function (spark) {
    spark.emit('news', { hello: 'world' });
    spark.on('my other event', function (data) {
      console.log(data);
    });
  });

  // serve index.html
  app.get('/', function (req, res) {
    res.sendfile(__dirname + '/index.html');
  });

  server.listen(8080);
```

#### Client

```javascript
  var primus = new Primus('http://localhost:8080/');

  primus.on('news', function (data) {
    console.log(data);
    primus.emit('my other event', { my: 'data' });
  });
```

### Sending and receiving events.

Primus.IO allows you to emit and receive custom events:

#### Server

```javascript
  var Primus = require('primus.io')
    , server = require('http').Server();

  var primus = new Primus(server, { transformer: 'websockets', parser: 'JSON' });

  primus.on('connection', function (spark) {

    spark.emit('welcome', 'welcome to the server');

    spark.on('private message', function (from, msg) {
      console.log('I received a msg by ', from, ' saying ', msg);
    });

  });

  server.listen(8080);
```

#### Client

```javascript
  var primus = new Primus('http://localhost:8080/');
  
  primus.on('welcome', function (msg) {
    primus.emit('private message', 'Bob', 'hi!');
  });  
```

Check for more documentation on event emitting here [primus-emitter](https://github.com/cayasso/primus-emitter).

### Using channels (or known as namespaces).

Channels provides the benefit of `multiplexing` a single connection.

#### Server

```javascript
  var Primus = require('primus.io')
    , server = require('http').Server();

  var primus = new Primus(server, { transformer: 'websockets', parser: 'JSON' });

  var chat = primus.channel('chat');
  var news = primus.channel('news');

  chat.('connection', function (spark) {
    spark.emit('chat', 'welcome to this chat');
  });

  news.on('connection', function (socket) {
      socket.emit('news', { news: 'item' });
  });

  server.listen(8080);
```

#### Client

```javascript
  var primus = new Primus('http://localhost:8080/')
    , chat = primus.channel('chat')
    , news = primus.channel('news');
  
  chat.on('chat', function (msg) {
    console.log(msg); //-> welcome to this chat
  });
  
  news.on('news', function (data) {
    console.log(data.news); //-> item
  });
```

Checkout this [post](https://www.rabbitmq.com/blog/2012/02/23/how-to-compose-apps-using-websockets/) 
for more deep understanding of channels and why it's implemented like this.

Also check out for more documentation on multiplexing here [primus-multiplex](https://github.com/cayasso/primus-multiplex).

### Acknowledgements

To get a callback when the server or client confirmed the message reception, simply pass a function as the last parameter of `.emit`.

#### Server

```javascript
  var Primus = require('primus.io')
    , server = require('http').Server();

  var primus = new Primus(server, { transformer: 'websockets', parser: 'JSON' });

  primus.on('connection', function (spark) {
    spark.on('chat', function (name, fn) {
    console.log(name); //-> Bob
    fn('woot');

    spark.emit('What is your name', function (name) {
      console.log(name); //-> My name is Ann
    });
  });

});
```

#### Client

```javascript
  var primus = new Primus('http://localhost:8080/');

  primus.on('open', function () {
    primus.emit('chat', 'Bob', function (msg) {
      console.log(msg); //-> woot
    });

    primus.on('What is your name', function (fn) {
      fn('My name is Ann')
    });
  });
```


### Broadcasting messages (server side).

To broadcast a message to all connected clients simple use the `primus.write` method. The same apply for channels.

#### Server

```javascript
  var Primus = require('primus.io')
    , server = require('http').Server();

  var primus = new Primus(server, { transformer: 'websockets', parser: 'JSON' });

  primus.write('Some data');

});
```

#### Client

```javascript
  var primus = new Primus('http://localhost:8080/');

  primus.on('data', function (data) {
    console.log(data); //-> Some data
  });
```

You can also use the `primus.forEach` method to iterate over all current connections.

#### Server

```javascript
  var Primus = require('primus.io')
    , server = require('http').Server();

  var primus = new Primus(server, { transformer: 'websockets', parser: 'JSON' });

  primus.forEach(function (spark, id, connections) {
    if (spark.query.foo !== 'bar') return;

    spark.write('message');
  });

});
```

Check out more information on [broadcasting with Primus](https://github.com/3rd-Eden/primus#broadcasting).

### Rooms

#### Server

```javascript
var Primus = require('primus.io');
var server = require('http').createServer();

// primus instance
var primus = new Primus(server, { transformer: 'websockets' });

primus.on('connection', function (spark) {

  spark.on('join', function (room) {
    spark.join(room, function () {

      // send message to this client
      spark.emit('sport', 'you joined room ' + room);

      // send message to all clients except this one
      spark.room(room).emit('sport', spark.id + ' joined room ' + room);
    });
  });

  spark.on('leave', function (room) {
    spark.leave(room, function () {
        
      // send message to this client
      spark.emit('sport', 'you left room ' + room);
    });
  });

});

server.listen(8080);
```

#### Client

```javascript
var primus = Primus.connect('ws://localhost:8080');

primus.on('open', function () {

  // Send request to join the sport room
  primus.emit('join', 'sport');

  // Then later send request to leave the sport room
  primus.emit('leave', 'sport');

  // print server message
  primus.on('sport', function (message) {
    
    console.log(message); 

    // First output is
    //-> you joined room sport

    // Then later
    //-> you left room sport

  });

});

```

You can check for more documentation on rooms here [primus-rooms](https://github.com/cayasso/primus-rooms).

### Run tests

There will be some very minor integration test soon, for all the
tests check out each individual plugin test.

``` bash
$ make test
```

### Todo

 * Add broadcasting from the server with `primus.emit`.  

### Credits

 * To Arnout Kazemier [3rdEden](https://twitter.com/3rdEden) for the awesome idea of building [Primus](https://github.com/3rd-Eden/primus).

### License

(The MIT License)

Copyright (c) 2013 Jonathan Brumley &lt;cayasso@gmail.com&gt;

Permission is hereby granted, free of charge, to any person obtaining
a copy of this software and associated documentation files (the
'Software'), to deal in the Software without restriction, including
without limitation the rights to use, copy, modify, merge, publish,
distribute, sublicense, and/or sell copies of the Software, and to
permit persons to whom the Software is furnished to do so, subject to
the following conditions:

The above copyright notice and this permission notice shall be
included in all copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED 'AS IS', WITHOUT WARRANTY OF ANY KIND,
EXPRESS OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF
MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT.
IN NO EVENT SHALL THE AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY
CLAIM, DAMAGES OR OTHER LIABILITY, WHETHER IN AN ACTION OF CONTRACT,
TORT OR OTHERWISE, ARISING FROM, OUT OF OR IN CONNECTION WITH THE
SOFTWARE OR THE USE OR OTHER DEALINGS IN THE SOFTWARE.
