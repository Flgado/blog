+++
title = 'Tracing MongoDB Queries with eBPF: From Syscalls to Latency Insights'
date = 2025-11-05T11:31:07+01:00
draft = false
+++

# Introduction
In this blog, I’ll dive into a recent side project I built called mongo-pulse.

This small tool shows how powerful eBPF can be for tracking system calls to measure MongoDB query latency — and it even gives you some juicy details about the query payload, request ID, and how long each query took to run. Basically, it helps you spot slow queries in your application without needing any instrumentation (or you could just use Mongo Atlas Query Insights… but where’s the fun in that, right? 😅).

I’ll walk through some of the implementation details and design decisions, explain which system calls are involved, and touch on the Mongo wire protocol — without diving too deep into the protocol rabbit hole.

I tested everything on **Linux kernel 6.8.0-86-generic**, using the **Go MongoDB client**, **Node.js MongoDB client** and **Python mongo client**
*(Spoiler: they use system calls quite differently — and yes, it’s kind of funny once you notice it!)*

Ah, and this blog isn’t about what eBPF, maps, or helpers are — there are already plenty of great posts on those topics.

- [Source Code](https://github.com/Flgado/mongo-pulse)

# Background: What We Need to Know First

Before getting our hands dirty with the implementation, let’s quickly review a few concepts that make the rest of this post easier to follow.

## The MongoDB Wire Protocol (aka How Mongo Talks)

### Header

MongoDB uses a binary protocol called the `Mongo Wire Protocol` for communication between clients and the server. Every message starts with a standard header that looks like this:

``` go
struct MsgHeader {
    int32   messageLength; // total message size, including this
    int32   requestID;     // identifier for this message
    int32   responseTo;    // requestID from the original request
                           //   (used in responses from the database)
    int32   opCode;        // message type
}
```
Since `mongo-pulse` doesn’t support compression yet, we’ll focus only on this header. This structure is at the heart of every MongoDB message — it’s how the client and server identify, match, and interpret requests and responses.

In user space, we’ll mainly use the `requestID` and `responseTo` fields (with a little help from kernel-level data — more on that later) to map requests to their corresponding responses and measure latency.

Another important field is `opCode`.
Before MongoDB v5.1, this could represent many different message types. But since 5.1, there are only two that matter:

- **2012** → Compressed message

- **2013** → Standard (uncompressed) message

In our eBPF program, we’ll focus on messages with `opCode = 2013`, since:

1. Compression isn’t supported yet, and
2. That’s the code used by MongoDB 5.1+ for standard communication.

So, whenever we detect a system call involving a message with `opCode = 2013`, we know it’s part of the MongoDB query flow — and that’s where the fun begins!

- **References**:
 - [MongoDB Wire Protocol - Standard Messsage Header](https://www.mongodb.com/docs/manual/reference/mongodb-wire-protocol/#standard-message-header)

### Synchronous Behavior (Thank You, MongoDB!)

According to the MongoDB wire protocol specification:

"Single Track: A connection MUST limit itself to one request/response at a time. A connection MUST NOT multiplex or pipeline requests to an endpoint."

In plain English: every MongoDB connection handles one request at a time — no multiplexing, no parallel messages on the same socket.

And that’s fantastic news for us! 🎉
Because this synchronous behavior makes our job of mapping requests and responses much easier.

Each MongoDB client connection corresponds to a single socket, and each process that opens that socket has its own process ID (pid). When we observe system calls in the kernel, we can use the combination of `pid` (the process making the call) and `fd` (the file descriptor representing the socket) to uniquely identify the connection.

- **pid** → identifies the process (e.g., your Node.js or Go app)
- **fs** → identifies the file descriptor, which is essentially the “handle” the process uses to talk to MongoDB
- **Together**, `pid + fs` uniquely identify a socket — and since Mongo doesn’t multiplex, that socket can only be handling one query at a time.

Since Mongo doesn’t multiplex, that socket can only handle one query at a time — so we can confidently match a `write()` (the request) with a `read()` (the response) using just those two values.

MongoDB’s “no pipelining” rule basically saves us from a lot of eBPF headaches. Thanks, MongoDB, for keeping it simple (for once) 😅

- **References**:
    - [MongoDB Specifications - Connection Monitoring and Pooling](https://github.com/mongodb/specifications/blob/master/source/connection-monitoring-and-pooling/connection-monitoring-and-pooling.md)


## System Calls in Play (aka eBPF’s Gossip Sources 🕵️‍♂️)

Before eBPF can work its magic, we need to decide where to listen in.
System calls are perfect for this — they’re the “checkpoints” every user-space program passes through to ask the kernel for help. Whether your code is reading a file, writing to a socket, or connecting to a database, at some point it’s whispering:

 “Hey kernel, could you do this thing for me?”

So, if we hook those syscalls, we can observe what the application is doing — without modifying it at all.

### 🧾 write() — Sending Data Out
**Normal purpose**:
`write(fd, buf, count)` is used by applications to send data somewhere — it could be writing bytes to a file, sending an HTTP request over a socket, or even logging something to stdout.

The first argument (`fd`) identifies where to write, and the buffer contains the actual data.

**In our context:**

When a MongoDB client executes a query (like `find()`), it serializes that command into the **Mongo Wire Protocol** and sends it through a **TCP socket** to the database.
That’s exactly what this syscall does — it’s the client saying:

    “Hey MongoDB, here’s my query. Please be nice.”

By hooking `sys_enter_write`, mongo-pulse can see when a query is sent, how big it is, and (if we peek into the buffer) details like the standard header.

### 📖 read() — Getting Data Back

**Normal purpose:**
`read(fd, buf, count)` is the mirror image of `write()`. It asks the kernel to read up to count bytes from the object identified by `fd`.
If that fd is a socket, this call will block until data arrives (unless it’s non-blocking).

**In our context:**
After the client sends a query, it waits for MongoDB’s response — that happens right here.
Hooking both `sys_enter_read` and `sys_exit_read` gives us the exact time we receive the response. Playing with write and read and using the power of maps you will see how we measure the latency.

Here’s a real strace example from the Go client:

``` go
[pid 29315] write(8, "\221\0\0\0\24\0\0\0\0\0\0\0\335\7\0\0\0\0\0\0\0|\0\0\0\2find\0\6"..., 145) = 145
[pid 29315] read(8, "=\0\0\0\24\0\0\0\335\7\0\0\0\0\0\0\0005\2\0\0\3cursor\0\34\2\0"..., 582) = 582
```

Let’s translate:
1. `write(8,...)` → send 145 bytes of query data through socket 8.
2. `read(8, …)` → receive 582 bytes back from MongoDB through that same socket.

you can understand how go client know the exact numbers of bytes it will receive! We will get into that, and was really fun this discovery !

Because MongoDB connections are synchronous (one request at a time), we can easily match these two calls using `(pid, fd)` to compute the query duration.

### 🔌 connect() — Making Friends on the Network

**Normal purpose:**
`connect(fd, sockaddr, addrlen)` is used to initiate a connection to another process — usually over the network.
For TCP sockets, this means starting the `three-way handshake (SYN, SYN-ACK, ACK)`

**In our context**:
When a MongoDB client connects to `localhost:27017` (or any Mongo URI), it calls connect().
By hooking `sys_enter_connect`, we can detect when a new MongoDB connection is created, record its IP/port, and tag it as “a socket we care about.”
That way, later on, we can safely ignore unrelated syscalls from the same process (like logging or HTTP calls)..

### 🚪 close() — Saying Goodbye (Politely)

**Normal purpose**:
`close(fd)` releases the file descriptor so the kernel can free up resources.
Whether it’s a file, socket, or pipe — once you call `close()`, that fd is dead to you.

**In our context**:
When the MongoDB client finishes or the connection pool recycles a socket, it’ll eventually call `close()`.
Hooking sys_enter_close allows us to clean up any internal state we’re tracking (like maps of pid + fd → query metadata).

Think of it as our cleanup crew — no dangling sockets left behind!


# My implementation

Alright, let’s get to the funny part — the implementation!
In this section, I’ll walk through what happens in each eBPF hook we talked about earlier. Hopefully, by the end, everything clicks together like a well-tuned kernel module (minus the panic 😅).

## Connect()

Let’s start at the beginning of every database conversation — the connection.

This hook is triggered every time my service calls `connect()`, i.e., when it opens a TCP connection to the MongoDB server.
Right now, I’m mainly using it to record the connection’s IP and port, which I can later associate with queries.

To be honest, I’m still deciding how much value this adds — maybe I’ll use it to `track connection pool behavior` in the future. For now, it’s mostly there to give extra context like “which connection this query went through.”

The data is stored in an eBPF map called connections.
Later, in the write() hook, I’ll show how this gets linked to queries.

🧠 Implementation Details

Here’s the eBPF hook for connect():

``` C
SEC("tp/syscalls/sys_enter_connect")
int handle_connect(struct trace_event_raw_sys_enter *ctx) {
    struct sockaddr sa;
    struct sockaddr *addr_ptr = (struct sockaddr *)ctx->args[1];
    
    if (!addr_ptr) {
        return 0;
    }

    if (bpf_probe_read_user(&sa, sizeof(sa), addr_ptr)) {
        return 0;
    }

    struct sock_key key = {
        .pid = bpf_get_current_pid_tgid() >> 32,
        .fd = ctx->args[0],
    };

    struct connection_event_t event = {
        .fd = ctx->args[0],
    };

    if (sa.sa_family == AF_INET) {
        struct sockaddr_in sa4;
        if (bpf_probe_read_user(&sa4, sizeof(sa4), addr_ptr)) {
            return 0;
        }
        
        event.port = bpf_ntohs(sa4.sin_port);
        
        event.dst_ip[10] = 0xff;
        event.dst_ip[11] = 0xff;
        __u32 ip4 = sa4.sin_addr.s_addr;
        event.dst_ip[12] = (ip4 >> 0) & 0xff;
        event.dst_ip[13] = (ip4 >> 8) & 0xff;
        event.dst_ip[14] = (ip4 >> 16) & 0xff;
        event.dst_ip[15] = (ip4 >> 24) & 0xff;
        
        event.flags = 0;
    } else if (sa.sa_family == AF_INET6) {
        struct sockaddr_in6 sa6;
        if (bpf_probe_read_user(&sa6, sizeof(sa6), addr_ptr)) {
            return 0;
        }
        
        event.port = bpf_ntohs(sa6.sin6_port);
        
        *(__u32 *)&event.dst_ip[0] = sa6.sin6_addr.in6_u.u6_addr32[0];
        *(__u32 *)&event.dst_ip[4] = sa6.sin6_addr.in6_u.u6_addr32[1]; 
        *(__u32 *)&event.dst_ip[8] = sa6.sin6_addr.in6_u.u6_addr32[2];
        *(__u32 *)&event.dst_ip[12] = sa6.sin6_addr.in6_u.u6_addr32[3];
        
        event.flags = 1;
    } else {
        return 0;
    }

    bpf_map_update_elem(&connections, &key, &event, BPF_ANY);

    return 0;
}
```

🧩 What’s Happening Here

Let’s break this down step-by-step:

1. **Grab the connection info**:
We extract the socket address from the syscall’s arguments and safely read it from user space with `bpf_probe_read_user()` — because in eBPF, direct pointer dereferencing is a one-way ticket to rejection by the verifier. 🚫

2. **Build a key**:
The key for our map is a combination of the process ID (pid) and file descriptor (fd).
This `(pid, fd)` pair uniquely identifies a connection.

3. **Detect address family**:
We handle both IPv4 (AF_INET) and IPv6 (AF_INET6).

- For IPv4, we manually expand the 4-byte IP into a 16-byte array (IPv6-mapped format).
- For IPv6, we copy the full address directly.
The flags field marks which type it is.

4. **Store the connection**:
This is where things start to get interesting — the moment your app actually sends a query to MongoDB.

## Write()

Snipped of the code:

``` C
struct mongodb_header {
    __u32 message_length;
    __u32 request_id;
    __u32 response_to;
    __u32 op_code;
};

 if(buf_len < sizeof(struct mongodb_header)){
        return 1;
    }

    struct mongodb_header header;
    if (bpf_probe_read_user(&header, sizeof(header), buf)){
        return 1;
    }

    if (header.op_code != OP_MSG){
        return 1;
    }
```
This is the part where I try to detect whether the write() system call is actually sending MongoDB traffic or just some random data.

Let’s break it down:

1. **Check the buffer size**
The first thing I do is check if the buffer size (`buf_len`) is smaller than the size of a MongoDB message header (remember that from earlier?).
If it’s smaller, I can safely ignore it — there’s no way it’s a valid Mongo message.

2. **Copy the header from user space**
Using `bpf_probe_read_user()`, I read the first few bytes from the buffer into my mongodb_header struct. This lets me safely inspect the header without breaking the eBPF verifier’s heart 💔.

3. **Check the op_code**
Finally, I look at the op_code field — the fourth value in the header.
If it’s 2013, that means the message is of type OP_MSG, the standard (uncompressed) MongoDB message format used since version 5.1.
That’s my signal: this system call is part of a MongoDB query! 🚀

If the `op_code` doesn’t match, I just skip it — not every `write()` is Mongo-related, after all.

### 🧩 Building and Sending the Query Event

Now that I’ve identified a `write()` call as a valid MongoDB request, the next step is to build an event I can recognize later — when I see the corresponding response in `read()`.

For that, I need a unique key to identify this specific connection and query.

🗝️ **Creating the Key**

Since multiple processes (and even threads) can talk to MongoDB, I combine two pieces of information that together are guaranteed to be unique for a single connection:

- **pid** → identifies the process (e.g., your Go or Node.js client)

- **fd** → identifies the file descriptor for the socket in that process

Together they form our unique key (`pid + fd`):

``` C
struct sock_key key = {
    .pid = pid,
    .fd = fd,
};
```

This same key will later help us match the query with its response, and also link it back to connection info (like IP and port) stored earlier.

🧱 Building the mongo_query_event

Once I have the key, I start assembling the data that I want to send up to user space — my custom structure called `mongo_query_event`.

This includes:

1. **Connection info** from the connections map (the IP and port recorded in the connect() hook).

2. The `pid` and `fd` from the current syscall.

3. The requestId extracted earlier from the MongoDB message header.

4. Up to `1024` bytes from the payload — not because I love round numbers, but because eBPF doesn’t let you go wild with memory. The kernel verifier keeps you on a short leash, so grabbing a kilobyte of data is a safe way to peek at the query without getting yelled at by the verifier.

After building all that, I send the event to user space through a BPF ring buffer called mongo_queries.

On the user-space side, the tool stores this data in a map so that when the response arrives (from the read() hook), it can match them and print the full query info along with its latency.


⏱️ **Tracking the Start Time**

Before sending the event, I also record the current timestamp.
This lets me calculate how long MongoDB took to reply later.

Here’s the helper function that handles both the ring buffer submission and time tracking:

``` C
static __always_inline void send(struct mongodb_event *event) {
    if (event == NULL) {
        return;
    }

    struct sock_key key;
    key.pid = event->pid;
    key.fd = event->fd;

    // Send event to user space
    bpf_ringbuf_submit(event, 0);

    // Save the start time for latency calculation
    struct mongodb_timing_t timing_entry = {
        .start_time = bpf_ktime_get_ns()
    };
    bpf_map_update_elem(&timing, &key, &timing_entry, BPF_ANY);
}
```

So what’s happening here?

- The event is pushed to the ring buffer, where user space can consume it asynchronously.

- At the same time, the hook stores the current time (bpf_ktime_get_ns()) in the timing hash map under the same (pid + fd) key.

When the corresponding read() event happens later, I’ll look up that timestamp to compute the query latency.

It’s a bit like dropping a pin in time —

    “MongoDB query sent at 10:42:17.623... Let’s see when it comes back.”

## read() — Where the Magic (and Latency) Happens

This is where I finally calculate the query latency!
But before getting there… oh boy — this is where things got interesting (and slightly painful).

Just like in the write() hook, I first need to detect if a read() system call is MongoDB-related.
Easy, right? Well, nope — because clients behave very differently here. 😅

🧩 **Different Clients, Different Behaviors**

When using the **Mongo shell** or the **Node.js** driver, everything is nice and simple:
the client reads the entire MongoDB response in a single read() system call.
Life is good, everything fits nicely in one go.

But then comes the **Go MongoDB driver**, and things get funnier.

The Go client first reads **only the first 4 bytes** of the message — that’s the `messageLength` field in the MongoDB header.
Why? Because those 4 bytes tell the client exactly how big the message will be, including the header itself.
So Go does this small pre-read to know how much memory to allocate for the full message.
Smart move! (and also mildly annoying for me 😅).

Here’s what that looks like in an *`strace` trace from a Go client:

``` bash
[pid 29315] write(8, "\221\0\0\0\24\0\0\0\0\0\0\0\335\7\0\0\0\0\0\0\0|\0\0\0\2find\0\6"..., 145) = 145
[pid 29315] read(8, "J\2\0\0", 4)       = 4
[pid 29315] read(8, "=\0\0\0\24\0\0\0\335\7\0\0\0\0\0\0\0005\2\0\0\3cursor\0\34\2\0"..., 582) = 582
```

That second line is the 4-byte sneak peek. Beautiful — and also the reason I had to adjust my logic in the eBPF program. Curiose how other clients do this !

🧠 **Handling Partial Reads in eBPF**

So on the kernel side, I had to handle both cases:
- Clients that read everything at once (easy mode ✅)
- Clients like Go that read the message in chunks (boss level 😈)

Here’s a simplified snippet from the `read()` hook that shows how I deal with this:

``` C
__u32 pid = bpf_get_current_pid_tgid() >> 32;

if (bytes_read < sizeof(struct mongodb_header)) {
    return 1;
}

// Some drivers (like Go) read partial headers — handle carefully
__u32 first_bytes = 0, second_bytes = 0, third_bytes = 0, fourth_bytes = 0;
if (bytes_read >= 4)  bpf_probe_read_user(&first_bytes, 4, buf);
if (bytes_read >= 8)  bpf_probe_read_user(&second_bytes, 4, (char*)buf + 4);
if (bytes_read >= 12) bpf_probe_read_user(&third_bytes, 4, (char*)buf + 8);
if (bytes_read >= 16) bpf_probe_read_user(&fourth_bytes, 4, (char*)buf + 12);

struct mongodb_header header;

// Case 1: Drivers that read the full message at once
if (fourth_bytes == OP_MSG) {
    if (bpf_probe_read_user(&header, sizeof(header), buf)) {
        return 1;
    }
// Case 2: Drivers that read in parts (like Go)
} else if (third_bytes == OP_MSG) {
    header.message_length = 0;
    header.request_id     = first_bytes;
    header.response_to    = second_bytes;
    header.op_code        = third_bytes;
} else {
    return 1;
}
```

So what’s going on here?

- I carefully read the first 16 bytes of the user buffer — 4 bytes at a time — so I can reconstruct the MongoDB header no matter how much the client decided to read.

- If the op_code (the 4th field) matches OP_MSG (value 2013), I know it’s a MongoDB message.

- Then, if the read looks partial (like in the Go case), I manually piece together the header fields.

This part took me way too long to get right, but once I realized why the Go driver behaved that way… it was oddly satisfying. 😎

⏱️ **Calculating the Latency**

Then, you probably already know what’s coming next…
Time to measure how long that query actually took!

I grab the timestamp I previously saved in the `timing` map (remember that from the `write()` section?) and calculate the difference between that and the current time.
**Voilà — query latency!** 🎯

After that, I send a new timing event to user space through another ring buffer called query_timings, containing:
- pid
- fd
- requestId
- and `delta` (the latency in nanoseconds)

On the user-space side, I’ve got a small Go routine constantly reading from this buffer.
As soon as it receives a timing event, it looks up the matching entry in the `mongo_query` map, grabs the query details, and prints everything together.


## 📊 Results

So what does all this actually look like in action?
Here’s a real example of what mongo-pulse prints out when it detects a MongoDB operation:

``` go
🔥 MongoDB Operation Detected!
PID: 144431
FD: 8
Request ID: 16
 ⏱️  Response Time: 895894 ns
 🌐 Connection: 127.0.0.1:27017
 📝 All Readable Strings: [hello helloOk $db admi]
 🔒 This query was encrypted
```

Pretty cool, right?
You get all the juicy details: process info, connection info, and even a sneak peek at the query payload — all without adding any instrumentation to your code.

🧩 **More Examples**

Here are a few more snippets showing different MongoDB operations and their response times:

```
⏱️  Response Time: 1536027 ns
📝 All Readable Strings: [aggregate users pipeline $lookup from orders localField _id foreignField user_id as user_orders $unwind $user_orders $group total_spent $sum $user_orders.amount orders_count $sum _id $_id name $first $name $sort total_spent $limit cursor lsid id &|H C^ $db testd]

--------
📝 All Readable Strings: [find users filter age $gt 4@ limit lsid id &|H C^ $db testd]
-------

------
⏱️  Response Time: 15208394 ns
📝 All Readable Strings: [insert testcol ordered lsid id ~Nt hH $db testdb documents _id l_ data test_data_0 id _id l_ id data test_data_1 _id l_ id data test_data_2 _id l_ id data test_data_3 _id l_ id data test_data_4 _id l_ id data test_data_5 _id l_ id data test_data_6 _id l_ id data test_data_7 _id l_ data test_data_8 id _id l_ id data test_data_9 _id l_ data test_data_10 id _id l_ id data test_data_11 _id l_ id data test_data_12 _id l_ id data test_data_13 _id l_ id data test_data_14 _id l_ id data test_data_15 _id l_ id data test_data_16 _id l_]


-------
⏱️  Response Time: 1312401 ns
📝 All Readable Strings: [getMore +;| collection testcol batchSize lsid id =^ $db testd]

--------------
 ⏱️  Response Time: 9700951 ns
 📝 All Readable Strings: [aggregate testcol pipeline $match id $gte $group _id $id count $sum $sort _id $limit cursor lsid id =^ $db testd]

 -----------
⏱️  Response Time: 9700951 ns
 📝 All Readable Strings: [aggregate testcol pipeline $match id $gte $group _id $id count $sum $sort _id $limit cursor lsid id =^ $db testd]
```

## 🕵️‍♂️ Wait, But This Guy Knows the Data Is Encrypted… Right?
Hehe, of course I didn’t forget that! 😏
You’re right — in most cases, the data sent over the wire is indeed encrypted.

Most modern applications handle TLS termination entirely in user space, using libraries like OpenSSL (or equivalents). These libraries manage all the complex encryption and decryption internally before any data ever reaches the kernel.
That means any packet-level observation from kernel space (like what you’d get from `tcpdump` or socket-level eBPF hooks) will only see ciphertext.

So… is it impossible to see the plaintext?

👉 Nope!

This is where eBPF gets really interesting.

eBPF lets you attach probes directly to **user-space functions** — these are called `uprobes` (for function entry) and `uretprobes` (for function return).
You can use them on any ELF executable or shared object, as long as the symbols aren’t stripped.

By hooking specific OpenSSL functions like `SSL_write_ex` and `SSL_read_ex`, you can catch the data **right before it gets encrypted (on SSL_write_ex) or right after it’s decrypted (on SSL_read_ex)**.
In other words, you get a peek at the plaintext TLS traffic, just inside the app — right before it puts on (or takes off) its encryption disguise. 🔍✨

**Why _ex Functions?**

Newer versions of OpenSSL (≥ 1.1.1) introduced these `_ex` APIs — for example, SSL_write_ex and SSL_read_ex.
They use safer types (size_t instead of int) and clearer semantics.
Modern runtimes (like Python 3, Node.js, etc.) typically link against these updated APIs.
So if you hook the older SSL_write or SSL_read functions, you might see… nothing (I’ve been there 😅).
But as soon as you switch to the _ex versions — everything comes to life.

Most languages, like Python and Node, rely on the **system’s OpenSSL library**, often located at something like `/usr/lib/x86_64-linux-gnu/libssl.so.3`.
So by attaching uprobes there, you’re effectively tracing what their TLS layer is doing — before the data ever leaves user space.

(For example Go and Java are exceptions — they implement their own TLS stacks, so no libssl calls to hook.)

In my next blog, I’ll apply what I have build here to a Go application. It will be interesting to demonstrate how eBPF can attach to a specific function in a binary (spoiler alert).

🧠 **Why Non-Stripped Functions Matter**

When a binary or shared library is not stripped, it still contains symbol information — essentially, human-readable function names like SSL_write_ex, SSL_read_ex, etc.
This makes it straightforward for tools (including eBPF) to locate and attach probes to specific functions.
If the binary is stripped, all that metadata is removed, leaving only raw addresses — you can still hook functions, but it’s much harder because you need to know their exact memory offsets.
That’s why using system libraries (like OpenSSL) is convenient — they’re almost always shipped with symbols intact.

**TL;DR**:
Even though TLS encrypts data before it hits the network, eBPF can hook into user-space functions where the data is still in plaintext.
By tracing `SSL_write_ex` and `SSL_read_ex`, you can observe what’s really being sent or received — right before or right after encryption.
Pretty wild, huh? 🔓🐝

{{< figure src="/images/post_5/libssl_so.png" alt="Traces" title="The power of eBPF in action" class="center" >}}



## 🧠 Conclusion
Building `mongo-pulse` started as a simple experiment to track MongoDB query times — but it quickly became a deep dive into how clients, drivers, and the kernel interact.

By tracing syscalls like `connect`, `write`, and `read`, we uncovered how MongoDB clients communicate at a level that’s usually invisible. The synchronous nature of the wire protocol made it surprisingly straightforward to map requests and responses, while eBPF gave us a safe, zero-instrumentation window into that behavior.

The best part?
We did all this `**without changing a single line of application code** — just by observing what’s already happening.

In the next post, we’ll take things up a notch: moving from system calls to user-space (others that are not so easy as SSL_READ and SSL_WRITE!) function tracing with eBPF uprobes. We’ll attach directly to Go runtime functions and peek inside the binary itself — turning eBPF from a tracing tool into something that feels like developer superpowers. ⚡

Stay tuned — it’s going to be fun. 🧠🐝