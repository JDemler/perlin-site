+++
content_type = "Blog"
date = "2016-10-13T10:43:11+02:00"
title = "Indexing: Memory Access"
+++
[Last blog post](/post/simplify_indexing/) dealt with benchmarking indexing speed and improving it.
Since then indexing was further improved by about 2x and I learned about what is limiting the indexing speed.

This post will explain you to the improved indexing algorithm and then walk through the process of discovering the limiting factor (spoiler alert: It's memory access).

## Improved Indexing

Up until now, indexing in perlin was single threaded and worked basically like this:

```
for every document
    for every term
        assign term_id to term
        add document_id and term_position to the inverted index
```

I decided that splitting up this task was possible and would result in faster indexing speed.
Ideally I wanted it to not share data-structures so it could be non-locking.
So the resulting algorithm works like this:

```
task_1: Assigning term_ids to term
for every document
    for every term
        assign term_id to term
        store (term_id, document_id, term_position) in buffer
    send buffer to task 2
    
task_2: sort and group triplets
for every chunk
    sort chunk by term_id
    group chunk by term_id
    send sorted and grouped chunk to task_3
    
task_3: Insert the resulting listings into the inverted index
for every sorted and grouped chunk
    append or push (document_id, <term_positions>) to term_id entry in inverted index
```

No data-structures are shared; Threads communicate using [mpsc::channel](https://doc.rust-lang.org/std/sync/mpsc/fn.channel.html);
In my first attempt task 2 and 3 where not yet split up and took clearly longer than task 1.
After the split up task 1 was limiting; it currently uses `BTreeMap` to store the vocabulary and assign term_ids to terms. \
Nevertheless, this algorithm is about two times faster for large collections (~40-50Mb/s vs. ~25Mb/s). If you are interested in measuring yourself please have a look at the [perlin-testbench](https://github.com/JDemler/perlin-testbench) repository.

## Digging deeper

Surely better data-structures than `BTreeMap` can be found to solve task 1s problem. First tests show that `HashMap` is faster. But what is the next limiting factor?
Let's have a look at what perf has to say:
```
perf stat ./testbench ~/test.bin

DONE! Indexed 100000 documents each 250 terms totalling at  200Mb in 3477ms
At a rate of 57Mb/s
100000

 Performance counter stats for './testbench /home/jdemler/test.bin':

      11593.721335      task-clock:u (msec)       #    1.935 CPUs utilized          
                 0      context-switches:u        #    0.000 K/sec                  
                 0      cpu-migrations:u          #    0.000 K/sec                  
            20,901      page-faults:u             #    0.002 M/sec                  
    29,994,414,061      cycles:u                  #    2.587 GHz                    
    16,401,507,550      stalled-cycles-frontend:u #   54.68% frontend cycles idle   
     9,220,804,434      stalled-cycles-backend:u  #   30.74% backend cycles idle    
    36,189,798,943      instructions:u            #    1.21  insn per cycle         
                                                  #    0.45  stalled cycles per insn
     6,769,146,655      branches:u                #  583.863 M/sec                  
       225,139,782      branch-misses:u           #    3.33% of all branches        

       5.991013316 seconds time elapsed
```

That looks bad. ~50% frontend and ~30% backend cycles idle. What does that mean? [This SO-Answer](https://stackoverflow.com/questions/22165299/what-are-stalled-cycles-frontend-and-stalled-cycles-backend-in-perf-stat-resul) tells us that it might be related to cache misses and memory access. 
Might be:

```
perf stat -e L1-icache-loads,L1-icache-load-misses,L1-dcache-loads,
             L1-dcache-load-misses,LLC-loads,LLC-load-misses ./testbench ~/test.bin

DONE! Indexed 100000 documents each 250 terms totalling at  200Mb in 3580ms
At a rate of 55Mb/s
100000

 Performance counter stats for './testbench /home/jdemler/test.bin':

    14,758,826,574      L1-icache-loads:u                                             (66.70%)
        75,199,038      L1-icache-load-misses:u                                       (66.34%)
     7,729,428,780      L1-dcache-loads:u                                             (66.53%)
       396,842,316      L1-dcache-load-misses:u   #    5.13% of all L1-dcache hits    (66.69%)
       150,157,164      LLC-loads:u                                                   (66.79%)
        56,172,416      LLC-load-misses:u         #    0.75% of all LL-cache hits     (66.99%)

       7.347828777 seconds time elapsed
```
(L1-icache and -dcache are L1 instruction and data cache, LLC is the last-level-cache (in my case L3). Misses in the last-level-cache result in direct memory access)

Instruction cache seems to be OK. Below 1% L1-icache misses.
But L1-dcache and LLC are bad. According to [latency numbers every programmer should know](https://gist.github.com/jboner/2841832) 56 million LLC-load-misses are equal to [5.6 seconds](http://www.wolframalpha.com/input/?i=1*10%5E-7+*+56*10%5E6) of waiting. Please note, that deallocation of the index is measured, too. Measuring only the indexing part results in a waiting time for memory of about 2.5 seconds. That's about seventy percent of our indexing time!
I'm interested in the details now. By sorting the chunks by `term_id` before indexing them, I thought, the access pattern to memory would be much better.

But let me explain the details first:

``` rust
// This code can be found in perlin::index::boolean_index::mod.rs:353
// Task 3:
// receives sorted listings. merges them into the complete inverted index 
// `Listing` is an alias for Vec<(u64, Vec<u32>)> or in English: 
// A list of document_ids with the positions the term occurred in these documents
fn invert_index(grouped_chunks: mpsc::Receiver<Vec<(u64, Listing)>>) -> Result<Vec<Listing>> {
    let mut inv_index: Vec<Listing> = Vec::with_capacity(8192);
    while let Ok(chunk) = grouped_chunks.recv() {
    // Threshold determines at what term_id the terms are new.
        let threshold = inv_index.len();
        for (term_id, mut listing) in chunk {
            let uterm_id = term_id as usize;
            if uterm_id < threshold {
                // term_id is already known. Append listing
                inv_index[uterm_id].append(&mut listing);
            } else {
                // term_id is new. Push listing
                inv_index.push(listing);
            }
        }
    }
    Ok(inv_index)
}
```

The memory layout of the resulting `inv_index` is the following:
```
 inv_index: Vec<Vec<(u64, Vec<u32>)>>

 +--------------------------------+
 |   data   |  length  | capacity |
 +--------------------------------+
       |
       |  Listings = Vec<(u64, Vec<u32>)>
       |
 +-----v-----------------------------------------------------------+
 |   data   |  length  | capacity |   data   |  length  | capacity | ...
 +-----------------------------------------------------------------+
       |                               |
       |                               |
       |  (u64, Vec<u32>)              +-------------------------------------------------------+
       |  = Listing for one term                                                               |
 +-----v---------------------------------------------------------------------------------+     |
 |  doc_id  |   data   |  length  | capacity |  doc_id  |   data   |  length  | capacity |     |
 +---------------------------------------------------------------------------------------+     |
                                                                                               |
                                                                                               |
       +---------------------------------------------------------------------------------------+
       |  = Listing for another term
 +-----v---------------------------------------------------------------------------------+
 |  doc_id  |   data   |  length  | capacity |  doc_id  |   data   |  length  | capacity |
 +---------------------------------------------------------------------------------------+
```

In my mind, the postings where close together in memory; and thus, when postings are changed which belong to adjacent term_ids they are close together in memory. Really?
Let's have a look!

## Tracking Memory Access

My idea was to log memory access by logging the memory locations of the postings. 
``` rust
    println!("{:?}:{}|{}|{}", inv_index[uterm_id].as_ptr(), uterm_id, listing_len, new);
```

This line prints the memory location of the listing, the term_id, the length of the appended listing and if the term_id was known before or not:

```
0x7fd6edc0c020:0|1|true
0x7fd6edc0c040:1|1|true
0x7fd6edc0c060:2|1|true
0x7fd6edc0c0a0:3|1|true
0x7fd6edc0c0e0:4|1|true
0x7fd6edc0c100:5|1|true
0x7fd6edc0c120:6|1|true
0x7fd6edc0c140:7|1|true
0x7fd6edc0c160:8|1|true
0x7fd6edc0c180:9|1|true
0x7fd6edc0c1a0:10|1|true
```

Not really helpful. What I am really interested in is the difference in memory location between each access.
Let's parse it:
``` rust
use std::io::BufRead;

fn main() {
    let stdin = std::io::stdin();
    let stdinlock = stdin.lock();
    let mut last = 0;
    for line in stdinlock.lines() {        
        let line = line.unwrap();
        let z = u64::from_str_radix(&line[2..14], 16).unwrap();
        if last == 0 {
            last = z;
        } 
        println!("{:?}{}", z as i64 - last as i64, &line[14..]);
        last = z;
    }    
}
```

Now it looks like this:
```
0:0|1|true
32:1|1|true
32:2|1|true
64:3|1|true
64:4|1|true
32:5|1|true
32:6|1|true
32:7|1|true
32:8|1|true
32:9|1|true
32:10|1|true
```
Cool, exactly what I wanted. \
(Maybe someone wants to build a cool visualisation tool for this? :D)\
Listings are close to each other. Let's try it with more data.
It starts similar and everything is fine the first few chunks. Then fireworks start:
```
-3983328:1|1|false
100352:2|1|false
...
191488:382|1|false
-269312:385|1|false
-5120:395|1|false
312320:404|1|false
156672:410|1|false
-160768:447|1|false
-307200:493|1|false
312320:498|1|false
```
This is very bad. Random Access! These differences in memory location are beyond any cache-line or page.
This has to be the problem why cycles are stalled and why cache-loads are missed.
Why does this happen? \
Listings grow during the indexing process. And thus they need to be reallocated. Adjacent space is already occupied so it is allocated somewhere else.\
New data is still allocated close to each other, though:
```
32:6222|1|true
32:6223|1|true
32:6224|1|true
32:6225|1|true
32:6226|1|true
32:6227|1|true
32:6228|1|true
32:6229|1|true
32:6230|1|true
```

## Conclusion
Memory becomes very fragmented when it needs to be reallocated. This slows the indexing process.\
How can we improve on that?
We cannot estimate the length of listings, some are small some are large, we don't even know how large the collection is going to be at that moment. Thus, preallocating listings seems to go in the wrong direction.
Also, linked-lists would remove the need for reallocation but would even introduce more pointers to random memory locations.

Nevertheless, there is still a huge potential for improved indexing speed.
I have two ideas though that I will be examining in more detail the next weeks:

1. Write less data to memory by compressing listings as already done when writing the index to disk
2. Implement a data-structure that supports chunked allocations to minimize reallocations
