# A Mental Map for Understanding REST, GraphQL, JSON-RPC, HTTP, WebSockets, TCP, and MCP

Having spent most of my time in data engineering, terms like REST, GraphQL, JSON-RPC, HTTP, WebSockets, TCP, MCP, and gRPC always felt related, but I didn't have a simple way to organize them in my head.

After spending some time reading and exploring these topics, I gradually settled on the mental model below. This is not a formal networking model, nor am I presenting it as one. It is simply a mapping that helped me understand how these concepts fit together.

What helped me was thinking about the different questions each technology answers.


**Who am I talking to?**

```text
IP  -> Which machine?
TCP -> Which service/process on that machine?
```

Some answer:

**How are we having the conversation?**

```text
HTTP      -> Structured request/response conversation
WebSocket -> Structured ongoing conversation
stdio     -> Structured local conversation
```

And others answer:

**What are we talking about?**

```text
REST      -> Resources
JSON-RPC  -> Procedures/actions
GraphQL   -> Data and relationships
```

Once I started grouping the concepts this way, it became easier for me to see how they fit together rather than viewing them as a collection of unrelated terms.

---

## Start with a Building

Imagine a large office building.

### IP Address: Which building?

The first question is:

> Which building am I trying to reach?

That's what IP addresses do.

```text
Building A
Building B
Building C
```

IP helps route traffic to the correct machine.

For my purposes, that's all I need to remember:

```text
IP -> Which building?
```

---

## TCP Port: Which person inside the building?

Once I reach the building, there are many people inside.

```text
Building
├── Alice
├── Bob
├── Carol
└── David
```

Each person represents a service running on the machine.

Examples:

```text
443  -> Web Server
5432 -> PostgreSQL
9092 -> Kafka
```

TCP ports help identify who inside the building should receive the conversation.

My simplified understanding became:

```text
IP  -> Which building?
TCP -> Which person inside the building?
```

---

## The Next Question: How Are We Having the Conversation?

Once I have reached the correct person, I need a way to communicate.

This is where HTTP, WebSockets, and stdio come in.

These technologies are not really about the business meaning of the message.

They are more about the rules of the conversation.

### HTTP: Structured Request/Response Conversation

HTTP follows a simple pattern:

```text
Client asks
Server answers

Client asks
Server answers
```

Every interaction follows a request-response model.

For example:

```http
GET /users/123
```

The server responds:

```http
200 OK
```

HTTP defines how requests and responses should be structured.

### WebSocket: Structured Ongoing Conversation

WebSocket changes the conversation style.

Instead of:

```text
Question
Answer
Question
Answer
```

both sides stay connected.

```text
Client <-----------------> Server
```

Either side can send messages whenever it wants.

The conversation remains open.

### stdio: Structured Local Conversation

stdio is different.

There is no network.

There is no IP address.

There is no TCP connection.

Two programs running on the same machine exchange bytes directly.

```text
Program A
    ↔
Program B
```

The conversation still follows agreed rules, but everything happens locally.

This was the piece that helped me understand how MCP can communicate with local tools without involving a network.

---

## Another Question: What Are We Talking About?

This is where my mental model initially became a little fuzzy.

At first I thought HTTP only answered:

> How are we having the conversation?

But then I realized many applications communicate using HTTP directly.

For example:

```http
GET /users/123
```

In this case, the HTTP request itself already carries the business meaning.

The URL identifies the resource and the HTTP method describes the action.

There is no additional language sitting inside the request.

Conceptually:

```text
HTTP
└── Business Meaning
```

This is common in REST-style APIs.

However, other approaches add another layer on top of HTTP.

For example:

```text
HTTP
└── GraphQL
```

or

```text
HTTP
└── JSON-RPC
```

In these cases, HTTP provides the structure of the conversation, while GraphQL or JSON-RPC provide the language used inside that conversation.

So a more accurate way to think about it is:

```text
REST often uses HTTP directly as its language.

GraphQL often rides inside HTTP.

JSON-RPC often rides inside HTTP.
```

This distinction helped me understand why REST felt different from GraphQL and JSON-RPC.

REST often expresses meaning directly through HTTP itself, while GraphQL and JSON-RPC usually place their meaning inside the HTTP message.

### REST: Talk About Resources

REST encourages thinking in terms of things.

Examples:

```text
User
Movie
Order
Document
```

Typical requests become:

```text
Get User 123
Delete Order 456
Create Movie
```

REST is essentially a resource-oriented language.

### JSON-RPC: Talk About Actions

JSON-RPC thinks in terms of actions.

Examples:

```text
getUser(123)
searchMovies("Matrix")
calculatePrice()
```

The conversation is about invoking procedures.

This is why the name RPC (Remote Procedure Call) started to make sense to me.

### GraphQL: Talk About Data

GraphQL did not fit neatly into either REST or RPC in my head.

The simplest explanation I found is:

REST asks for resources.

RPC asks for actions.

GraphQL asks for exactly the data shape you want.

Example:

```text
User
  name
  email
  orders.total
```

The client describes the data it wants and the server returns that shape.

---

## Where Do These Messages Go?

One question I kept asking was:

> If REST, GraphQL, and JSON-RPC describe what I want to say, where do those messages actually live?

### HTTP

When GraphQL or JSON-RPC are used over HTTP, they are typically carried inside the HTTP request.

Conceptually:

```text
HTTP Request
└── GraphQL or JSON-RPC Message
```

### WebSocket

With WebSockets, the message is carried inside a WebSocket message.

```text
WebSocket Message
└── GraphQL or JSON-RPC Message
```

### stdio

With stdio, the message is written directly into the stream between two processes.

```text
stdio Stream
└── JSON-RPC Message
```

The message itself does not necessarily change.

Only the conversation mechanism changes.

---

## Why MCP Started Making More Sense

One thing that initially confused me was hearing that MCP uses JSON-RPC over stdio.

Using the mental model above, that can be read as:

```text
What are we talking about?
    JSON-RPC

How are we having the conversation?
    stdio
```

There is no network involved.

The same JSON-RPC message could also travel through:

```text
JSON-RPC
    +
HTTP
```

or

```text
JSON-RPC
    +
WebSocket
```

The message stays the same.

Only the conversation mechanism changes.

---

## The Mental Model I Use Now

### Who am I talking to?

```text
IP  -> Which building?
TCP -> Which person inside the building?
```

### How are we having the conversation?

```text
HTTP      -> Structured request/response conversation
WebSocket -> Structured ongoing conversation
stdio     -> Structured local conversation
```

### What are we talking about?

```text
REST      -> Resources
JSON-RPC  -> Procedures/actions
GraphQL   -> Data and relationships
```

This may not be a textbook explanation, but it is the mental map that helped me connect these concepts together and understand how they relate to one another.
