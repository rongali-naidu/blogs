# If Everything Is Bytes, Why Do We Talk About Text, Binary, and Raw Data?

When I first learned about files, people kept saying things like:

* "This is a text file."
* "This is a binary file."
* "Give me the raw bytes."

And I remember thinking:

> Hold on...
>
> Doesn't everything eventually become bytes?
>
> My text file is bytes.
>
> My JPEG is bytes.
>
> My Kafka message is bytes.
>
> So why are we acting like text and binary are fundamentally different things?

It turns out they're not different at the storage layer.

The difference exists only in how we interpret those bytes.

Once this clicked for me, a lot of concepts in operating systems, networking, Kafka, databases, and data engineering suddenly became much easier to understand.

---

# Let's Start With a Secret Message

Imagine I hand you this:

```text
72 101 108 108 111
```

What does it mean?

You don't know.

It could be:

* Lottery numbers
* Coordinates
* Random measurements
* A phone number
* Something else entirely

Now suppose I tell you:

> Interpret these numbers as ASCII character codes.

Suddenly:

```text
72  → H
101 → e
108 → l
108 → l
111 → o
```

becomes:

```text
Hello
```

The numbers didn't change.

Only the interpretation changed.

That's exactly how bytes work.

---

# Everything Starts as Bytes

Imagine a file contains these bytes:

```text
0x48 0x69
```

Or in binary:

```text
01001000 01101001
```

To your SSD, memory, operating system, and CPU, that's all it is: two bytes.

There is no built-in concept of:

* Text
* Images
* JSON
* Videos
* Kafka messages

At this level, the storage system only sees:

```text
Byte 1 = 0x48
Byte 2 = 0x69
```

Nothing more.

---

# Storage Is Surprisingly Boring

When you save:

```text
Hello
```

your SSD doesn't think:

> Ah yes, a greeting in English.

When you save a photo, it doesn't think:

> Ah yes, a picture of a dog.

When you save a Netflix viewing event, it doesn't think:

> Ah yes, a user paused a movie.

The SSD only sees patterns of bits:

```text
01001000
01100101
01101100
...
```

Storage doesn't understand meaning.

It only stores bytes.

Meaning exists in the software that reads and writes those bytes.

---

# The Missing Step Most Explanations Skip

When a program saves data, it doesn't store text, images, or objects directly.

First, it converts them into bytes.

The complete lifecycle looks like this:

```text
Meaning / Data Structure
           ↓
Encoding / Serialization
           ↓
Bytes
           ↓
Disk / Network / Memory
           ↓
Bytes
           ↓
Decoding / Deserialization
           ↓
Meaning / Data Structure
```

This is one of the most important ideas in computing.

Storage stores bytes.

Applications create meaning.

Encoders convert meaning into bytes.

Decoders convert bytes back into meaning.

---

# What Is a Text File?

Suppose you write:

```text
Hello
```

Before it can be saved, the characters must be converted into bytes.

Using ASCII or UTF-8:

```text
H → 0x48
e → 0x65
l → 0x6C
l → 0x6C
o → 0x6F
```

Result:

```text
0x48 0x65 0x6C 0x6C 0x6F
```

This process is called encoding.

The flow looks like:

```text
Characters
     ↓ UTF-8 Encoding
Bytes
```

A text file is simply:

> Bytes that are intended to represent characters.

Common examples:

```text
notes.txt
data.csv
config.json
query.sql
```

---

# What Is a Binary File?

Now consider these bytes:

```text
0xFF 0xD8 0xFF 0xE0 0x00 0x10 ...
```

If you try to interpret them as text, you'll mostly see gibberish.

But if a JPEG decoder interprets them according to the JPEG specification, you get an image.

The flow looks like:

```text
Image
   ↓ JPEG Encoding
Bytes
```

Examples of binary files:

```text
image.jpg
video.mp4
archive.zip
program.exe
data.parquet
```

A binary file is simply:

> Bytes that are intended to represent something other than plain text.

---

# The Key Insight

Many beginners think:

```text
Text files = characters
Binary files = bytes
```

