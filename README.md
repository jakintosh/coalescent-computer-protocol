# Coalescent Computer Protocol

A protocol that lets individual machines act as nodes in a coalescent, distributed, global, virtual computer — no blockchain required.

Heavily inspired by holochain.org, ceptr.org, and infocentral.org.

## Overview

At its core, this protocol relies on two core concepts:
  1) immutable, content-addressable data via cryptographic hashes
  2) "code as data"

Essentially: we use the determinism of this protocol to use each node's filesystem as a single universal instance of 256-bit content addressable RAM. To the "virtual machine" (which only exists conceptually), there is no name/location addressable storage, only a single distributed instance of RAM with 10^77 addresses. The protocol is network agnostic; it doesn't care how (or if) you network with other nodes. In practice, you can resolve address lookups using only data local to the node, over local network, over internet via p2p or client-server model, or even through blockchain (though security, validation, and permissions can be handled in far more efficient ways with this protocol, rendering most benefits of blockchain obsolete). On the Coalescent Computer, all data is content addressable, and all code is data.

The compoents of the protocol are:
  1) Data **schema**: structure of data and primitive types of fields; defined using a deterministic binary format
  2) Data **translator**: map one schema onto another; defined using a deterministic binary format
  3) Data **validator**: a validation function for a given schema; defined using a deterministic binary format
  4) Data **blob**: actual binary data; binary data is prefixed with their schema hash

---

## Components

### Schemas

When we define a schema in a deterministic binary format, we can hash those bits and use that hash to refer to it. This eliminates namespace collisions, while also enabling "coalescence" between two agents who have (un)knowingly defined the same schema. Since we refer to the schema via its hash, we get the benefit of immutability (versions/migrations can be handled with **translators**). This also means that any old data using an old hash will never become unreadable, because the data it points to is immutable.

### Translators

When we define a translator in a deterministic binary format, we get all of those same benefits as schemas. Translators can be semantically linked to the schemas they operate on, making it simple for an agent to say "I have data in {this} schema, is there a translator available to map it onto {that} schema?". This also enables coalescence; if multiple schemas are being devloped in parallel, when one (or several) gain major traction, the defunct schemas can be mapped onto the leading standard, or mutiple similar schemas can be mapped between each other to enable interoperability; for example, if a betamax could have been placed into a VHS player, and the VHS player knew how to identify and translate that data automatically.

### Validator

If we define a data validator in a deterministic binary format, again, we get those same content addresability benefits. What a validator enables is any agents to play by certain rules with their data. A schema defines how to read the data, but a validator is a layer on top that defines what the boundaries are for those values. When a group of agents decide to play by certain rules, they can choose those rules using the hash of a validator; they can then all access the validator, and validate any data they interact with to ensure it is valid.

### Blobs

Finally, blobs are the actual raw bits of data. They are prefixed with their schema hash; this means that a data blob is self describing. Given only the data itself, an agent has all the information necessary to understand the data.

---

## Concepts

### Hashes are Memory Addresses in the Global Computer

In the existing model of a computer, the RAM is the part of the computer where data can be stored by an address. Each row of memory is referenced by its index number, so memory addresses start at address 0, then go to address 1, and so on. If you want to access a piece of data, you need to know which memory slot it is in. To programmers, this is the "pointer"; the slot in memory that the data you are looking for resides at. However, this is awkward because it describes a location, not the data you are looking for. Because it's just a location, the data at that address can change over time. This is a problem that has been solved in many ways through language features, software development practices, and more. However, if, instead of using a location for a piece of data (like a URL does), we just used its content hash, we avoid all of this. We now effectively have a virtual memory system with 2^256 possible slots, based on the 256 bit hash/address. We also never have to worry about the wrong data being at that address; only the expected data can live there.
