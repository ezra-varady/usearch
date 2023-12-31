<h1 align="center">USearch</h1>
<h3 align="center">
Smaller & Faster Single-File<br/>
Vector Search Engine<br/>
</h3>
<br/>

<p align="center">
<a href="https://discord.gg/A6wxt6dS9j"><img height="25" src="https://github.com/unum-cloud/.github/raw/main/assets/discord.svg" alt="Discord"></a>
&nbsp;&nbsp;&nbsp;
<a href="https://www.linkedin.com/company/unum-cloud/"><img height="25" src="https://github.com/unum-cloud/.github/raw/main/assets/linkedin.svg" alt="LinkedIn"></a>
&nbsp;&nbsp;&nbsp;
<a href="https://twitter.com/unum_cloud"><img height="25" src="https://github.com/unum-cloud/.github/raw/main/assets/twitter.svg" alt="Twitter"></a>
&nbsp;&nbsp;&nbsp;
<a href="https://unum.cloud/post"><img height="25" src="https://github.com/unum-cloud/.github/raw/main/assets/blog.svg" alt="Blog"></a>
&nbsp;&nbsp;&nbsp;
<a href="https://github.com/unum-cloud/usearch"><img height="25" src="https://github.com/unum-cloud/.github/raw/main/assets/github.svg" alt="GitHub"></a>
</p>

<p align="center">
Euclidean • Angular • Jaccard • Hamming • Haversine • User-Defined Metrics
<br/>
<a href="https://unum-cloud.github.io/usearch/cpp">C++11</a> •
<a href="https://unum-cloud.github.io/usearch/python">Python</a> •
<a href="https://unum-cloud.github.io/usearch/javascript">JavaScript</a> •
<a href="https://unum-cloud.github.io/usearch/java">Java</a> •
<a href="https://unum-cloud.github.io/usearch/rust">Rust</a> •
<a href="https://unum-cloud.github.io/usearch/c">C99</a> •
<a href="https://unum-cloud.github.io/usearch/objective-c">Objective-C</a> •
<a href="https://unum-cloud.github.io/usearch/swift">Swift</a> •
<a href="https://unum-cloud.github.io/usearch/golang">GoLang</a> •
<a href="https://unum-cloud.github.io/usearch/wolfram">Wolfram</a>
<br/>
Linux • MacOS • Windows • Docker • WebAssembly
</p>

---

