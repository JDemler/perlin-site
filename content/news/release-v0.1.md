+++
content_type = "News"
date = "2016-09-22T16:51:26+02:00"
title = "Perlin v0.1"
+++
Perlin received its [first release](https://github.com/JDemler/perlin/releases/tag/v0.1) today. 

In version 0.1 boolean retrieval including phrase queries, filters and persistence on disk, is supported.

The motivation of this release is not to provide usefull functionality but mainly to get feedback on usage and clearness of the library. \
There are still some issues to be resolved for perlin to be a good information retrieval library for boolean retrieval.

These issues are:

* Indexing is incredibly slow
* Loading indices from corrupted data does not yield good or useful errors
* Data in RAM is not compressed
* Indices are non mutable. Once they are create documents can not be added or removed

They will be resolved with perlin 0.2.
