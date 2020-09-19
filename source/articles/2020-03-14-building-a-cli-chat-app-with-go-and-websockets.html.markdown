---
layout: post
title: Building a CLI chat app with go and WebSockets
date: 2020-03-14 10:00 UTC
tags: [go, WebSockets]

---

With the current global situation and the need to stay home, I decided to finish writing this post that I started during the last Christmas holidays.

During Christmas, I wanted to learn more about the [WebSockets](https://en.m.wikipedia.org/wiki/WebSocket) protocol.
READMORE
For that, I thought a simple chat app would be a good exercise.

I went on the internet and searched for tutorials on how to build a chat app using WebSockets and [Go](https://golang.org/), but I could not find any tutorial that showed a more realistic example. The majority of the samples were echo servers.

I wanted some examples that involved some user interaction.

I wanted to build something that resembles more a real chat like [Slack](https://slack.com/), [WhatsApp](https://www.whatsapp.com/), [Telegram](https://telegram.org/), etc.. There are many examples out there.

Full disclosure I'm not a professional Go developer, but I'm slowly learning for personal projects, and work.

Ok, let's start!!

First, we need to decide which WebSocket library to use, Go provides a WebSocket library, but the [Go team advises](https://godoc.org/golang.org/x/net/websocket) to use other solutions built by the community. You can even see a message when you want to read the doc for the library in GoDoc.


I decided to use [websocket](https://github.com/nhooyr/websocket) because it was the most recent of all and had support for context and rate-limiting.

Once we move away from deciding on the WebSocket library, we need a way to route request to our server, Go provides a fantastic [http](https://godoc.org/net/http) package for doing that. Still, often I find myself reaching for [another solution from the community to build the routes](https://github.com/julienschmidt/httprouter). Not for any technical reason, but it was the first library I used to create a web app some time ago, and I'm sticking with what I know and have worked for me.

Before we continue, I want to make clear that this post is not going to discuss goroutines synchronization, there is fantastic documentation on the [sync package](https://golang.org/pkg/sync) and tones of resources out there. So far, simplicity, I'm going to remove those bits from the code examples.


Fantastic, we are ready to start building the CLI chat app.

So I want to divide the code between `server` and `client`.

The server handles new WebSocket connections, listen to messages on those socket connections and broadcast messages to the right users. Also, it takes care of adding and removing users to different chat rooms.

The client connects to the server, listens to messages that are sent to it and print them. It also allows the users to submit new messages to the server.

Those are going to be our two Go packages.

### Let's start with the server:

The server must provide an endpoint that clients can connect using the WebSocket protocol, for that we are going to use the [Http.Server](https://godoc.org/net/http#Server) abstraction that the Http package provides.

The handler is going to have a single route `/chat/:chat_room/:user_name`.

We have four abstractions: `Hub`, `Chat`, `Message` and `User`.

The `Hub` will hold a list of chats. You can think of the hub as a workspace in Slack; each workspace has different chat rooms.

When a user opens a WebSocket connection to `/chat/general/gustavo`, we are going to check if the chat `general` exists. If not, we will create it, store in the `Hub` and add gustavo to the list of users of that chat.
If the chat room exists, we will check if the user already is there and if not create a new user or return an error message if the user already exists.

```go
func (h *hub) chatRoom(w http.ResponseWriter, r *http.Request, ps httprouter.Params) {
  chatRoom := ps.ByName("chat_room")
  userName := ps.ByName("user_name")
  c, ok := h.rooms[chatRoom]
  if !ok {
    c := h.addChat(chatRoom)
    user, err := newUser(userName, w, r)
    if err != nil {
      log.WithError(err).Fatal("Error creating user to new chat")
    }
    c.addUser(user)
    c.run()
  } else {
    if c.hasUser(userName) {
      log.WithFields(log.Fields{
        "chat":     chatRoom,
        "username": userName,
      }).Info("User already exists in chat room")
    } else {
      user, err := newUser(userName, w, r)
      if err != nil {
        log.WithError(err).Fatal("Error creating user for chat")
      } else {
        c.addUser(user)
      }
    }
  }
}
```

The chat holds a list of users connected to that chat.
The most important thing that the chat does is acting as a coordinator between the multiple goroutines that will work to make listening, broadcasting and managing users possible.

Before we continue, we need to know what a user is. The user holds the name and the WebSocket connection for that particular user.

```go
type user struct {
  name      string
  conn      *websocket.Conn
  listening bool
}
```

The user connection is an important detail because the chat uses those connections both for listening to messages and broadcasting messages to the right client.

Now that we know how the user and chat works let's dive into how we can listen, broadcast and keep the users updated for each chat.

Once we create a chat room, we are going to call the `run` function, which will start three different goroutines.

```go
func (c *chat) run() {
  go c.listen()
  go c.broadcast()
  go c.keepUserListUpdated()
}
```

The `listen` goroutine, loops over all the users on the chat and creates another goroutine to listen to incoming messages. After receiving the message, we send it to the internal messages channel.

```go
func (c *chat) listen() {
  for {
    if len(c.users) > 0 {
      for _, user := range c.users {
        if !user.listening {
          user.listening = true
          go c.listenToUser(user)
        }
      }
    }
  }
}

func (c *chat) listenToUser(user *user) {
  for {
    _, msg, err := user.conn.Read(c.ctx)
    if err == nil {
      c.messages <- message{
        bytes:  msg,
        author: user,
      }
    } else {
      c.dropUsers <- user
      break
    }
  }
}
```

It is essential to mention that if we get an error while listening to a message from a user, we are going to send a message to the `dropUsers` channel. We are going to use this channel on the `keepUserListUpdated` function.

Why do we need a separate goroutine per user? Great question. The act of listening for a message in the WebSocket connection is [a blocking action.](https://github.com/nhooyr/websocket/blob/c62c0dcc9318d1ad612613d433ec90d7c34378dc/ws_js.go#L99-L116) So when iterating over the users, it will stop and wait for the first user to send a message, once the user has sent a message it will iterate to the next user and repeat the process.

If we do not create a different goroutine, we will listen to the messages in the order the users have connected to the chat. But we want to have a real-time experience when sending and receiving messages on our CLI chat app.

The `broadcast` goroutine listens on the `messages` channel, and every time we get a new message, we are going to send it to all the users except to the author of the message.

```go
func (c *chat) broadcast() {
loop:
  for {
    select {
    case message := <-c.messages:
      usersToSend := c.userToSend(message.author)
      bytes, err := message.print()
      if err == nil {
        for _, user := range usersToSend {
          user.conn.Write(c.ctx, websocket.MessageText, bytes)
        }
        c.messagesRead = append(c.messagesRead, message)
      } else {
        log.WithError(err).Warn("Error building the message")
      }
    case <-c.ctx.Done():
      break loop
    }
  }
}
```

Finally, our last goroutine is the `keepUserListUpdated`, which takes care of adding and removing users on each chat.
We remove users from the chat every time there is an error reading from the WebSocket connection. You can check `listenToUser` above to see the logic that handles those cases.

Remember that previously we mentioned that when there is an error listening to users, we send a message to the `dropUsers` channel?

The `keepUserListUpdated` function listens to that channel, and every time a new message comes, it deletes the user from the list of active users in the chat.

```go
func (c *chat) keepUserListUpdated() {
loop:
  for {
    select {
    case user := <-c.addedUsers:
      users := c.users
      users = append(users, user)
      c.users = users
      c.broadcastMessage([]byte(fmt.Sprintf("%s joined %s\n", user.name, c.name)))
    case user := <-c.dropUsers:
      users := c.deleteUser(user)
      c.users = users
    case <-c.ctx.Done():
      break loop
    }
  }
}

```

This function also takes care of adding users to the chat in a similar way, with the addition of broadcasting to all users that a new user joined the chat.

And that is all there is to the server package.

### Let's dive into the client-side of the CLI app.

The client functionality is to connect to the server, listen for new messages, print them and allow the users to send new messages to the server.

In our example, the client store the name of the user, a WebSocket connection to the server and a channel which we will use to send messages.

```go
type client struct {
  userName string
  conn     *websocket.Conn
  ctx      context.Context
  message  chan string
}
```

When using the CLI as a client, we need to pass the chat room and the user who is connecting to it.

`go run client/main.go --user_name=gustavo --chat_room=general`

Executing that line will open a new connection to the server using the WebSocket protocol. `ws://localhost:8080/chat/general/gustavo`, notice that we haven't used `http` or `http` on the URL but rather `ws`.

All this logic is taking care of by the WebSocket library we are using.

Once the client has initialized, we are going to call the `run` method, which will start two new goroutines: `listen` and `getInput`.

The listen goroutine is very similar to the one on the server, but we do not have to loop over the different users since here we only have one user listening for messages.

```go
func (c *client) listen() {
  for {
    _, reader, err := c.conn.Reader(c.ctx)
    if err != nil {
      log.WithError(err).Warn("Error receiving message")
      break
    } else {
      io.Copy(os.Stdout, reader)
    }
  }
}
```

What is that `io.Copy(os.Stdout, reader)` doing? It gets the message from the WebSocket and copies to the terminal [stdout.](https://www.howtogeek.com/435903/what-are-stdin-stdout-and-stderr-on-linux/) Go provides the [Reader and Writer](https://medium.com/@xeodou/understanding-golang-reader-writer-2c855eae0a94) abstractions that help us manipulate data between entities that implement the reader and writer [interface.](https://gobyexample.com/interfaces)


The `getInput` function is a simple [`for`](https://gobyexample.com/for) loop that will listen for the user input coming from the terminal [stdin.](https://www.howtogeek.com/435903/what-are-stdin-stdout-and-stderr-on-linux/)
Once we have a message from the user, we are going to send that message to the `message` channel of the client.

```go
func (c *client) getInput() {
  for {
    in := bufio.NewReader(os.Stdin)
    result, err := in.ReadString('\n')
    if err != nil {
      log.WithError(err).Fatal(err)
    }

    if result != "" {
      c.message <- result
    }
  }
}
```

It makes sure that it is not empty since the user could use the return key, but haven't type anything, and we do not want to send blank messages to the server, that will get broadcast to all the users of the chat.

Finally, we have another `for` loop and [`select`](https://gobyexample.com/select) listening for messages on the `message` channel and every time we get a new message, we are going to write to the server.

```go
func (c *client) run() {
  go c.listen()
  go c.getInput()
loop:
  for {
    select {
    case text := <-c.message:
      err := c.conn.Write(c.ctx, websocket.MessageText, []byte(text))
      if err != nil {
        break loop
      }
    case <-c.ctx.Done():
      break loop
    }
  }
  c.close()
}
```

And this is all that you need to create a CLI chat app with go and WebSockets.
I hope you enjoyed reading as much as I enjoyed writing the app and the article.
If you have any questions, please use the comments below, and I will answer them.

You can find all the code [here](https://github.com/GustavoCaso/go-chat-websockets)

One last thing, I know the functionality is far from complete, but I haven't got the time to work more on it if you are curious and want to improve the app here a list of things you could work on:

- Broadcast all previous messages, when a new user connects to the chat.
- Handle connection retry both on the server and the client.