- ✅ Benchmark-topping performance.
- ✅ Simple and extensible [single C++11 header][usearch-header] implementation.
- ✅ SIMD-optimized and [user-defined metrics](#user-defined-functions) with JIT-compilation.
- ✅ Variable dimensionality vectors for unique applications, including search over compressed data.
- ✅ Bitwise Tanimoto and Sorensen coefficients for [Genomics and Chemistry applications](#usearch--rdkit--molecular-search).
- ✅ Hardware-agmostic `f16` & `f8` - [half-precision & quarter-precision support](#memory-efficiency-downcasting-and-quantization).
- ✅ [View large indexes from disk](#disk-based-indexes) without loading into RAM.
- ✅ Space-efficient point-clouds with `uint40_t`, accommodating 4B+ size.
- ✅ Compatible with OpenMP and custom "executors", for fine-grained control over CPU utilization.
- ✅ Supports multiple vectors per label.
- ✅ On-the-fly deletions.
- ✅ [Semantic Search](#usearch--ai--multi-modal-semantic-search) and [Joins](#joins).

[usearch-header]: https://github.com/unum-cloud/usearch/blob/main/include/usearch/index.hpp
[obscure-use-cases]: https://ashvardanian.com/posts/abusing-vector-search

## Comparison with FAISS

FAISS is a widely recognized standard for high-performance vector search engines.
USearch and FAISS both employ the same HNSW algorithm, but they differ significantly in their design principles.
USearch is compact and broadly compatible without sacrificing performance, with a primary focus on user-defined metrics and fewer dependencies.

|                    | FAISS                         | USearch                            |
| :----------------- | :---------------------------- | :--------------------------------- |
| Implementation     | 84 K [SLOC][sloc] in `faiss/` | 3 K [SLOC][sloc] in `usearch/`     |
| Supported metrics  | 9 fixed metrics               | Any User-Defined metrics           |
| Supported ID types | `uint32_t`, `uint64_t`        | `uint32_t`, `uint40_t`, `uint64_t` |
| Dependencies       | BLAS, OpenMP                  | None                               |
| Bindings           | SWIG                          | Native                             |
| Acceleration       | Learned Quantization          | Downcasting                        |

[sloc]: https://en.wikipedia.org/wiki/Source_lines_of_code

Base functionality is identical to FAISS, and the interface must be familiar if you have ever investigated Approximate Nearest Neigbors search:

```py
$ pip install usearch numpy

import numpy as np
from usearch.index import Index

index = Index(
    ndim=3, # Define the number of dimensions in input vectors
    metric='cos', # Choose 'l2sq', 'haversine' or other metric, default = 'ip'
    dtype='f32', # Quantize to 'f16' or 'f8' if needed, default = 'f32'
    connectivity=16, # Optional: How frequent should the connections in the graph be
    expansion_add=128, # Optional: Control the recall of indexing
    expansion_search=64, # Optional: Control the quality of search
)

vector = np.array([0.2, 0.6, 0.4])
index.add(42, vector)
matches, distances, count = index.search(vector, 10)

assert len(index) == 1
assert count == 1
assert matches[0] == 42
assert distances[0] <= 0.001
assert np.allclose(index[42], vector)
```

## User-Defined Functions

While most vector search packages concentrate on just a couple of metrics - "Inner Product distance" and "Euclidean distance," USearch extends this list to include any user-defined metrics.
This flexibility allows you to customize your search for a myriad of applications, from computing geo-spatial coordinates with the rare [Haversine][haversine] distance to creating custom metrics for composite embeddings from multiple AI models.

![USearch: Vector Search Approaches](https://github.com/unum-cloud/usearch/blob/main/assets/usearch-approaches-white.png?raw=true)

Unlike older approaches indexing high-dimensional spaces, like KD-Trees and Locality Sensitive Hashing, HNSW doesn't require vectors to be identical in length.
They only have to be comparable.
So you can apply it in [obscure][obscure] applications, like searching for similar sets or fuzzy text matching, using [GZip][gzip-similarity] as a distance function.

> Read more about [JIT and UDF in USearch Python SDK]().

[haversine]: https://ashvardanian.com/posts/abusing-vector-search#geo-spatial-indexing
[obscure]: https://ashvardanian.com/posts/abusing-vector-search
[gzip-similarity]: https://twitter.com/LukeGessler/status/1679211291292889100?s=20

## Memory Efficiency, Downcasting, and Quantization

Training a quantization model and dimension-reduction is a common approach to accelerate vector search.
Those, however, are only sometimes reliable, can significantly affect the statistical properties of your data, and require regular adjustments if your distribution shifts.

![USearch uint40_t support](https://github.com/unum-cloud/usearch/blob/main/assets/usearch-neighbor-types.png?raw=true)

Instead, we have focused on high-precision arithmetic over low-precision downcasted vectors.
The same index, and `add` and `search` operations will automatically down-cast or up-cast between `f32_t`, `f16_t`, `f64_t`, and `f8_t` representations, even if the hardware doesn't natively support it.
Continuing the topic of memory-efficiency, we provide a `uint40_t` to allow collection with over 4B+ vectors without allocating 8 bytes for every neighbor reference in the proximity graph.

|              | FAISS, `f32` | USearch, `f32` | USearch, `f16` |     USearch, `f8` |
| :----------- | -----------: | -------------: | -------------: | ----------------: |
| Batch Insert |       16 K/s |         73 K/s |        100 K/s | 104 K/s **+550%** |
| Batch Search |       82 K/s |        103 K/s |        113 K/s |  134 K/s **+63%** |
| Bulk Insert  |       76 K/s |        105 K/s |        115 K/s | 202 K/s **+165%** |
| Bulk Search  |      118 K/s |        174 K/s |        173 K/s | 304 K/s **+157%** |
| Recall @ 10  |          99% |          99.2% |          99.1% |             99.2% |

> Dataset: 1M vectors sample of the Deep1B dataset.
> Hardware: `c7g.metal` AWS instance with 64 cores and DDR5 memory.
> HNSW was configured with identical hyper-parameters:
> connectivity `M=16`,
> expansion @ construction `efConstruction=128`,
> and expansion @ search `ef=64`.
> Batch size is 256.
> Both libraries were compiled for the target architecture.
> Jump to the [Performance Tuning][benchmarking] section to read about the effects of those hyper-parameters.

[benchmarking]: https://github.com/unum-cloud/usearch/blob/main/docs/benchmarks.md

## Disk-based Indexes

With USearch, you can serve indexes from external memory, enabling you to optimize your server choices for indexing speed and serving costs.
This can result in **20x costs reduction** on AWS and other public clouds.

```py
index.save("index.usearch")

loaded_copy = index.load("index.usearch")
view = Index.restore("index.usearch", view=True)

other_view = Index(ndim=..., metric=CompiledMetric(...))
other_view.view("index.usearch")
```

## Joins

One of the big questions these days is how will AI change the world of databases and data-management?
Most databases are still struggling to implement high-quality fuzzy search, and the only kind of joins they know are deterministic.
A `join` is different from searching for every entry, as it requires a one-to-one mapping, banning collisions among separate search results.

| Exact Search | Fuzzy Search | Semantic Search ? |
| :----------: | :----------: | :---------------: |
|  Exact Join  | Fuzzy Join ? | Semantic Join ??  |

Using USearch one can implement sub-quadratic complexity approximate, fuzzy, and semantic joins.
This can come handy in any fuzzy-matching tasks, common to Database Management Software.

```py
men = Index(...)
women = Index(...)
pairs: dict = men.join(women, max_proposals=0, exact=False)
```

> Read more in post: [From Dating to Vector Search - "Stable Marriages" on a Planetary Scale 👩‍❤️‍👨](https://ashvardanian.com/posts/searching-stable-marriages)

## Functionality

By now, core functionality is supported across all bindings.
Broader functionality is ported per request.

|                         |  C++  | Python | Java  | JavaScript | Rust  | GoLang | Swift |
| :---------------------- | :---: | :----: | :---: | :--------: | :---: | :----: | :---: |
| add/search/remove       |   ✅   |   ✅    |   ✅   |     ✅      |   ✅   |   ✅    |   ✅   |
| save/load/view          |   ✅   |   ✅    |   ✅   |     ✅      |   ✅   |   ✅    |   ✅   |
| join                    |   ✅   |   ✅    |   ❌   |     ❌      |   ❌   |   ❌    |   ❌   |
| user-defiend metrics    |   ✅   |   ✅    |   ❌   |     ❌      |   ❌   |   ❌    |   ❌   |
| variable-length vectors |   ✅   |   ✅    |   ❌   |     ❌      |   ❌   |   ❌    |   ❌   |
| 4B+ capacities          |   ✅   |   ❌    |   ❌   |     ❌      |   ❌   |   ❌    |   ❌   |

## Application Examples

### USearch + AI = Multi-Modal Semantic Search

AI has a growing number of applications, but one of the coolest classic ideas is to use it for Semantic Search.
One can take an encoder model, like the multi-modal UForm, and a web-programming framework, like UCall, and build a text-to-image search platform in just 20 lines of Python.

```python
import ucall
import uform
import usearch

import numpy as np
import PIL as pil

server = ucall.Server()
model = uform.get_model('unum-cloud/uform-vl-multilingual')
index = usearch.index.Index(ndim=256)

@server
def add(label: int, photo: pil.Image.Image):
    image = model.preprocess_image(photo)
    vector = model.encode_image(image).detach().numpy()
    index.add(label, vector.flatten(), copy=True)

@server
def search(query: str) -> np.ndarray:
    tokens = model.preprocess_text(query)
    vector = model.encode_text(tokens).detach().numpy()
    matches = index.search(vector.flatten(), 3)
    return matches.labels

server.run()
```

We have pre-processed some commonly used datasets, cleaning the images, producing the vectors, and pre-building the index.

| Dataset                                |            Modalities | Images |                              Download |
| :------------------------------------- | --------------------: | -----: | ------------------------------------: |
| [Unsplash 25K][unsplash-25k-origin]    | Images & Descriptions |   25 K | [HuggingFace / Unum][unsplash-25k-hf] |
| [Conceptual Captions 3M][cc-3m-origin] | Images & Descriptions |    3 M |        [HuggingFace / Unum][cc-3m-hf] |
| [Arxiv 2M][arxiv-2m-origin]            |    Titles & Abstracts |    2 M |     [HuggingFace / Unum][arxiv-2m-hf] |

[unsplash-25k-origin]: https://github.com/unsplash/datasets
[cc-3m-origin]: https://huggingface.co/datasets/conceptual_captions
[arxiv-2m-origin]: https://www.kaggle.com/datasets/Cornell-University/arxiv

[unsplash-25k-hf]: https://huggingface.co/datasets/unum-cloud/ann-unsplash-25k
[cc-3m-hf]: https://huggingface.co/datasets/unum-cloud/ann-cc-3m
[arxiv-2m-hf]: https://huggingface.co/datasets/unum-cloud/ann-arxiv-2m

### USearch + RDKit = Molecular Search

Comparing molecule graphs and searching for similar structures is expensive and slow.
It can be seen as a special case of the NP-Complete Subgraph Isomorphism problem.
Luckily, domain-specific approximate methods exists.
The one commonly used in Chemistry, is to generate structures from [SMILES][smiles], and later hash them into binary fingerprints.
The later are searchable with bitwise similarity metrics, like the Tanimoto coefficient.
Below is na example using the RDKit package.

```python
from usearch.index import Index, MetricKind
from rdkit import Chem
from rdkit.Chem import AllChem

import numpy as np

molecules = [Chem.MolFromSmiles('CCOC'), Chem.MolFromSmiles('CCO')]
encoder = AllChem.GetRDKitFPGenerator()

fingerprints = np.vstack([encoder.GetFingerprint(x) for x in molecules])
fingerprints = np.packbits(fingerprints, axis=1)

index = Index(ndim=2048, metric=MetricKind.Tanimoto)
labels = np.arange(len(molecules))

index.add(labels, fingerprints)
matches = index.search(fingerprints, 10)
```

[smiles]: https://en.wikipedia.org/wiki/Simplified_molecular-input_line-entry_system
[rdkit-fingerprints]: https://www.rdkit.org/docs/RDKit_Book.html#additional-information-about-the-fingerprints

## TODO

- JavaScript: Allow calling from "worker threads".
- Rust: Allow passing a custom thread ID.
- C# .NET bindings.

## Integrations

- [x] GPT-Cache.
- [ ] LangChain.
- [ ] Microsoft Semantic Kernel.
- [ ] PyTorch.

## Citations

```txt
@software{Vardanian_USearch_2022,
doi = {10.5281/zenodo.7949416},
author = {Vardanian, Ash},
title = {{USearch by Unum Cloud}},
url = {https://github.com/unum-cloud/usearch},
version = {0.13.0},
year = {2022}
month = jun,
}
```
