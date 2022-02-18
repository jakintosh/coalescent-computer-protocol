# Coda Protocol

The `Coda Protocol` is a set of binary representations and procedures for storing and transmitting ***co***alescent ***da***ta. It is similar to binary serialization formats like CBOR, DER, and MessagePack, but is slightly more opinionated about what you are doing with. The protocol is designed to provide a foundational data layer for the Coalescent Computer, which is a networked computing environment built to replace the World Wide Web. In the context of the Coalescent Computer, coda it is not just a serialization format, but also the ***virtual file system***. As such, the protocol must be capable of supporting any possible data type, and must be flexible enough to be adapted for speed and size constraints in many different storage and networking environments.


## What is Coalescent Data?

### Overview
Coalescent data are pieces of immutable information that additively create a conceptual whole. Coalescent data is replicated in a distributed manner across any number of nodes, and is reconstructed as needed by querying the network, which can be peer-to-peer, client-server, [client-swarm](https://github.com/jakintosh/coalescent-swarm-rs), or any other conceivable configuration. Coalescent data is **always** defined within a concrete schema, and that schema is an inherent part of the data itself.

A complete representation of coalescent data requires the full schema definitions of all nested data, as well as the bytes of data itself.

### An Introductory Example

> Since `Coda` is a binary format, this example will be written in a loose syntax that looks like any programming langauge with curly brackets; pretend it's Rust, C#, JavaScript, or whatever makes you most comfortable.

The following schema describes a person.

```coda
schema Person {
    birthday: Date,
    name: UTF8String,
}
```

Because coalescent data must be fully self describing, a valid representation of an instance of data must also include the schema for the `Date` type (`UTF8String` is a built-in type):

```
schema Date {
    day: UInt8,
    month: UInt8,
    year: UInt32,
}
```

These two schemas can then be compiled into their deterministic [Coda Schema](#coda-schema) representation, and have those bits hashed to produce a content identifier:

```
schema Date {...} => b3701287bab07sbe00c0r0808080ad0e8c012082b0c0808d08dae8671276ac76
schema Person {
    birthday: b370...ac76
    ...
} => a05eca7be19bcd636319c0a0edc07070273565c6e0cd071204cc307c0aa2399a
```

In the example, you can see that before `Person` is hashed, its references to the human readable schema names are replaced by that schema's content-hash. Since `Date` is composed of base data types, it doesn't need any replacement.

With dependencies resolved and schemas compiled, we can now create a complete instance of coalescent data:

```
let instance = {
    schemas: {
        b370...ac76:  {
            day: UInt8,
            month: UInt8,
            year: UInt32,
        },
        a05e...399a: {
            birthday: b370...ac76,
            name: UTF8String,
        },
    }
    data: {
        {
            1,
            1,
            1970,
        },
        "jane doe",
    }
}
```

You might think this is overkill for a simple struct of data that contains a name and a birthday, and you'd be right. However, only in the most robust archival formats will a single piece of data ever contain all of these pieces together like this. In most situations, all of the schemas will be separated from the data.

```
let schemas = {
    b370...ac76: {
        day: UInt8,
        month: UInt8,
        year: UInt32,
    },
    a05e...399a: {
        birthday: b370...ac76,
        name: UTF8String,
    },
}

let instance = {
    schema: a05e...399a,
    data: {
        {
            1,
            1,
            1970,
        },
        "jane doe",
    }
}

```

Now we can see that an instance only contains its schema identifier, as well as the raw data itself. In a transmission context, we can agree upon mappings of smaller integers to our schemas to save on transmission as well.

```
let schemas = {
    b370....ac76: {
        day: UInt8,
        month: UInt8,
        year: UInt32,
    },
    a05e...399a: {
        birthday: b370....ac76,
        name: UTF8String,
    },
}

let transmision_context = [
    a05e...399a,        // mapped to index 0
    b370...ac76         // mapped to index 1
]

let received_data = {
    schema: 0,          // index in transmission context
    data: {
        {
            1,
            1,
            1970,
        },
        "jane doe",
    }
}
```

The componentization of schema and data allow us to be flexible in our implementations depending on our needs. In situations where we are dealing with large blobs of heterogenous data, we can prefix each blob with it's type hash instead of creating a map. If we are sending thousands of the same type of message logs per second across a network channel, we can create a transmission context for that one schema and send only the raw bits to reduce transmission load.

## Specification 

- **Schema**: array of (identifer, type)
- **Map**: array of buffer pointers
- **Buffer**: contiguous buffer of data bytes

All of these components are constructed seperately. To work with a buffer (an instance of coalescent data), you must first create a context that provides a schema and a map. In contexts where all instances of a given schema are a fixed size, the map can be constructed from the schema and be reused for all buffers. In contexts where elements inside the buffer are not a fixed size, each buffer must provide its own map.

Everything is stored in bytes, so that data can be read from natively without parsing at the bit level.

### Binary Representations

what does a type declaration look like? can type declarations have associated data? for example, a fixed size array, how do you attach the fixed size? am i worrying too much about optimizing the schema definition when those will be transmitted extremely infrequently? the important part is to make it as easy as possible to reuse maps, and to keep buffers aligned with memory buckets. in order to maximize map reuse, we need to make it easy to have fixed sizes in the buffer; if this means dynamic sizes in the schema, then that's okay. so if we need to have a dynamically sized type declaration in the schema to make it easier to have fixed length arrays, that's a good trade.

#### Schema
```
[
    [
        [length prefix][utf8 bytes],
        [type id](optional)[length prefix][optional data],
    ],
    [...],
]
```

#### Map
```
[
    (0) 0x00 00 00 00 00 00 00 00,
    (1) 0x00 00 00 00 00 00 00 01,
    (2) 0x00 00 00 00 00 00 00 02,
    (3) 0x00 00 00 00 00 00 00 0F,
]
```

#### Buffer
```
[
    0x00 ... 0x00
]
```

### Built-in Types

#### Bytes
These values are always a fixed one byte
```
false => 0x00
true  => 0xFF
byte  => 0x__
```

#### Integers
These values are always a fixed size (the bits in their suffix)
```
--Type---
|-ID-|--Value
UInt8  => 0x10  0x00
UInt16 => 0x11  0x00 00
UInt32 => 0x12  0x00 00 00 00
UInt64 => 0x13  0x00 00 00 00 00 00 00 00
Int8   => 0x14  0x00
Int16  => 0x15  0x00 00
Int32  => 0x16  0x00 00 00 00
Int64  => 0x17  0x00 00 00 00 00 00 00 00
```

#### Floats
These values are always a fixed size (the bits in their suffix)
```
Float32 => 0x00 00 00 00
Float64 => 0x00 00 00 00 00 00 00 00
```

### Arrays
An array can be of a fixed size, or a dynamic size.
```
|------- Type ------|---------------- Buffer ----------------|
|--- Name ---|- ID -|---------- Length ---------|--- Data ---|
| Array8  => | 0x00 | 0x00 •••••••••••••••••••• | 0x00..0x00 |
| Array16 => | 0x00 | 0x00 00 ••••••••••••••••• | 0x00..0x00 |
| Array32 => | 0x00 | 0x00 00 00 00 ••••••••••• | 0x00..0x00 |
| Array64 => | 0x00 | 0x00 00 00 00 00 00 00 00 | 0x00..0x00 |

|----------------------- Type -----------------------|-- Buffer --|
|----- Name ------|- ID -|---------- Length ---------|--- Data ---|
| FixedArray8  => | 0x00 | 0x00 •••••••••••••••••••• | 0x00..0x00 |
| FixedArray16 => | 0x00 | 0x00 00 ••••••••••••••••• | 0x00..0x00 |
| FixedArray32 => | 0x00 | 0x00 00 00 00 ••••••••••• | 0x00..0x00 |
| FixedArray64 => | 0x00 | 0x00 00 00 00 00 00 00 00 | 0x00..0x00 |
```

### Maps

A map is an array

### Enums

### Optional Values

### Default Values

### Bounded Values

### Coda Schema

A coda schema is itself defined with a coda schema; it is just an array of Properties, which are a string identifier paired with a Type enum. That means that all schemas have the same associated type.

```
schema CodaType => {
    type: Byte,
    external: Optional<Byte[32]>,
}
schema CodaProperty => {
    identifier: UTF8String,
    type: CodaType,
}
schema CodaSchema => CodaProperty[]
```