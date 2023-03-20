# QUICKSTART:

cargo run, then `wscat -c localhost:8080/c05554ae-b4ee-4976-ac05-97aaf3c98a23`. the route at the end of the connection string is your "lobby". everyone else in that "lobby" can see what you send. Lobby ID MUST be a valid uuid.


# WebSockets in Actix Web full tutorial - WebSockets & Actors

This tutorial will walk you through each step of writing a blazingly fast WebSocket client in Actix Web, in depth and with a working repository as refrence.

We'll be building a simple chatroom that echos messages to everyone in a room, and include private messages. I'll also explain every step, so you can extend this example and write your own WebSocket server in Actix Web. WebSockets in Actix heavily uses the Actor framework, something that you don't need to be too familiar with to write Actix Web - but will add another tool into your Rusty toolbelt.

### Prerequisites:
1. Know what WebSockets are at a general level
2. Know some basic Rust

Repo for the completed project: `https://github.com/antholeole/actix-sockets`

Video version of this tutorial coming out soon! If you want to get notified when the video comes out, leave a comment and I'll let you know :)


## The setup

first, run `cargo init ws-demo` or something similar. then, go to your `Cargo.toml` and make sure you depend on these packages: 

```toml
[dependencies]
actix-web="3.2.0" # duh
actix-web-actors="3" # actors specific to web
actix = "0.10.0" # actors
uuid = { version = "0.8", features = ["v4", "serde"] } # uuid's fit well in this context.
```

you might want to `cargo run` the hello world and go make a coffee; compiling all these crates will take a minute or two.

## Warm up to actix architecture

In the actix architecture, there are two primary components: Actors and Messages. Think of each actor as it's own object in memory, with a mailbox. Actors can read their mailbox and respond to their mail accordingly, whether it be by sending mail to another actor, changing it's state, or maybe by doing nothing at all. That's it! That's all that actors are - simple little things that read and respond to mail. 

Actors are so ungodly fast because they work entirely independent of each other. One actor can be on it's own thread, or on a different machine entirely, and as long as the actor can read it's mail, it works perfectly. 

It's important to note that the actor just exists in memory, with it's address passed around like `Addr<Actor>`. The Actor itself can mutate its properties (maybe you have a "messages_received" property, and you need to increment it on every message) but you can't do that anywhere else. Instead, with the `Addr<Actor>` element, you can do `.send(some_message)` to put a message in that Actor's mailbox. 

In Actix web, each socket connection is it's own Actor, and the "Lobby" (we'll get to that) is it's own actor also.

## App achitecture

A brief on the app achitecture:

Each socket sits in a "room" and each room sits in a singular lobby struct. That's it!

## ws.rs

Forewarning: we're going to be writing it one file at a time, rather than jumping from place to place, so at a lot of points your code won't compile. Don't worry! At the end, everything will fit together like glue and run perfectly, if you followed the tutorial correctly.

the first step will be to define our WebSocket object. create a file called `ws.rs`. You'll need the following imports:

```rust
use actix::{fut, ActorContext};
use crate::messages::{Disconnect, Connect, WsMessage, ClientActorMessage}; //We'll be writing this later
use crate::lobby::Lobby; // as well as this
use actix::{Actor, Addr, Running, StreamHandler, WrapFuture, ActorFuture, ContextFutureSpawner};
use actix::{AsyncContext, Handler};
use actix_web_actors::ws;
use actix_web_actors::ws::Message::Text;
use std::time::{Duration, Instant};
use uuid::Uuid;


const HEARTBEAT_INTERVAL: Duration = Duration::from_secs(5);
const CLIENT_TIMEOUT: Duration = Duration::from_secs(10);
```


### the struct

Define a struct with the following signature: 

```rust
pub struct WsConn {
    room: Uuid,
    lobby_addr: Addr<Lobby>,
    hb: Instant,
    id: Uuid,
}
```

