Proposal for better OHC implementation:

- same API as 'linked' implementation
- but use tables instead of links

'linked' implementation:

OHCacheImpl is non-synchronized and contains several segments implemented by OffHeapMap, which has synchronized methods.
OffHeapMap contains a simple table containing one pointer to the first hash entry of a bucket - this keeps the
table pretty dense.
Each hash entry lives in its own memory region and consists of the header, the key and the value.
Key and value are just the serialized representations.
Header consists of 7 long values:
- pointer to next entry in LRU list (over whole OffHeapMap)
- pointer to previous entry in LRU list (over whole OffHeapMap)
- pointer to next entry in bucket
- reference counter (initially 1, incremented on each 'live' access, decremented when no longer used, if decremented to 0, memory can be freed)
- 64 bit hash
- value length
- key length
Additionally each OffHeapMap contains two longs to the head and the tail of the LRU list.

Problems occur especially on NUMA machines insisting in high to very high system CPU usage. It is not caused
by synchronization or locks (which can achieve rates of 1M per second and much more). It is caused by letting multiple
CPUs (in multiple sockets) access the structure.

Each read has to:
- Access the OffHeapMap table
- Access the first hash entry (in the bucket)
- Compare the hash and key length against the key to match
- Compare the key value (if hash and key length match)
- Traverse via 'pointer to next entry' to the next entry in the bucket (and restart with hash + key length comparison)
- Look up LRU prev and LRU next of its own and its LRU predecessor and antecessor
- Modify LRU prev, LRU next of its own and its LRU predecessor and antecessor (or LRU head and LRU tail in OffHeapMap)

Each write has to:
- Access the OffHeapMap table
- Store the first hash entry in its 'next entry' pointer
- Modify the OffHeapMap table
- Set the right LRU next value in its header to OffHeapMap's LRU head
- Eventually set OffHeapMap's LRU tail (if it's the first entry in the table)
- Modify the LRU prev value of the previous LRU head

Both reads and writes have one bad characteristic: they have to a lot of different memory locations that force the
CPU to load and evict its cache contents. This is a bad behaviour since many distinct regions in whole RAM need to
be read and written.

'table' implementation:

Assumption: Stress tests showed that each hash bucket has at most 8 entries (maybe nine in very rare cases).

1st change: Blow up per-OffHeapMap table.
The table shall include pointers and the hashes of _all_ entries of the whole OffHeapMap - i.e.
eliminate the 'pointer to next entry' in 'linked' implementation.
Size estimation on 32 core system:
- 'linked' implementation : 4MB (= 8192 * 8 * 64 with default configuration)
- 'table' implementation  : 32MB (= 8192 * (8 + 8) * 8 * 64 with default configuration)

2nd change: Condense LRU information.
Per-OffHeapMap LRU information in a separate table.
The lru-table is sized to be able to contain 'hash table size * load factor' (at least).
Two index-pointers are used as pointers to the next write-index and the eldest entry.
If "write-index" is outside of the table, the whole structure needs to be compacted - starting with
the eldest-entry-index to the end of the lru-table.