But that's not quite right.

The reality is:

```text
Text files   = bytes interpreted as characters
Binary files = bytes interpreted as something else
```

Both are bytes.

The difference is interpretation.

---

# What Does "Raw Data" Mean?

The term raw usually means:

> These bytes have not yet been interpreted.

Imagine receiving data from a network:

```text
0x48 0x65 0x6C 0x6C 0x6F
```

At this point, we simply have bytes.

In Python:

```python
data = b'\x48\x65\x6c\x6c\x6f'
```

Now we decide to decode them:

```python
data.decode("utf-8")
```

Result:

```text
Hello
```

The bytes didn't change.

We simply applied an interpretation.

That's why people often say:

> Give me the raw bytes.

They want the data before any decoding or interpretation occurs.

---

# Kafka Doesn't Know JSON

This idea becomes extremely important in data engineering.

People often say:

> Kafka stores JSON messages.

Not exactly.

Suppose a producer sends:

```json
{
  "user_id": 123,
  "event": "play"
}
```

Before Kafka sees this message, the producer converts it into bytes.

Kafka stores those bytes.

Later, a consumer reads those bytes and decides:

> These bytes should be interpreted as JSON.

Kafka itself has no idea what JSON is.

At its core, Kafka stores:

```text
key   → bytes
value → bytes
```

That's it.

The interpretation happens outside Kafka.

---

# Another Way to Think About It

Imagine receiving these bytes:

```text
0x48 0x65 0x6C 0x6C 0x6F
```

One program might interpret them as text:

```text
Hello
```

Another program might interpret them as part of a custom protocol:

```text
Message Type   = 0x48
Payload Length = 0x65
Flags          = 0x6C
...
```

Same bytes.

Different interpretation.

The bytes don't contain meaning by themselves.

Meaning comes from the rules used to interpret them.

---

# Where Does Base64 Fit In?

Base64 often confuses people because they hear:

> Convert binary to text.

A more accurate statement is:

> Represent arbitrary bytes using text characters.

Suppose we start with:

```text
0x48 0x69
```

which represents:

```text
Hi
```

Base64 encoding produces:

```text
SGk=
```

Now the data consists entirely of printable text characters.

This is useful when a system can safely transport text but may not safely handle arbitrary bytes.

Common examples:

* Email attachments
* JSON payloads
* XML documents
* URLs

The important thing to remember is:

> Base64 does not change the underlying information.

It only changes the representation.

---

# The Base64 Surprise

These all represent the same information:

```text
Hi
```

```text
0x48 0x69
```

```text
SGk=
```

```text
0100100001101001
```

One is plain text.

One is hexadecimal.

One is Base64.

One is binary.

Different representations.

Same underlying information.

---

# The Mental Model I Wish Someone Had Told Me Earlier

Most beginners think:

```text
Text
Binary
Images
Videos
JSON
Kafka Messages
```

are fundamentally different things.

They're not.

A computer ultimately sees all of them as:

```text
Bytes
```

The only difference is the set of rules used to convert meaning into bytes and later recover that meaning from those bytes.

You can think of computing as one giant cycle:

```text
Meaning
   ↓
Encode / Serialize
   ↓
Bytes
   ↓
Store / Transfer
   ↓
Bytes
   ↓
Decode / Deserialize
   ↓
Meaning
```

Text files, JPEGs, Avro records, Protobuf messages, Kafka events, and Parquet files are all variations of this same idea.

They're simply different agreements for converting meaning into bytes.

---

# Final Takeaway

When someone says:

* "This is text data."
* "This is a binary file."
* "Give me the raw bytes."

they are not talking about different physical storage.

They are talking about different levels of interpretation.

The simplest mental model is:

> Storage stores bytes.
>
> Applications create meaning.
>
> Encoders convert meaning into bytes.
>
> Decoders convert bytes back into meaning.

Once that clicks, concepts like UTF-8, JSON, Avro, Protobuf, Base64, Kafka, Parquet, file formats, and network protocols all become variations of the same story:

> How do we convert meaning into bytes and later recover that meaning from those bytes?
