# Coda Transfer Protocol (CTP)

The `Coda Transfer Protocol` is how two different instances of the coalescent computer negotiate and exchange coalescent data (coda).

## WIP Notes

### Context Declarations

Part of establishing a CTP connection is setting a _context_. The context defines the form of the coda that will be sent over the connection, including any type mappings or assumed types.

### Data Sharding

When requesting data, especially large pieces of data, one option is to shard the data like BitTorrent does. Someone who has a complete file can generate a shard declaration for it by defining the block size, and then hashing checksums of those pieces into a merkle tree. The original coda identifiee, piece size, and piece checksums define the shard declaration, which can then be sent to any CTP node to initiate a shard transfer. The 