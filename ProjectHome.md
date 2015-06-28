# An implementation of Judy Arrays in 1250 Lines of C Code. #

A Judy array consists of a pointer to a tree of nodes.  An empty tree is indicated by a `NULL` pointer.  A Judy object is returned by `judy_open`, and initially contains an empty tree.  Integer or string keys are added to the tree with `judy_cell` which returns a pointer to the mapped `uint` (`long long` in the 64 bit version) cell for each key value added.  This cell must be filled with a non-zero value, usually a line number, or a count of insertions for duplicate key tracking, or a pointer to a data area to be associated with the key, prior to calling a subsequent tree operation.

A demonstration Penny Sort is included that uses Judy Arrays for both sorting and merging, as is a variable key length string sorter.

## Judy Tries ##

The Judy tree consists of cascaded levels which each break down the next 4 (or 8) bytes of key values. There are two types of Judy nodes in the tree:  radix nodes, and six sizes of linear array nodes.  The bottom 3 bits of a Judy tree pointer specify the type and size of the node, and the position in the tree determines the element size in the node.

Linear array nodes contain from 1 to 51 keys with associated tree pointers, depending on the node's overall size, and the size of the keys stored in that node.  The linear array node breaks out from 1 to 4 bytes (or 1 to 8 bytes in the 64 bit version) of key value depending on how many radix node sets preceed it at the tree level it occupies.  When a linear array node fills up, it is promoted to the next larger linear array node by doubling its current size.  When the largest size node fills up, its high order key bytes are broken down into radix node sets with a new set of linear array nodes containing the original keys, each minus the byte assigned to its radix node slot.  If there are any key bytes left after the radix nodes at each level, a linear array node occurs that completes the remaining key bytes for that tree level.

The radix node sets come in pairs that encode inner and outer nibbles of the next key byte.  The outer node contains 16 tree pointers to inner radix nodes which in turn contain 16 tree pointers to each underlying linear array node, or cascaded radix node, or a leaf cell.  Each radix node pair breaks out 1 byte of key value making each underlying linear array node at that level one key byte smaller.  It is possible for a tree level to be occupied entirely by 4 sets of cascaded radix node pairs, or any combination of leading radix nodes with their subsequent linear array nodes.

## String or Integer Keys ##

The Judy Array supports either string or integer keys.  String keys are passed as pointers to arrays of bytes along with the string length, while integer keys are passed as arrays of native integers (32 or 64 bits) and the depth of the array is passed when the Judy array is created.  The address of the high-order integer key pointer is passed to the judy functions as a character pointer and the string length argument should be set to the `depth * JUDY_key_size`.

## Comparison with Other Sorted Access Methods ##

The `judy64n` version is included that processes the benchmark developed by Dr. Askitis, the distinct\_1 dataset, in 18 seconds on a 64 bit linux system. This is comparable with the HAT trie as a sorted collection method.  The distinct\_1 and skew1\_1 datasets are available at http://www.naskitis.com.  Compile `judy64n` with `-D ASKITIS`, and run with distinct\_1 as the parameter for the benchmark.

For a comparison with my implementation of the HAT trie, please see the hat-trie project page: http://code.google.com/p/hat-trie.  For sorting strings, the HAT trie code is 33% faster than `judy64n`, and the code size is 20% smaller.

## Node Layouts ##

A linear array node ranges in size from 8 to 256 bytes, in 6 powers of two.  From 1 to 4 bytes of key for the tree level are stored in slots up from the bottom of the node address space, and the corresponding tree pointers are stored in `uint` slots down from the end of the node.  The highest numbered slots are used first. For string key values ending with a zero byte, a leaf slot with the corresponding address of the `uint` cell is returned to the caller.  For integer key values, a leaf slot is returned when the depth of the tree is reached. Otherwise, the slot contains a Judy tree pointer to the next level of the tree where the search continues.

Outer and inner radix nodes are 64 bytes each and paired into a 16x16 array of 256 tree pointer slots.  For string keys, slot 0x0 is always a cell for a tree leaf for the key that ends at that radix node slot.  For Integer keys, a tree leaf cell occurs at the depth of the tree.

## Judy Path Stack ##

A Judy object includes space for a path stack down the tree to the most recent tree leaf referenced.  Since there are 4 bytes of key handled at each tree level by radix and linear array nodes, the minimum size for this stack is the maximum key size / 4.  To allow for up to 4 radix nodes at each level, the theoretical stack size is 4 times larger than the minimum size.

The path stack is set by `judy_slot` and used by `judy_key` to reassemble the tree leaf's key value, and by `judy_nxt` and `judy_prv` to iterate to the next or previous key in the tree.  The path stack also identifies a particular key to be deleted by `judy_del`.

## Memory Allocation ##

Because judy tree pointers must be aligned on 8 byte multiples to leave the bottom 3 bits to indicate the node type, a virtual memory allocator is provided which requests memory blocks (normally 65536 bytes in length) from the underlying OS and parcels them out into new nodes as needed.  Under WIN32, these blocks are guaranteed to reside on 64K boundaries, and under linux they reside on 4K boundaries.  An externally callable allocator `judy_data` will also return space from these memory blocks, which will be deleted when the Judy object is closed by `judy_close`.  Note that memory cannot be allocated from cloned judy trees.

