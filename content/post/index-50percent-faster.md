+++
content_type = "Blog"
date = "2016-10-19T10:03:10+02:00"
title = "Index 50% Faster!"

+++

Perlin is a library with the aim to provide information retrieval functionality in a performant and understandable manner. \
This blog post is part three of a series about improving indexing speed.

----

Last [blog post](/post/indexing-memory-access/) identified memory access patterns as bottleneck for indexing speed. It concluded, that chunking allocations and compressing listings during indexing might result in better memory locality and thus solve the problem of LL-cache misses.

In this post implementation details of these ideas are described and their impact on performance is evaluated. 

## Abstract Idea
Let's start by looking at the adjusted algorithm in a abstract way. Like in the last post there are three basic tasks to be done during indexing:

1. Matching term to term_id
2. Sorting and grouping (term_id, doc_id, position)-triplets
3. And appending the results to the listing of each term_id

This does not change at all. What changes, is the way `(document_id, <term_position>)` is appended to the inverted index. That task is done by a new struct called `IndexingChunk`.

## IndexingChunk
`IndexingChunk` is a element in an [unrolled linked list](https://en.wikipedia.org/wiki/Unrolled_linked_list) of compressed indexing data. Its task is to compress listings and append the resulting bytes to itself. 
When full, it shall report how much of the listing fitted into it.

To remove one level of indirection, the payload of this chunk is not modelled as `Vec` but rather as a fixed size array:

``` rust
pub struct IndexingChunk {
    //Some fields omitted
    previous_chunk: u32,
    capacity: u16, 
    last_doc_id: u64, 
    data: [u8; SIZE], 
}
```

The field 'last_doc_id' stores the `document_id` of the last appended posting and makes it thus possible to [delta-encode](https://en.wikipedia.org/wiki/Delta_encoding)`doc_id`s.
Also, `capacity` stores the remaining capacity of the chunk and `previous_chunk` will be filled with the id of the next chunk once it is created.

Acting as central and single method of `IndexingChunk` is `append` which takes a listing to be appended and returns either `Ok` or the number of postings that fitted into the chunk:

``` rust
pub fn append(&mut self, listing: &[(u64, Vec<u32>)]) -> Result<(), usize>
```

Simplified and translated into pseudo-code, it does the following:

```
for every (doc_id, positions) in listing
    delta and vbyte encode doc_id
    try to write resulting bytes to payload
        for every position in positions
            delta and vbyte encode position
            try to write resulting bytes to payload
     else 
         reset capacity
         return current iteration over listing
```
   

So far so good. These chunks are organised by a struct called 'ChunkedStorage` (this name seems bad and will probably be changed).

## ChunkedStorage
`ChunkedStorage` has the task to organise `IndexingChunks` by keeping hot chunks (the ones which are currently written to) close together and archive full chunks so that they can be accessed later for querying.

In its current state it is not fully functional in the sense that it does not provide functionality for querying. But more on that later.

``` rust 
pub struct ChunkedStorage {
    hot_chunks: Vec<IndexingChunk>, //Has the size of vocabulary
    archived_chunks: Vec<IndexingChunk>, 
}
```

In later iterations, archived_chunks will be replaced by something more general, something that allows writing chunks to different storage types (i.e. RAM, Memory, maybe Network).

`ChunkedStorage` has three relevant methods, they are all straightforward and fun:

``` rust
impl ChunkedStorage {

    pub fn new_chunk(&mut self, id: u64) -> &mut IndexingChunk {
        self.hot_chunks.push(IndexingChunk {
            previous_chunk: 0,
            last_doc_id: 0,
            capacity: SIZE as u16,
            data: unsafe { mem::uninitialized() },
        });
        &mut self.hot_chunks[id as usize]
    }

    pub fn next_chunk(&mut self, id: u64) -> &mut IndexingChunk {
        let next = IndexingChunk {
            previous_chunk: self.archived_chunks.len() as u32,
            last_doc_id: self.hot_chunks[id as usize].last_doc_id,
            capacity: SIZE as u16,
            data: unsafe { mem::uninitialized() },
        };
        //Move the full chunk from the hot to the cold
        //That's more fun than I thought
        self.archived_chunks.push(mem::replace(&mut self.hot_chunks[id as usize], next));
        &mut self.hot_chunks[id as usize]
    }    

    #[inline]
    pub fn get_current_mut(&mut self, id: u64) -> &mut IndexingChunk {
        &mut self.hot_chunks[id as usize]
    }
}
```

By keeping the hot chunks in an extra vector it is ensured, that they are close together in memory which should result in less cache-misses and thus improve performance.

## Adjusted Implementation for Task_3

Task 3 now has to use `ChunkedStorage` and `IndexingChunk` of course. But it all feels good and comes very naturally:
``` rust
let mut storage = ChunkedStorage::new(4000);
while let Ok(chunk) = grouped_chunks.recv() {
    let threshold = storage.len();
    for (term_id, listing) in chunk {
        let uterm_id = term_id as usize;
        // Get chunk to write to or create if unknown
        let result = {
            let stor_chunk = if uterm_id < threshold {
                storage.get_current_mut(term_id)
            } else {
                storage.new_chunk(term_id)
            };
            stor_chunk.append(&listing)
        };
        // Listing did not fit into current chunk completely
        // Get the next and put it in there
        // Repeat until done
        if let Err(mut position) = result {
            loop {
                let next_chunk = storage.next_chunk(term_id);
                if let Err(new_position) = next_chunk.append(&listing[position..]) {
                    position += new_position;
                } else {
                    break;
                }
            }
        }
    }
}
```


## Results
Now. I hope you are as exited as I am. Let's see what we've got.
I am using the same machine and the same test collection as in lasts blog post, so results should be comparable.

```
./testbench ~/test.bin 

DONE! Indexed 100000 documents each 250 terms totalling at  200Mb in 2446ms
At a rate of 81Mb/s
```

That's an improvement of 50% or about 25Mb/s over the last version. Let's see if the cache misses and stalled cycles changed:

```
perf stat ./testbench ~/test.bin 

DONE! Indexed 100000 documents each 250 terms totalling at  200Mb in 2445ms
At a rate of 81Mb/s
100000

 Performance counter stats for './testbench /home/jdemler/test.bin':

       9168.706474      task-clock:u (msec)       #    3.690 CPUs utilized          
                 0      context-switches:u        #    0.000 K/sec                  
                 0      cpu-migrations:u          #    0.000 K/sec                  
           342,090      page-faults:u             #    0.037 M/sec                  
    21,329,469,020      cycles:u                  #    2.326 GHz                    
     9,293,938,235      stalled-cycles-frontend:u #   43.57% frontend cycles idle   
     5,693,540,682      stalled-cycles-backend:u  #   26.69% backend cycles idle    
    37,224,017,304      instructions:u            #    1.75  insn per cycle         
                                                  #    0.25  stalled cycles per insn
     5,635,594,793      branches:u                #  614.655 M/sec                  
       117,443,035      branch-misses:u           #    2.08% of all branches        

       2.484579972 seconds time elapsed
```
That's 10% less frontend cycles idle and +0.5 instructions per cycle.

```
perf stat -e L1-icache-loads,L1-icache-load-misses,L1-dcache-loads,L1-dcache-load-misses,
             LLC-loads,LLC-load-misses ./testbench ~/test.bin

DONE! Indexed 100000 documents each 250 terms totalling at  200Mb in 2389ms
At a rate of 83Mb/s
100000

 Performance counter stats for './testbench /home/jdemler/test.bin':

    12,159,900,932      L1-icache-loads:u                                             (66.56%)
        12,582,764      L1-icache-load-misses:u                                       (66.54%)
     7,979,740,445      L1-dcache-loads:u                                             (66.71%)
       416,592,405      L1-dcache-load-misses:u   #    5.22% of all L1-dcache hits    (66.75%)
        70,240,608      LLC-loads:u                                                   (66.77%)
         9,644,759      LLC-load-misses:u         #    0.16% of all LL-cache hits     (66.77%)

       2.429317333 seconds time elapsed
```
As expected, the LLC-load-misses are down from 56M to 10M. The L1-dcache-misses though stayed about the same.
400M in the old version vs 416M now.


## Conclusion and future Work
The indexing is now 50% faster, but everything else is broken. I prototyped this change into the existing code, leaving nothing but scorched earth. The next week will be spent cleaning this mess up and marrying the query system with the new chunked indexing system.

Nevertheless, L1-cache misses are still high and could probably still be optimised.
But, I think micro-optimisations beyond this point should only be considered when real world applications see a need for it. The current indexing speed should be sufficient for most information retrieval needs in small projects.
I spent more than enough time on that anyway.

<center>
![Messy Git-Tree](/images/git_mess.png)
</center>