the fields are as follows: 
- room: each Socket exists in a 'room', which in this implementation will just be a simple HashMap that maps Uuid -> List of Socket Ids. More on that later, but having the room stored in the signature of the socket itself will be useful later down the line for private messages.
- addr: This is the Addr of the lobby that the socket exists in.  This will be used to send data to the lobby. so, sending a text message to the lobby might look like this: `self.addr.do_send('hi!')`. Without this property, it would be impossible for this actor to find the Lobby in memory.
- hb: Although WebSockets do send messages when they close, sometimes WebSockets close without any warning. instead of having that socket exist forever, we send it a heartbeat every N seconds, and if we don't get a response back, we terminate the socket. This property is the time since we recieved the last heartbeat. In a lot of libraries, this is handled automatically.
- id: this is the ID assigned by us to that socket. This is useful for private messaging, so we can `/whisper <id> hello!` to whisper to that client. 

### 'new'
We'll write a quick `new` trait so that we can spin up the socket a little bit easier:

```rust
impl WsConn {
    pub fn new(room: Uuid, lobby: Addr<Lobby>) -> WsConn {
        WsConn {
            id: Uuid::new_v4(),
            room,
            hb: Instant::now(),
            lobby_addr: lobby,
        }
    }
}
```

this way, we don't have to deal with setting up the heartbeat or assigning an id. 

### Making it into an actor

Notice how the `WsConn` is just a plain old Rust struct. to convert it into an actor, we need to implement the Actor trait on it. 

here's the code in full, and then we'll dissect it:

```rust
impl Actor for WsConn {
    type Context = ws::WebsocketContext<Self>;

    fn started(&mut self, ctx: &mut Self::Context) {
        self.hb(ctx);

        let addr = ctx.address();
        self.lobby_addr
            .send(Connect {
                addr: addr.recipient(),
                lobby_id: self.room,
                self_id: self.id,
            })
            .into_actor(self)
            .then(|res, _, ctx| {
                match res {
                    Ok(_res) => (),
                    _ => ctx.stop(),
                }
                fut::ready(())
            })
            .wait(ctx);
    }

    fn stopping(&mut self, _: &mut Self::Context) -> Running {
        self.lobby_addr.do_send(Disconnect { id: self.id, room_id: self.room });
        Running::Stop
    }
}
```

first, you'll see we define a type called `Context` in the definition (`type Context = ws::WebsocketContext<Self>;`). That's mandated by the actor. That's the `context` in which this actor lives; here, we're saying that the context is the WebSocket context, and that it should be allowed to do WebSocket stuff, like start listening on a port. We'll be defining a plain old context in the `Lobby` section in the future.

We also write the `started` and `stopping` methods - this will create and destroy the Actor respectively. 