## 64 bit version ##

The judy64 downloads will compile to either a 32 bit or 64 bit program depending on the compilation environment. In 64 bit mode the `uint` judy cells are promoted to 64 bit `long long` values. To accommodate the larger 64 bit keys, cells and tree pointers, the linear array node sizes have been doubled to 16, 32, 64, 128, 256, and 512 bytes.  Each tree level encodes 8 bytes of its keys.

## Judy3 enhancement for string keys ##

Judy3 is an extension to the judy2 code which is also contained in the judy64 code.  A third node type is added to the trie which stores trailing string key bytes contiguously in 28 byte `JUDY_span` arrays, instead of being broken down into 4 byte `JUDY_8` linear array nodes.  This expands the key byte coverage of a level of the tree from 4 to as many as 28 bytes.  If another node is inserted into the trie which needs to land in a span node, the span node is first split up into as many as 7 `JUDY_8` linear array nodes and the insert operation proceeds into those linear array nodes as in judy2.  This enhancement improves performance in both space and time for most input files.  A file of 10,000,000 32 byte random hex keys sorts in 8 seconds by judy3 vs. 14 seconds for linux sort (with LANG=C set) and 26 seconds for judy2.

## Concurrent Judy Array Access ##

Usage of the Judy Array will need to be synchronized between threads.  A Judy object will need to have a semaphore allocated, and additional calls made to acquire and release access to the Judy array.  If all access to the Judy array becomes read-only in nature after building, concurrent access can be supported by cloning the Judy object with `judy_clone` for use by each additional thread.  Note that the cloned copy will be deleted when `judy_close` is called for its parent, and further additions to the Judy array are not supported under the cloned copy.

## Demonstration Penny Sort ##

Judy64j.c includes a memory mapped string sorter designed to process large pennysort files with a sort/merge approach. Judy Arrays are used for both sorting and merging.  Initial runs of 819200 records are sorted in memory and then written into temporary files which are then merged together to produce the final sort output. Usage: `judy64j infile outfile 10` to specify the 10 byte keys for the pennysort ascii file.  It also illustrates usage of judy cells to contain structure pointers.  The demonstration program sorts a 5GB penny sort file in 160 seconds, compared to 290 seconds for linux sort (with LANG=C) on a 64 bit linux 2.6.32 system.

A standard string sorter demonstration with variable length records is invoked by `judy64j infile outfile`.

## Judy Functions ##

### Open Array ###

`void *judy_open (uint levels, uint depth)`

Allocate and return a new judy object pointer with an empty judy array, and with internal stack space for `levels` of tree to be used for `judy_nxt, judy_prv, judy_key`.  This object pointer is passed to the subsequent functions as a `Judy *`.  The depth argument is set to zero for string keys, otherwise for integer keys it is set to the depth of the tree in Integers (32 or 64 bit).

### Clone Array ###

`void *judy_clone (Judy *judy)`

Clone a copy of a judy object for use by an independent thread for read access to the Judy array.  Each thread needs an independent internal Judy stack.

### Inserting Keys ###

`uint *judy_cell (Judy *judy, uchar *buff, uint len)`

Insert a new key (or find an existing one) and return a pointer to its `uint` cell value.  This cell must be filled in with a non-zero value by the caller prior to the next judy function call.  For integer keys, the `buff` argument is replaced by a pointer to an array of native integers, and the len argument is ignored.

### Start Iterator ###

`uint *judy_strt (Judy *judy, uchar *buff, uint len)`

Find the first key greater than or equal to the given key and return the cell address.  The internal Judy stack is set to identify the given key.

### Next/Previous Iterators ###

```
uint *judy_prv (Judy *judy)
uint *judy_nxt (Judy *judy)
```

Iterate to the next or previous key in the tree depending on the state of the internal Judy stack.  Set the stack to the new entry location.

### Lookup Key Value ###

`uint *judy_slot (Judy *judy, uchar *buff, uint len)`

Find the cell associated with the given key and return its address, or return `NULL` if the key is not in the Judy tree.  Set the internal Judy stack to the key cell entry returned.

### Assemble Key Value ###

`uint judy_key (Judy *judy, uchar *buff, uint max)`

Using the internal Judy stack, construct the key indicated by the stacked tree levels into the buffer provided, returning its string length for string keys, and returning the tree depth for Integer keys.  Note that `max` must be set to the size of the integer array for integer keys.

### Delete Key Value ###

`uint *judy_del (Judy *judy)`

Delete the key value identified by the current Judy stack contents.  The previous judy cell pointer is returned.

### Allocate Memory ###

`void *judy_data (Judy *judy, uint amt)`

Allocate memory from within the Judy object for optional external use.  The resulting zeroed memory will be located on an 8 byte memory boundary, and will be freed when `judy_close` is called.  Note that `amt` must be less than or equal to `(JUDY_seg - 8)` bytes (normally 65528).

### Judy Close ###

`void judy_close (Judy *judy)`

Free all allocated memory used by the Judy array and the Judy object and any cloned copies of the Judy object.

## Author Contact Information ##

Please address any problems found or questions to the program author,
Karl Malbrain: malbrain-at-yahoo-dot-com.