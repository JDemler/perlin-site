+++
content_type = "Blog"
date = "2016-10-04T10:54:49+02:00"
title = "Simplify Indexing"
+++

In the release notes of [Perlin v0.1](/news/release-v0.1/) we state that "indexing is incredibly slow".\
Having worked on this for the last week I have to rephrase that statement into "indexing benchmarks are incredibly wrong".

This post tells the story of how indexing benchmarks were flawed, how they were fixed and finally how I got something out of it and improved indexing speed.

## The Benchmark [v0.1]
The idea of the indexing benchmarks are straightforward: create random documents with random terms with term count and term distribution being similar to the ones in natural language. In other words: the number of distinct terms is given by [Heaps' Law](https://en.wikipedia.org/wiki/Heaps%27_law) and the document terms will be randomly distributed using the [Zipf's Law ](https://en.wikipedia.org/wiki/Zipf%27s_law). \
For more information why these two laws apply read [chapter 5.1](http://nlp.stanford.edu/IR-book/pdf/irbookonlinereading.pdf#section.5.1) of the IR-Book.

**Let's see some code!**

Implementation of Heaps' Law:
``` rust
pub fn voc_size(k: usize, b: f64, tokens: usize) -> usize {
    ((k as f64) * (tokens as f64).powf(b)) as usize
}
```

Zipf's Law is implemented by a generating iterator:
``` rust
#[derive(Clone)]
struct ZipfGenerator {
    voc_size: usize,
    factor: f64,
    acc_probs: Box<Vec<f64>>,
}

impl ZipfGenerator {
    fn new(voc_size: usize) -> Self {
        let mut res = ZipfGenerator {
            voc_size: voc_size,
            factor: (1.78 * voc_size as f64).ln(),
            acc_probs: Box::new(Vec::with_capacity(voc_size)),
        };
        let mut acc = 0.0;
        for i in 1..voc_size {
            acc += 1.0 / (i as f64 * res.factor);
            res.acc_probs.push(acc);
        }
        res
    }
}

impl<'a> Iterator for &'a ZipfGenerator {
    type Item = usize;

    fn next(&mut self) -> Option<Self::Item> {
        let dice = rand::random::<f64>();
        let mut c = 0;
        loop {
            if dice < self.acc_probs[c] {
                return Some(c);
            }
            c += 1;
        }
    }
}

```


Probably you already noticed the problem. But lets see how these were used to benchmark indexing speed:

``` rust
#[bench]
fn index_100kb(b: &mut Bencher) {
    let rng = ZipfGenerator::new(voc_size(45, 0.5, 12500));
    b.iter(|| {
        let count = test::black_box(100);
        IndexBuilder::<_, RamStorage<_>>::new().create((0..count).map(|_| rng.take(125)));
    });
}
```

A new generator is created and for every document it generates 125 terms.
The generator is not fast: it is not optimized and uses cryptographically safe randomness which is absolutely unneccersarry for this task. This kills the results of the benchmark: as the generator is lazy, the generation of the terms happen during the indexing and the generator is about an order of magnitude slower than indexing. 
From the results of this benchmark I concluded "indexing is incredibly slow".

## Fixed Benchmark

The solution to this problem is simple: generate the documents before the indexing and do it only once, not every iteration:

``` rust
lazy_static!{
    pub static ref COLLECTIONS: [Vec<Vec<usize>>; 4] = [
        (0..100).map(|_| ZipfGenerator::new(voc_size(45, 0.5, 12500)).take(125).collect()).collect(),//100kb
        (0..1000).map(|_| ZipfGenerator::new(voc_size(45, 0.5, 125000)).take(125).collect()).collect(),//1MB
        (0..10000).map(|_| ZipfGenerator::new(voc_size(45, 0.5, 1_250_000)).take(125).collect()).collect(),//10MB
        (0..100000).map(|_| ZipfGenerator::new(voc_size(45, 0.5, 1_500_000)).take(125).collect()).collect(),//100MB
    ];
}

#[bench]
fn index_100mb(b: &mut Bencher) {
    b.iter(|| {
        test::black_box(&IndexBuilder::<_, RamStorage<_>>::new().create(COLLECTIONS[3].iter().map(|i| i.iter())));
    });
}
```

## Improve Indexing
Now, with sound and realistic indexing benchmarks, I noticed. Indexing is not slow. Perlin 0.1 indexes at least 12.5 Mb/s:
```
running 4 tests
test index_100kb ... bench:   3,767,254 ns/iter (+/- 178,005)
test index_100mb ... bench: 7,770,690,219 ns/iter (+/- 652,561,624)
test index_10mb  ... bench: 665,334,106 ns/iter (+/- 103,991,620)
test index_1mb   ... bench:  42,403,164 ns/iter (+/- 2,008,641)
```

My previous attempts to parallelize indexing yielded similar results and seemed a bit pointless after discovering that the benchmarks were flawed.
Nevertheless, my continous work on the indexing process grew an idea of how indexing could be simplified.
Perlin 0.1 indexes into two very basic containers:
A dictionary that maps a term to a term id and an inverse index, which contains the mapping between term id and document id. These were both implemented as `BTreeMap`s:
```rust
//Term to term ID
BTreeMap<TTerm, u64>
//Term ID to list of document ID and position pairs called `Listing`
BTreeMap<u64, Vec<(u64, Vec<u32>)>> 
```

Term IDs are assigned in ascending order, meaning that if we have 1000 terms the corresponding term ids would be 0 to 999. When we turn that assumption into a requirement, the second BTreeMap becomes totally useless and can be replaced by a simple vector:
``` rust
Vec<Vec<(u64, Vec<u32>)>>
```
But where did the term id go? It is encoded in the index of the listing.
``` rust
inv_index[0]
```
gets us the listing for term id 0. Easy and about two times faster:
```
running 4 tests
test index_100kb ... bench:   2,845,210 ns/iter (+/- 17,947)
test index_100mb ... bench: 3,923,865,654 ns/iter (+/- 313,731,489)
test index_10mb  ... bench: 474,433,864 ns/iter (+/- 31,567,143)
test index_1mb   ... bench:  37,581,279 ns/iter (+/- 2,284,396)
```

