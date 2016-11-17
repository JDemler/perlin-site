+++
content_type = "Blog"
date = "2016-11-17T12:50:39+01:00"
title = "Don't Decode Everything"
+++

Perlin is a library with the aim to provide information retrieval functionality in a performant and understandable manner. \
This blog post describes how query execution speed was improved after changes to the underlying data structures.

----

## Recap 

As discussed in the [last blog post](/post/index-50percent-faster/), to allow faster indexing, the data structures that store postings had to change. \
Listings are not stored anymore in continuous space of memory, but rather in chunks with static sizes.
This allows for much faster writing to the listings as they do not have to be moved inside memory. If a listing grows larger than the size chunk, a new chunk is allocated. \

Additionally, we distinguish between hot chunks (chunks that represent the end of a listing and will be written to) and archived chunk (chunks that represent anything but the end of a listing). This enables continuous memory access during the indexing process.


The following chart illustrates the idea. Term 0 occurs often in this collection. It needs five chunks (`HotIndexingChunk` #0, `IndexingChunk` #0, #1, #6 and #7) to store its listing. 
The chunks hold delta- and vbyte encoded postings in ascending order (`IndexingChunk` #0 >= `IndexingChunk` #1 ... >= `HotIndexingChunk` #0):

Note, that indexing chunks for this particular term_id are not adjacent to each other in memory.
But, more importantly, `HotIndexingChunk`s, the ones that are frequently written to during the indexing process are close together in memory and will often be accessed in predictable manner.

```
                                                Archive

 +--------------------------+              +------------------+
 |   HotIndexingChunk       +--------------> IndexingChunk #0 |
 |   for term_id 0          |       |      +------------------+
 +--------------------------+       +------> IndexingChunk #1 |
 |   HotIndexingChunk       |       |      +------------------+
 |   for term_id 1          +--------------> IndexingChunk #2 |
 +--------------------------+       |      +------------------+
 |   HotIndexingChunk       +--------------> IndexingChunk #3 |
 |   for term_id 2          |       |      +------------------+
 +--------------------------+       | +----> IndexingChunk #4 |
 |   HotIndexingChunk       +---------+    +------------------+
 |   for term_id 3          |       | +----> IndexingChunk #5 |
 +--------------------------+       |      +------------------+
                                    +------> IndexingChunk #6 |
                                    |      +------------------+
                                    +------> IndexingChunk #7 |
                                           +------------------+

```



## Query Execution Performance
Comparing this data structure to the old implementation (just storing the postings in vectors (see [Indexing Memory Access](post/indexing-memory-access/)), this is far more complicated and takes time to decode. \
Unsurprisingly, query execution performance has thus dropped quite a bit.\
The first implementation after the rebuild of the indexing process just decoded the whole listing for every query term eagerly. This was far from optimal, especially because perlin allows lazy query execution.

### Lazy Decoding
The first step for improvement was obvious. Instead of decoding the whole listing at once and storing it in a `Vec`, just hand the query execution machinery an `Iterator` for every query term that decodes the postings when necessary.

This iterator holds a reference to the corresponding `HotIndexingChunk` as well as one to the archive. While advancing the iterator, it reads as many bytes as needed to decode the next posting from an archived chunk or the `HotIndexingChunk`.

Changing the decoding from eager to lazy improved query execution performance for queries that are only interested in a small portion of results.

The question now arises, if we can also improve query execution performance for the general case.

### Don't look at everything

When asked why grep is so fast, Mike Haertel responded with a [lengthy email](https://lists.freebsd.org/pipermail/freebsd-current/2010-August/019310.html). His key idea in this email is that "to make[..] programs fast is to make them practically do nothing."

Can we apply this idea to our case?

Let's assume following query:
```
("for" AND "science")
```

"for" is a common term. It's listing will be spread over several `IndexingChunk`s. "science" on the other hand, will probably occur at least a magnitude less often than "for".

Consider following collection:
```
doc_0: "For a long time, people have been studying the stars"
doc_1: "The search for life on Mars is ongoing"
doc_2: "Jupiter can be seen with the naked eye, for it is the largest of the planets"
doc_3: "Venus seems uninhabitable for human beings"
doc_4: "He exclaimed For Science! while lithobraking on Pluto"
```
For this very small collection, the listings for "for" and "science" look like this (positions left out for simplicity):

```
              chunk 1  | chunk 2
           +---+---+-------+---+
           |   |   |   |   |   |
  "for"    | 0 | 1 | 2 | 3 | 4 |
           |   |   |   |   |   |
           +---+---+---+---+---+

           +---+
           |   |
"science"  | 4 |
           |   |
           +---+

```

"for" occurs in document 0, 1, 2, 3 and 4 while "science" only occurs in document 4.

Assume now further, that the listing of "for" is spread over two chunks.

When executing the query, the old way to do this is to look at every posting of every query term.
In this case the process would executed as follows (pseudo-code for clarity):

``` rust
let focus = "science".next()
loop {
    if focus == "for".next() {
        return focus;
    }
}
```

We call the `next()` method six times. Once for every stored posting.

If additionally to chunk ids HotIndexingChunk could store the doc_ids they start with, we could allow seeking access to the potentially interesting chunks and ignore irrelevant ones.

The process would be simpler, too:

``` rust
let focus = "science".next();
if "for".next_seek(&focus) == focus {
    return focus;
}
```

In total we only need to call next three times: \
`"science".next()`\
`"for".next_seek()` wich will call `next()` internally two times (first time yielding 3 and second time yielding 4)

Let's try that.


## Implementation Details

`HotIndexingChunk` now needs new capabilities. It needs a method that takes a DocId and it returns the relevant chunk, and the byte offset where the first posting of this chunk is encoded. This is needed because encoded positions can overflow a chunk. \
Additionally it must give us the DocId of that first encoded posting, because postings are delta encoded and otherwise decoding would not be possible.

```rust 
    pub fn doc_id_offset(&self, doc_id: &u64) -> (u64/*doc_id*/, usize/*byte_offset*/) {
```

For that to be possible, `HotIndexingChunk` needs to know a bit more about its archived chunks:

```rust
    //Before:
    archived_chunks: Vec<u32/*chunk_id*/>
    //Now:
    archived_chunks: Vec<(u64/*doc_id*/, u16/*offset*/, u32/*chunk_id*/)>
```

Also, we need capabilities to seek to a certain byte position while reading from a HotIndexingChunk:
```rust
pub struct ChunkRef<'a> {
    read_ptr: usize,
    chunk: &'a HotIndexingChunk,
    archive: &'a Box<Storage<IndexingChunk>>,
}

impl<'a> io::Seek for ChunkRef<'a> {
    fn seek(&mut self, style: io::SeekFrom) -> io::Result<u64> {
        use std::io::SeekFrom;
       let pos = match style {
            SeekFrom::Start(n) => { self.read_ptr = n as usize; return Ok(n) }
            SeekFrom::End(n) => self.bytes_len() as i64 + n,
            SeekFrom::Current(n) => self.read_ptr as i64 + n,
        };

        if pos < 0 {
            Err(io::Error::new(io::ErrorKind::InvalidInput,
                           "invalid seek to a negative position"))
        } else {
            self.read_ptr = pos as usize;
            Ok(self.read_ptr as u64)
        }
    }   
}
```

We can now determine the byte-offset where it is sensible to look for a DocId and can seek to it.
Now we need a way to express this throughout the query execution process:

```rust
pub trait SeekingIterator {
    type Item;

    /// Yields an Item that is >= the passed argument or None if no such element exists
    fn next_seek(&mut self, &Self::Item) -> Option<Self::Item>;
}
```

And implement it for `PostingDecoder`:

```rust
fn next_seek(&mut self, other: &Self::Item) -> Option<Self::Item> {
    // Check if the iterator is already too far advanced.        
    if self.last_doc_id >= *other.doc_id() {
        return self.next();
    }
    
    // Get the doc_id and offset for the next sensible searching position
    let (doc_id, offset) = self.decoder.underlying().doc_id_offset(other.doc_id());
    // Seek to the offset
    self.decoder.seek(SeekFrom::Start(offset as u64)).unwrap();
    // Decode the next posting
    let mut v = try_option!(self.next());
    // DocId is corrupt, because delta encoding is obviously not compatible with seeking
    // So overwrite it with the doc_id given to us
    v.0 = doc_id;
    
    // Store it for further decoding
    self.last_doc_id = doc_id;
    // If this seek already yielded the relevant result, return it
    if v >= *other {
        return Some(v);
    }
    // Otherwise continue to decode 
    loop {
        let v = try_option!(self.next());
        if v >= *other {
            return Some(v);
        }
    }
}
```


## Conclusion
This measure improves query execution performance especially for similar cases to the one showed above: ("seldom_term" AND "frequent_term").

We now not only have capabilities to jump to certain postings, which is a huge help for implementing operators, but also do not access some `IndexingChunk`s in certain situations. \
This will result in far better performance when `IndexingChunk`s are not in memory but on disk or stored somewhere on the network.

The next step will be to lazily decode positions. Positions are needed only for positional queries and then only if document ids match. So our current method, to decode them every time we look at a DocId is wrong.

These changes will be available in version 0.2 which will hopefully be released by the end of this year.