In Started, we begin the heartbeat loop; It's just a function that triggers on an interval, so after we start the loop, we don't have to worry about it. (We'll, again, write that a little later :) ) 

We also take the address of the lobby and send it a message saying "Hey! I connected. This is the lobby I want to get into, and my id, as well as the address of my mailbox you can reach me at." We send that message **asynchronously**. If we did `do_send` instead of `send`, we'd be sending it _kind of_ synchronously. By kind of, I mean "chuck the message at the mailbox and drive away." `do_send` doesn't care if the message ever sent or got read. `send` needs to be awaited, which is the purpose of this block:

```rust
.then(|res, _, ctx| {
     match res {
        Ok(_res) => (),
        _ => ctx.stop(),
    }
    fut::ready(())
})
```

If anything fails, we just stop the whole Actor with `ctx.stop`. This likely won't happen, but might if something is wrong with your `Lobby` actor. The client will see something along the lines of `ws handshake couldn't be completed.`

Stopping is much easier. You can see the `do_send` in action here: we try to send a Disconnect message to the lobby, but if we can't, no big deal. Just stop this actor. 

And that's it! Our `WsConn` is now an Actor. 

### heartbeat

here's the heartbeat method we were talking about:

```rust
impl WsConn {
    fn hb(&self, ctx: &mut ws::WebsocketContext<Self>) {
        ctx.run_interval(HEARTBEAT_INTERVAL, |act, ctx| {
            if Instant::now().duration_since(act.hb) > CLIENT_TIMEOUT {
                println!("Disconnecting failed heartbeat");
                ctx.stop();
                return;
            }

            ctx.ping(b"PING");
        });
    }
}
```

all that we do here is ping the client, and wait for a response on an interval. If the response doesn't come, the socket died; stop the client. This triggers the `stopping()` function we defined earlier to be called, such that we disconnect from the lobby.

### handling WS messages

This next method is a little bit long, but it's not too complicated:

```rust
impl StreamHandler<Result<ws::Message, ws::ProtocolError>> for WsConn {
    fn handle(&mut self, msg: Result<ws::Message, ws::ProtocolError>, ctx: &mut Self::Context) {
        match msg {
            Ok(ws::Message::Ping(msg)) => {
                self.hb = Instant::now();
                ctx.pong(&msg);
            }
            Ok(ws::Message::Pong(_)) => {
                self.hb = Instant::now();
            }
            Ok(ws::Message::Binary(bin)) => ctx.binary(bin),
            Ok(ws::Message::Close(reason)) => {
                ctx.close(reason);
                ctx.stop();
            }
            Ok(ws::Message::Continuation(_)) => {
                ctx.stop();
            }
            Ok(ws::Message::Nop) => (),
            Ok(Text(s)) => self.lobby_addr.do_send(ClientActorMessage {
                id: self.id,
                msg: s,
                room_id: self.room
            }),
            Err(e) => panic!(e),
        }
    }
}

```

It's a simple pattern match on all the possible WebSocket messages.
- the ping responds with a pong - that's the client heartbeating us. Respond with a pong. As a byproduct, since the client can heartbeat us, we know that it's alive so we can reset our heartbeat clock.
- The pong is the response to the ping we sent. Reset our clock, they're alive.
- If the message is binary, we'll send it to the WebSocket context which will figure out what to do with it. This realistically should never be triggered.
- If it's a close message just close.
- For this tutorial, we're not going to respond to continuation frames (these are, in short, WebSocket messages that couldn't fit into one message) 
- On nop let's nop (no operation)
- On a text message, (this one we'll be doing the most!) send it to the lobby. The lobby will deal with brokering it out to where it needs to go.
- On an error, we'll panic. You'll probably want to implement what to do here reasonably.

### Responding to text messages

```rust
impl Handler<WsMessage> for WsConn {
    type Result = ();

    fn handle(&mut self, msg: WsMessage, ctx: &mut Self::Context) {
        ctx.text(msg.0);
    }
}
```

Here, if the server puts a `WsMessage` (Which we need to define) mail into our mailbox, all we do is send it straight to the client. This is what 'reading the mail' from the mailbox looks like; `impl Handler<MailType> for ACTOR`. Note that we also need to define what the response to that mail may look like. If the mail is placed like `do_send`, the response type doesn't matter. If it's placed like `send()`, then the awaited result type will be what `Result` is. maybe you do `type Result = String`, or something similar. Regardless, whatever type `T` you put there, `handle` needs to return `T`.

Also, the signature of the handler message includes:
1. The message itself. You have complete control over how much or how little data this message passes. 
2. Self context. This is your own context, which is a "mailbox" of self. You can read memeber variables from the ctx, or you can put messages into your own mailbox here.

And that's it! That is the whole `WsClient`.

## The Messages for The Mailboxes

create a new file called `messages.rs`. This file will hold all the "messages" that get put into our actor's mailboxes. We use a struct with the two traits: the first is simple, and is just `#[derive(Message)]` that tells us that it's an actor message. The second is `rtype`. That return type must be the same `T` we talked about in the end of the last section. So if we wanted to define a message that returned a string, it'd look like this:

```rust
#[derive(Message)]
#[rtype(result = "String")] // result = your type T
pub struct MyMessage; // they usually carry info, but not for this example

impl Handler<MyMessage> for MyActor {
    type Result = String; // This type is T

    fn handle(&mut self, msg: MyMessage, ctx: &mut Self::Context) -> String { // Returns your type T
        ...
    }
}
```

And that's it! defining messages is actually pretty easy. Here's the code drop for the message file, with respective comments:

```rust
use actix::prelude::{Message, Recipient};
use uuid::Uuid;

//WsConn responds to this to pipe it through to the actual client
#[derive(Message)]
#[rtype(result = "()")]
pub struct WsMessage(pub String);

//WsConn sends this to the lobby to say "put me in please"
#[derive(Message)]
#[rtype(result = "()")]
pub struct Connect {
    pub addr: Recipient<WsMessage>,
    pub lobby_id: Uuid,
    pub self_id: Uuid,
}

//WsConn sends this to a lobby to say "take me out please"
#[derive(Message)]
#[rtype(result = "()")]
pub struct Disconnect {
    pub room_id: Uuid,
    pub id: Uuid,
}

//client sends this to the lobby for the lobby to echo out.
#[derive(Message)]
#[rtype(result = "()")]
pub struct ClientActorMessage {
    pub id: Uuid,
    pub msg: String,
    pub room_id: Uuid
}
```

Defining messages for actors is super easy. One of my favorite parts about the actor framework is if you need to get data from one Actor to another, creating a message or adding data to an existing message is super easy.

## lobby.rs

First, the imports for `lobby.rs`:

```rust
use crate::messages::{ClientActorMessage, Connect, Disconnect, WsMessage};
use actix::prelude::{Actor, Context, Handler, Recipient};
use std::collections::{HashMap, HashSet};
use uuid::Uuid;
```

we're almost done! Now, we have the actual lobby to write. Like we've said, the lobby is an Actor, but the actor is a plain old struct. Here's that struct (to be placed in `lobby.rs`):

```rust
type Socket = Recipient<WsMessage>;

pub struct Lobby {
    sessions: HashMap<Uuid, Socket>,          //self id to self
    rooms: HashMap<Uuid, HashSet<Uuid>>,      //room id  to list of users id
}
```

We store the Socket as a simple recepient of a WsMessage. With this setup, we can easily navigate through all, some, or find a specific client.

as a helper, we'll implement a default for the lobby:

```rust
impl Default for Lobby {
    fn default() -> Lobby {
        Lobby {
            sessions: HashMap::new(),
            rooms: HashMap::new(),
        }
    }
}
```

and now let's write a helper that sends a message to a client. 

```rust
impl Lobby {
    fn send_message(&self, message: &str, id_to: &Uuid) {
        if let Some(socket_recipient) = self.sessions.get(id_to) {
            let _ = socket_recipient
                .do_send(WsMessage(message.to_owned()));
        } else {
            println!("attempting to send message but couldn't find user id.");
        }
    }
}
```

This method takes a string and a id, and sends that string to a client with that Id (if it exists; if it doesn't, it just prints something to console. You probably want to handle this accordingly by returning a result or similar.)

### Making the lobby an actor

Ready for the shortest section in this whole tutorial? To make the lobby an actor, use this following code:

```rust
impl Actor for Lobby {
    type Context = Context<Self>;
}
```

That's it! We don't care about any lifecycle of the Lobby. We only have one, and we only mount it when the app starts and remove it when the app closes. The rooms are just HashSets, so we don't need to make them actors for this simple example.

### Handling Messages

The lobby will get 3 types of messages: Connects, Disconnects, and WsMessage from the clients. Both come from the WsConn lifecycle methods from the actor trait. This part isn't actix specific, but I'll explain the meat of the code.

```rust
/// Handler for Disconnect message.
impl Handler<Disconnect> for Lobby {
    type Result = ();

    fn handle(&mut self, msg: Disconnect, _: &mut Context<Self>) {
        if self.sessions.remove(&msg.id).is_some() {
            self.rooms
                .get(&msg.room_id)
                .unwrap()
                .iter()
                .filter(|conn_id| *conn_id.to_owned() != msg.id)
                .for_each(|user_id| self.send_message(&format!("{} disconnected.", &msg.id), user_id));
            if let Some(lobby) = self.rooms.get_mut(&msg.room_id) {
                if lobby.len() > 1 {
                    lobby.remove(&msg.id);
                } else {
                    //only one in the lobby, remove it entirely
                    self.rooms.remove(&msg.room_id);
                }
            }
        }
    }
}
```

All we're doing is responding to a disconnect message by either:
1. removing a single client from a lobby. send everyone else "UUID disconnected!
2. If that client was the last in the lobby, remove the lobby entirely (so we don't clog up our hashmap)

Next, we need to respond to the connection message. Again, nearly no logic that is actor specific:

```rust
impl Handler<Connect> for Lobby {
    type Result = ();

    fn handle(&mut self, msg: Connect, _: &mut Context<Self>) -> Self::Result {
        // create a room if necessary, and then add the id to it
        self.rooms
            .entry(msg.lobby_id)
            .or_insert_with(HashSet::new).insert(msg.self_id);

        // send to everyone in the room that new uuid just joined
        self
            .rooms
            .get(&msg.lobby_id)
            .unwrap()
            .iter()
            .filter(|conn_id| *conn_id.to_owned() != msg.self_id)
            .for_each(|conn_id| self.send_message(&format!("{} just joined!", msg.self_id), conn_id));

        // store the address
        self.sessions.insert(
            msg.self_id,
            msg.addr,
        );

        // send self your new uuid
        self.send_message(&format!("your id is {}", msg.self_id), &msg.self_id);
    }
}
```

All that is being done here is adding a socket and sending them messages.

Finally, we open the mailbox for clients to send messages to the lobby for the lobby to forward to clients. 

```rust
impl Handler<ClientActorMessage> for Lobby {
    type Result = ();

    fn handle(&mut self, msg: ClientActorMessage, _: &mut Context<Self>) -> Self::Result {
        if msg.msg.starts_with("\\w") {
            if let Some(id_to) = msg.msg.split(' ').collect::<Vec<&str>>().get(1) {
                self.send_message(&msg.msg, &Uuid::parse_str(id_to).unwrap());
            }
        } else {
            self.rooms.get(&msg.room_id).unwrap().iter().for_each(|client| self.send_message(&msg.msg, client));
        }
    }
}
```

This checks if the message starts with a \w. If it does, we know it's a whisper, and we send it to a specific client. If it's not, we send it to all users in the room. (this is **NOT** production ready! It will panic if there is an invalid UUID after the /w, and will just fail silently if it's just \w without anything after it.)

And that's the lobby! 

## Final step! Setting up the Route / Running Server

First, we have to open up a route that lets us connect to the server. create a file called "start_connection.rs" and put in this route: 

```rust
use crate::ws::WsConn;
use crate::lobby::Lobby;
use actix::Addr;
use actix_web::{get, web::Data, web::Path, web::Payload, Error, HttpResponse, HttpRequest};
use actix_web_actors::ws;
use uuid::Uuid;


#[get("/{group_id}")]
pub async fn start_connection(
    req: HttpRequest,
    stream: Payload,
    Path(group_id): Path<Uuid>,
    srv: Data<Addr<Lobby>>,
) -> Result<HttpResponse, Error> {
    let ws = WsConn::new(
        group_id,
        srv.get_ref().clone(),
    );

    let resp = ws::start(ws, &req, stream)?;
    Ok(resp)
}
```

We define a route that just has a group id (should be a valid uuid) as a path param. Then, we create a new WsConn with a refrence to the Lobby (we register the lobby with actix web in the next step). Finally, we upgrade the request to a WebSocket request, and bam! we now have an open persistant connection.

Our last step is to register the Lobby as shared data so we can get it like we just did (`srv: Data<Addr<Lobby>>`). Your main.rs should look like the following:

```rust
mod ws;
mod lobby;
use lobby::Lobby;
mod messages;
mod start_connection;
use start_connection::start_connection as start_connection_route;
use actix::Actor;

use actix_web::{App, HttpServer};

#[actix_web::main]
async fn main() -> std::io::Result<()> {
    let chat_server = Lobby::default().start(); //create and spin up a lobby

    HttpServer::new(move || {
        App::new()
            .service(start_connection_route) //. rename with "as" import or naming conflict
            .data(chat_server.clone()) //register the lobby
    })
    .bind("127.0.0.1:8080")?
    .run()
    .await
}

```

and boom! Our whole app now shares a single lobby. Here's everything we wrote:

1. Lobbied chats
2. Send private messages
3. Send broadcast messages
4. easily extensible!

All in actix web, using actors! To test the client, I'd use a simple websocket for either [chrome](https://chrome.google.com/webstore/detail/simple-websocket-client/pfdhoblngboilpfeibdedpjgfnlcodoo?hl=en) or [firefox](https://addons.mozilla.org/en-US/firefox/addon/simple-websocket-client/). Open multiple tabs and send whispers or broadcasts in differnt lobbies!

Happy WebSocketing!
