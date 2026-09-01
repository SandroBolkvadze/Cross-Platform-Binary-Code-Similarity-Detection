# Cross-Platform Binary Code Similarity Detection

### Task Overview

* In this project we work on the task of *Binary Code Similarity Detection*. Before we get into technical details, let's explain what this task involves and what difficulties we ran into along the way.

* When a programmer writes C code (for example, some function from the OpenSSL library), that code doesn't run directly on the processor. First, a *compiler* (e.g. gcc, clang) translates it into *machine code* — a sequence of processor instructions. The resulting file is called a *binary*, and its human-readable form is called *assembly*.

* The main problem is that *the exact same C function can produce completely different assembly*. There are several reasons for this:
    1. *Which architecture we compile for* — x86 processors, ARM (phones, embedded devices), and MIPS (routers, IoT) have completely different instruction set architectures (ISAs). Where x86 has `mov ebp, esp`, ARM might have `push {fp, lr}` in the same place. In other words, they share neither instruction names nor register names.
    2. *What optimization level we compile with* — gcc has levels `-O0` (no optimization), `-O1`, `-O2`, `-O3`. At higher levels the compiler actively transforms the code: it eliminates variables, merges loops, performs inlining (i.e. instead of calling a function, it copies the entire body of that function into the current code). As a result, a function compiled at `-O0` and the same function at `-O3` may look completely unlike each other visually. Beyond that, the number of instructions can also differ greatly between them.

For example, here is the `analyze_scores` function (in `math.c`), which is given an array of scores. This code sorts the array and returns the average score plus a `bonus` depending on what kind of scores are in the array (e.g. whether the maximum score is 100, etc.):

```c
    int analyze_scores(int *scores, int n)
    {
        int total = 0;
        int passed = 0;
        int failed = 0;
        int highest = -1;
        int swaps = 0;
        int i, j;

        for (i = 0; i < n; i++) {
            int s = scores[i];

            if (s < 0) {
                s = 0;         
            } else if (s > 100) {
                s = 100;        
            }

            total += s;

            if (s >= 90) {
                passed++;
            } else if (s >= 75) {
                passed++;
            } else if (s >= 50) {
                passed++;
            } else {
                failed++;
            }

            if (s > highest) {
                highest = s;
            }
        }

        if (n == 0) {
            return -1;     
        }

        for (i = 0; i < n - 1; i++) {
            for (j = 0; j < n - 1 - i; j++) {
                if (scores[j] > scores[j + 1]) {
                    int tmp = scores[j];
                    scores[j] = scores[j + 1];
                    scores[j + 1] = tmp;
                    swaps++;
                }
            }
        }

        if (passed > failed) {
            if (highest == 100) {
                return total / n + 2;
            }
            return total / n + 1;
        }

        if (swaps > n) {
            return total / n - 1;
        }

        return total / n;
    }
```

* We can compile this example function with `gcc -m32 -shared -fPIC -o math-x86-O0.so math.c` for the `x86` architecture at `-O0` optimization.

* The function turns into a linear sequence of assembly instructions that can already run on the processor (this is not the full picture, there are more instructions):

![alt text](./images/analyze_scores_x86_O0_linear_disas.png)

* However, we can break the linear sequence of assembly at branches and represent the machine code as a graph, giving us the following result:

![alt text](./images/analyze_scores_x86_O0_graph_disas.png)

* Each `node` (code `block`) in this graph is ordinary assembly instructions, and at the end of every `block` we have branching instructions such as `jmp`, `je`, `jne`, `call`, etc.:

![alt text](./images/analyze_scores_x86_O0_graph_node.png)

* It's also possible for a function to be compiled with different optimization levels (`-O0`, `-O1`, `-O2`, `-O3`) even on the same architecture (ISA), producing code that carries out exactly the same logic but with a different number of assembly instructions and a different graphical representation as well. For example, here is the same `analyze_scores` function, compiled on the same 32-bit `x86` architecture, but with `-O3` optimization, represented as a graph:

![alt text](./images/analyze_scores_x86_O3_graph_disas.png)

* It's easy to notice that the `analyze_scores` function compiled on the same architecture (`x86`) has two completely different graphical representations above under `-O0` and here under `-O3` optimization. Despite this difference, the two pieces of code are equivalent and carry out the same logic.

* Accordingly, our task is to build a model that, given *two assembly functions* (possibly from different architectures and different optimization levels), answers: *were these two pieces of assembly code compiled from the same original (source) C function or not*. For example, given the two representations of `analyze_scores` above, our model should tell us that these two pieces of assembly code come from the same function (in this case, `analyze_scores`). Most of our models (though not all) convert each function into a vector (*embedding*) such that vectors of different versions of the same source function end up close to each other (high cosine similarity), while vectors of different functions end up far apart.

* This model can have many practical applications, and let's consider a few cases. For example, a security issue was discovered in OpenSSL (a cryptography library) — the well-known *Heartbleed* bug, which is present in our training library openssl-1.0.1f. We want to find out where else this buggy function is used. But the main problem is that for routers, IoT devices, and most other firmware we don't have the source code. We only have the machine code, and that too compiled for different architectures. With a binary similarity model, we can use the embedding of the vulnerable function to search for the same function among millions of unknown binary functions. The same technique is used in identifying malware families or in detecting plagiarism, etc.

* This task also differs from most ML tasks in that it requires largely different metrics. One important and central metric we look at during model training is *retrieval*: given a query function (for example, the `SHA1_Update` function from the OpenSSL library, compiled for the ARM architecture at `-O2` optimization) and a pool of candidates (`1` **positive** — the same function under a different architecture/optimization, and `999` **negative** — completely different functions). When evaluating the `1000` query-candidate pairs, the model needs to rank the **positive** candidate as high as possible. Using this evaluation method, we can look at various metrics, such as the average rank at which the **positive** ends up, how many times out of **Q** queries it landed in the **Top K**, etc.

* One of the main sources of inspiration for this project's idea was the Gemini paper — "Neural Network-based Graph Embedding for Cross-Platform Binary Code Similarity Detection" (Xu et al., CCS 2017). We saw this paper first, and afterward discovered that there is a great deal of other research on this topic with many different models/architectures, and we decided to build this project.

### Repository Structure

```
Cross-Platform-Binary-Code-Similarity-Detection/
│
├── docker/
│   ├── Dockerfile
│   │   └── compilation and extraction environment
│   └── start_container.sh
│
├── data_source/
│   ├── C source code of the libraries
│   ├── openssl-OpenSSL_1_0_1f/
│   ├── openssl-OpenSSL_1_0_1u/
│   ├── zlib-1.3.1/
│   └── sqlite-amalgamation-3530300/
│
├── data_compiled/
│   ├── compiled .o and .so files
│   └── compile.sh
│       └── cross-compilation script (3 arch × 4 opt)
│
├── data_acfg/
│   └── ACFG: graph + 21 numeric features per block
│
├── data_insns/
│   └── instruction sequence (coarse normalization)
│
├── data_insns_rich/
│   └── instruction sequence (rich normalization)
│
├── data_acfg_insns/
│   └── graph + instructions inside blocks (coarse)
│
├── data_acfg_insns_rich/
│   └── graph + instructions inside blocks (rich)
│
├── Extraction Scripts (angr-based)
│   ├── extract_acfg_angr.py            + .sh
│   │   └── ACFG extraction with angr
│   ├── extract_insns_angr.py           + .sh
│   │   └── coarse instruction extraction
│   ├── extract_insns_rich_angr.py      + .sh
│   │   └── rich instruction extraction
│   ├── extract_acfg_insns_angr.py      + .sh
│   │   └── graph + coarse instructions
│   └── extract_acfg_insns_rich_angr.py + .sh
│       └── graph + rich instructions
│
├── Models / Experiments
│   ├── model_experiment_opcode-histogram.ipynb
│   │   └── Model 1: baseline
│   ├── model_experiment_feature-aggregator.ipynb
│   │   └── Model 2: baseline
│   ├── model_experiment_mlp-aggregator.ipynb
│   │   └── Model 3
│   ├── model_experiment_deepsets.ipynb
│   │   └── Model 4
│   ├── model_experiment_s2v.ipynb
│   │   └── Model 5: Structure2Vec (GNN)
│   ├── model_experiment_gin.ipynb
│   │   └── Model 6: GIN
│   ├── model_experiment_gmn.ipynb
│   │   └── Model 7: Graph Matching Network
│   ├── model_experiment_safe.ipynb
│   │   └── Models 8–9: SAFE (coarse & rich)
│   └── model_experiment_hbinsim.ipynb
│       └── Model 10: hierarchical SAFE + GNN
│
└── README.md
```

* The rich variant of SAFE (a separate project on W&B, `binsim_safe_rich`) uses the same SAFE pipeline, just run on the data_insns_rich data.

---

# Data Generation

This turned out to be one of the hardest and most noteworthy parts of the project — unlike some ML tasks, here we didn't have a "ready-made" dataset: we had to generate it ourselves through compilation and binary analysis. Below we describe the whole pipeline and the difficulties we ran into.

### Our Pipeline (IDA Pro vs angr)

* Gemini's original dataset (and the data in most of the papers we saw in this field) is built with *IDA Pro* — a disassembler that isn't free, is expensive (it cost $10,000 last time we checked), and which we didn't have. At first we tried using a model trained on Gemini's published, IDA-built dataset on our own functions, which we had processed with the open source tool **angr**. The results weren't very good, and we quickly figured out why: IDA and angr (and disassembler tools in general) *analyze the same binary slightly differently* — they split it into basic blocks differently, and compute features differently. A model trained on data processed by IDA will run into a *distribution shift* problem on angr's data (i.e., the training and test data are processed differently, and we're asking a model trained on data processed one way to perform well on data processed a different way).

* Accordingly, the only good solution we came up with was to build the entire pipeline — from compilation to feature extraction — from scratch ourselves, using only open source tools (`gcc` cross-compilers + `angr` + `capstone`). This means the training and evaluation data are *processed with the same tools*, so we no longer have a *distribution shift* problem. On top of that, the whole project became fully reproducible — nobody would need to buy IDA Pro.

### Training/Test Libraries

| Library | Version | Purpose |
|---|---|---|
| *OpenSSL* | 1.0.1f | *Training* library + *in-dataset* test evaluation |
| *OpenSSL* | 1.0.1u | Additionally compiled (a later version of the same library); not used in the final experiment evaluations |
| *zlib* | 1.3.1 | *Out-of-dataset* test evaluation — the model never sees this library's functions during training |
| *SQLite* | 3.53.3 (amalgamation) | *Out-of-dataset* test evaluation — the model never sees this library's functions during training |

* We chose OpenSSL-1.0.1f because it's the standard training library used in the Gemini paper (and it's historically interesting — it's the Heartbleed version). zlib and SQLite were deliberately chosen as code from a *completely different domain*: zlib is a file compression library (full of bit-manipulation instructions). SQLite is a large state machine and parser. If the model only memorized the "patterns" of OpenSSL's functions rather than learning general binary similarity, this would show up immediately in the zlib/SQLite metrics as poor results. We watched this generalization metric constantly, and it's one of the main components of our evaluation.

### Compilation: 3 Architectures × 4 Optimization Levels

* We compiled each library in *12 configurations*: {x86 (32-bit), ARM32, MIPS32} × {O0, O1, O2, O3}. We used GCC cross-toolchains:
```
declare -A CMPS=(
    [x86]="gcc -m32"
    [arm32]="arm-linux-gnueabi-gcc"
    [mips32]="mips-linux-gnu-gcc"
)
```

* The entire environment lives in a *Docker* container (container/Dockerfile, base ubuntu:22.04), where the cross-compilers, gcc-multilib (for 32-bit x86), i386 libc, and Python packages (angr, capstone, networkx) are installed together. This decision mattered for two reasons: (1) installing cross-compilers in Kaggle/Colab notebooks is a hassle, so we ran the data generation locally, offline, and uploaded and stored the results on GitHub; (2) the container makes the generation reproducible and removes the need to install these tools locally in our own environment (for example, on Fedora Linux I couldn't install some of these tools at all and it took a lot of struggling, whereas it was much easier to do this inside Docker's Ubuntu environment).

* We compiled all .c files directly into objects and then linked them into one `.so`:

## .c files -> .o objects
```sh
$cc -$opt -DOPENSSL_NO_ASM -fPIC $INC -c "$c" -o "$OBJ/....o" 2>/dev/null || true
```

## .o files -> one large .so library
```sh
$cc -shared -fPIC -Wl,--allow-multiple-definition -o "....so" $OBJ/*.o
```

Here we used three compiler flags that we considered important and worth mentioning:
1. `-DOPENSSL_NO_ASM` — we disabled OpenSSL's hand-written assembly optimizations by adding this macro, so that the same C code would be compiled on every architecture. If this macro were left out, x86's crypto functions would end up being entirely different, hand-written **inline** assembly code, and we could no longer consider the resulting machine code as coming from "one source function," since the assembly would be hand-written rather than compiler-generated.

2. `-Wl,--allow-multiple-definition` — some of OpenSSL's `.c` files repeat symbols, and without this the linker would crash.

3. `2>/dev/null || true` — some `.c` files (platform-specific, e.g. VMS/Windows code) wouldn't compile. In such cases we simply skip those `.c` files.

* **We did not use -s (strip)**. The symbol table is preserved, because function names (`SHA1_Update`, `inflate`, ...) are our **labels** and we need them for binary analysis. Functions with the same name across different (arch, opt) binaries form **positive** pairs of one class. In a real deployment, names likely wouldn't be present in the binary, and we don't need names during actual deployment either. We need the symbol table for training and evaluation (as **ground truth**).

### Extraction with angr

* For each .so file we run angr's CFGFast(normalize=True) analysis, which splits the given binary into functions and represents each function as a **Control Flow Graph** (CFG), where the vertices are **basic blocks** (an uninterrupted sequence of instructions with no branches), and the edges are possible branches (jump/branch, as mentioned above).

* Not every extracted function is usable (not good for training), so we decided to filter these functions:
    * We filter out PLT stub functions and angr's SimProcedures;
    * We filter out unnamed functions (with the `sub_...` prefix). They can't have a **label** in our data;
    * We filter out compiler-inserted functions (_init, _fini, frame_dummy, register_tm_clones, __libc_csu_*, ...). These are nearly identical across every binary and would fill the dataset with trivial pairs;

* We stored the extraction results in .json files. Each line contains information about various attributes of one function. Each line has this format: ({`src`, `fname`, `n_num`, `succs`, ...}), where `fname` is the function name, `n_num` is the number of blocks (if the extraction was done as a cfg; otherwise this attribute won't be in the function's json).

### Difficulties

We had to redo the data generation several times (due to our own mistakes), and there were quite a few challenges along the way. We've written up the most notable ones:

1. One difficulty was that **angr** ran slowly, so the extraction process wasn't very fast either. `CFGFast` takes a few minutes per binary (more for large binaries), and in total we had **48 binaries** to analyze (4 libraries × 12 configurations), and we repeated this 5 times for all 5 data variants (i.e., the acfg, insns, insns_rich, etc. extraction variants). On top of that, computing the betweenness centrality feature (a graph vertex feature that tells us how many shortest paths pass through a given vertex) in the ACFG variant on large graphs (we used the `networkx` library for this) slowed the process down even further. One full extraction run took several hours (2-3 hours).

2. Setting up and running the cross-compilers was also a struggle. In Docker, we first tried installing the cross-compilers on a Ubuntu 24 image, but couldn't manage it, because some of these compilers weren't in apt. We decided to switch to an older Ubuntu 22 image, and installed the compilers there.

3. angr fails to recognize some functions by name (they remain as `sub_...`) and can't extract them. This means we don't always have all 12 versions (`3 arch` x `4 opt`) of the same source function. Some classes have fewer than 12 members. So, when evaluating on the test data, we only take **positive** pairs from classes (we split functions into different classes by name) that have at least 2 members.

4. Small functions with 1-2 blocks carry almost no information. Models had trouble capturing similarities/differences between small functions. So when reading in the data we keep a `min_blocks` hyperparameter (in the later trained models we mostly used `min_blocks=4`, based on metrics obtained from earlier models). Functions with fewer than `4` blocks were filtered out of the dataset. As it turned out, and as we later found, this is a standard, already-used practice (some papers filter out functions under 5 blocks). In the `mlp-aggregator` experiments it turned out that a model trained on data with `min_blocks=4` had better generalization results than one trained with `min_blocks=0`.

5. We ran training on Kaggle/Colab GPUs (`Kaggle T4`, `Colab T4`), and several models would crash from GPU memory. Some trainings were run via Kaggle's Commit (Save & Run All), and we logged weights/metrics to W&B right away.

* This part of the project took us a long time. In total we created *5 different representations* of the data, and trained various model architectures on these representations (and some models simply wouldn't train on some representations).

**1) data_acfg - ACFG (Attributed Control Flow Graph): graph + 21 numeric features per block.**

This approach is similar to the approach in the Gemini paper. Each basic block is described by hand-selected numeric features. The paper presented 7 features, but we expanded this to **21** features as part of our feature engineering:

```python
FEATURES = ['str', 'imm', 'branch', 'call', 'insns', 'arith', 'outdeg',
        'betw', 'logic', 'shift', 'mul', 'div', 'move', 'cmp',       
        'pushpop', 'meminsn', 'fpsimd', 'indeg', 'operands', 'mnems', 'size']
```
That is, we count: the number of string (`str`) and numeric constants (`imm`), `branch`/`call` instructions, the counts of arithmetic (`arith`)/logical (`logic`)/shift/multiply-divide (`div`, `mul`)/move/comparison (`cmp`)/push-pop (`pushpop`)/memory (`meminsn`)/floating-point instructions (`fpsimd`), structural graph characteristics (out-degree, in-degree, betweenness centrality), the total number of operands (`insns`), the number of unique mnemonics (`mnems`), and the block's size in bytes (`size`). We manually grouped each architecture's mnemonics into semantic categories (for example, x86's `add`, ARM's `adds`, and MIPS's `addiu` all fall into the `arith` category). This is a small amount of cross-architecture normalization.

Let's also note one technical detail: to count the `str` feature (the number of strings used in the code, referenced via a constant pointing to a string), we use a heuristic that checks whether an `immediate` address in the assembly points to a `printable` byte in the code's `read-only` section. This isn't an exact method, but we considered it a reasonably good approximation.

* In the end, each such datapoint is represented like this in `acfg`'s .json files:

```json
{
    "src": "openssl_1_0_1f-x86-O0/ASN1_BIT_STRING_set", 
    "fname": "ASN1_BIT_STRING_set", 
    "n_num": 3, 
    "succs": [[1], [2], []], 
    "features": [[0.0, 2.0, 1.0, 1.0, 5, 1.0, 1, 0.0, 0.0, 0.0, 0.0, 0.0, 1.0, 0.0, 2.0, 0.0, 0.0, 0, 7.0, 4, 12], [0.0, 3.0, 1.0, 1.0, 7, 2.0, 1, 0.5, 0.0, 0.0, 0.0, 0.0, 1.0, 0.0, 3.0, 3.0, 0.0, 1, 10.0, 5, 24], [0.0, 1.0, 1.0, 0.0, 4, 1.0, 0, 0.0, 0.0, 0.0, 0.0, 0.0, 1.0, 0.0, 0.0, 1.0, 0.0, 1, 4.0, 4, 8]]
}
```

* `n_num` is the number of vertices/blocks in the graph.
* `succs` is the adjacency list within the graph.
* `features` is an array of arrays, where each array corresponds to the `21` features of one block of the function.

**2) data_insns - instruction sequence, coarse normalization.**

The downside of the previous data extraction variant is that hand-picking 21 features simply loses a lot of information about what the code actually does. So in this part *the bottleneck may not be the model architecture, but the hand-picked block feature selection*. Choosing good/rich block features might give more benefit than improving and scaling the model architecture. That's why (one of the reasons) we wrote a second extractor, which stores a function not as a matrix of hand-picked numeric features, but as a *sequence of normalized instructions*. We used this extraction as input for SAFE-style sequence models.

The normalization here is **coarse** (that's what we called it): we kept the mnemonic (the instruction's first token). We converted operands into types:

``` 
mov ebp, esp          →  mov reg reg
```

``` 
ldr r3, [r5, #4]      →  ldr reg mem
```

```
addiu $sp, $sp, -40   →  addiu reg reg imm
```

(an immediate operand is marked with the `imm` token, or the `str` token if a string is stored at that address). The full vocabulary turned out to be fairly compact (656 tokens), and architecture-specific elements (register names, specific offsets, addresses) were removed. We train the Word2vec model jointly on the corpus of all three architectures (ISAs), so the instruction embedding space came out shared/common across all three ISAs.

**3) data_insns_rich - instruction sequence, rich normalization.**

Generating this variant of the data was needed for one specific *experiment*. We wanted to measure exactly what benefit (or drawback) detailed operand information brings to the model. Unlike *coarse*, the rich version stores *actual register names (each architecture has its own register naming standard), exact values of small immediates (|imm| ≤ 5000), and the structure of memory accesses* [base+index+disp]. We can compare the *coarse* and *rich* variants:

```
coarse:  mov reg reg          rich:  mov ebp esp
```

```
coarse:  ldr reg mem          rich:  ldr r3 [r5+4]
```

```
coarse:  addiu reg reg imm    rich:  addiu $sp $sp -40
```

We had this hypothesis: rich tokens directly provide more architecture-specific signal (exact offsets and registers say a lot about a function), but *the vocabularies across architectures become nearly disjoint* — the `ebp` register only exists on `x86`, `$sp` only on `MIPS`, etc. So we expected cross-arch generalization to get worse. The vocabulary here consists of *93,926* tokens instead of *656*. Below we'll compare these two experiments, SAFE vs SAFE-rich, and see whether our hypothesis was confirmed or not.

**4-5) data_acfg_insns and data_acfg_insns_rich - graph + instructions inside blocks.**

The last two variations combine both previous approaches. We store the function as a CFG graph (as in the first approach), but instead of hand-written features, we store the normalized instruction sequence, as in the second approach (in both coarse and rich versions). We needed this processed data for the hierarchical model (HBinSim), which first generates block embeddings from instructions and then runs message passing on the graph.

### Dataset Size

After extraction (before filtering), the dataset looks like this:

| Library | Binaries | Functions | Unique functions (classes) |
|---|---:|---:|---:|
| openssl-1.0.1f | 12 | 70,968 | 6,627 |
| openssl-1.0.1u | 12 | 70,424 | 6,602 |
| sqlite3-3.53.3 | 12 | 25,637 | 5,804 |
| zlib-1.3.1 | 12 | 4,034 | 500 |
| *Total* | *48* | *~171,000* | - |

A "class," as mentioned above, is the set of functions with the same name (compiled for different architectures + different optimization levels). After the `min_blocks=4` filter, the working dataset shrinks slightly, but the structure stays the same and the number of functions remains sufficient for training and testing.

---

# Evaluation / Metrics

Before moving on to the models, let's describe the evaluation approach that we fixed during training/testing, and which we used throughout to compare models. This is an important element, because we wrote the evaluation part first and fixed it before starting work on the models, and we barely changed it afterward. This way, comparing the models to each other would be fairer and more accurate.

### Split

* *openssl-1.0.1f* is split into train/val/test parts with an *80/10/10* ratio. The split is done *by class (function name), not by individual function instances (taking architecture and optimization into account)*. This decision mattered, because if we had split by instance (accounting for arch-opt variants), the same function could end up both in training (e.g., the `x86`/`-O0` version) and in test (e.g., the `ARM`/`-O3` version), and the model would see "familiar" functions at test time, causing data leakage. When splitting by class, the test functions are entirely new to the model and it hasn't seen them during training.

* *zlib-1.3.1* and *sqlite3* are used entirely for evaluation. None of their functions appear in training. We use these libraries to measure how well the model learned the general task of binary similarity, and whether the model generalizes well or simply memorized patterns of OpenSSL's functions.

### Metrics

1. *ROC AUC / PR AUC*: we generate 10,000 pairs (half positive — two instances of the same class; half negative — different classes) and look at how well the model's returned similarity score ranks **positive** pairs above **negative** pairs. In some ways this is a good metric, because it helps us understand whether the model distinguishes **positive** pairs from **negative** pairs at all. However, as we'll see below, for strong models this metric quickly becomes *saturated* — nearly every good model scores **0.99+** on this metric, and the differences stop being very visible.

2. *Retrieval: Recall@k and MRR at various pool sizes (10 / 100 / 1000).* For a query function we create a pool: 1 **positive** (another instance of the same class) + (pool−1) randomly selected **negative** functions. We rank the candidates by cosine similarity and record the positive's rank. We repeat this process `Q` times. *Recall@k* measures how many times the **positive** function landed in the **top-k**; *MRR* (Mean Reciprocal Rank) computes the average of 1/rank across these `Q` repetitions. We repeat this process for a total of *n_queries = 3,000*. *The main metrics are Recall@1 pool=1000 and MRR pool=1000*. *Recall@1 pool=1000* measures how often the model puts the correct candidate in first place out of 1000 candidates, while *MRR pool=1000* measures the average rank of the correct candidate in a pool of 1000. Naturally, models scored higher at pool=10 than at pool=1000.

3. *Per-axis breakdown: XA / XO / XM.* Every (query, target) pair falls onto one of the following 'axes':
   * *XA* (cross-architecture): this pair is from different architectures, but the same optimization;
   * *XO* (cross-optimization): this pair is the same architecture, but different optimization;
   * *XM* (cross-mixed): this pair is from different architectures and different optimizations.

The metric computed on each axis measures how robust the model is to the specific compilation-time changes chosen. For example, the metrics computed on *XA* measure how well the model can find similarity between binaries compiled for two different architectures. The metrics computed on *XO* measure how well the model can find similarity between code of the same architecture but different optimization levels, while *XM* is the hardest metric and shows how well the model finds similarities between binaries when both components (architecture and optimization) differ. Breaking the metrics down by 'axis' was very interesting, and we learned a lot from it.

### General Training Framework

Every trained model (starting from model 3 onward) is trained with the same approach:

* *Siamese scheme + InfoNCE loss*: in a batch we take B classes, and from each class we pick an anchor-positive pair (two random instances of the same function). For an anchor instance, the 'correct' match is its **positive** pair (in the same class), while the remaining B−1 **positive** instances in the batch are **negative** for this anchor. For every anchor we compute the embedding, then compute a cosine-similarity score against the remaining B classes. The loss is cross-entropy over these B numbers, and we try to maximize the similarity score of the anchor's positive pair. We do this for every anchor, average the cross-entropy losses computed over these anchors, and that's the final loss. This is the InfoNCE-loss approach; we also tried triplet-loss and MSE-loss approaches, but InfoNCE loss turned out best.

* *temperature t = 0.05* — we fixed this value based on early experiments (out of 0.05 / 0.1 / 0.15, 0.05 consistently performed best, e.g., on mlp-aggregator we got R@1 of 0.462 / 0.446 / 0.434 respectively).

* *Optimization*: Adam, lr=1e-3/1e-4/1e-5, 5000/10000 steps, early stopping on val ROC AUC, restoring the best-step model weights.

* *Logging*: every run is logged to **W&B** — including configuration, metrics, plots (score distribution, Recall@k curves, per-axis heatmaps). We write all hyperparameters into the run name itself, to make it easier to tell models apart from the tables.

---

# Models

In total we tested *12 models*, distributed across 10 W&B projects (Structure2vec and GIN share one project), in this order (this is chronological order, and, for the most part, goes from weaker to stronger):

1. [binsim_baseline_opcode-histogram](https://wandb.ai/sbolk23-free-university-of-tbilisi-/binsim_baseline_opcode-histogram)
2. [binsim_baseline_feature-aggregator](https://wandb.ai/sbolk23-free-university-of-tbilisi-/binsim_baseline_feature-aggregator)
3. [binsim_mlp-aggregator](https://wandb.ai/sbolk23-free-university-of-tbilisi-/binsim_mlp-aggregator)
4. [binsim_deepsets](https://wandb.ai/sbolk23-free-university-of-tbilisi-/binsim_deepsets)
5. [binsim_gnn](https://wandb.ai/sbolk23-free-university-of-tbilisi-/binsim_gnn) (Structure2vec)
6. [binsim_gin](https://wandb.ai/sbolk23-free-university-of-tbilisi-/binsim_gnn) (GIN)
6. [binsim_gin](https://wandb.ai/sbolk23-free-university-of-tbilisi-/binsim_gnn) (GAT)
7. [binsim_gmn](https://wandb.ai/sbolk23-free-university-of-tbilisi-/binsim_gmn)
8. [binsim_safe](https://wandb.ai/sbolk23-free-university-of-tbilisi-/binsim_safe)
9. [binsim_safe_rich](https://wandb.ai/sbolk23-free-university-of-tbilisi-/binsim_safe_rich)
10. [binsim_hbinsim](https://wandb.ai/sbolk23-free-university-of-tbilisi-/binsim_hbinsim)
11. [binsim_safe_transformer](https://wandb.ai/sbolk23-free-university-of-tbilisi-/binsim_safe_transformer)

Our approach was the same everywhere: we'd run the model with a baseline configuration, then change parameters gradually, one at a time, and watch on W&B what improved or worsened the result. Below, these changes are described for each model.

---

## 1. Opcode Histogram (baseline)

*Idea:* The simplest representation we could think of — describe a function by how many times each instruction appears in it. We collect a token vocabulary, turn each function into a normalized histogram (a frequency vector), and measure similarity directly with cosine similarity. *There's no training at all* — this is the floor that every trained model needs to beat, otherwise training is pointless.

*What we tried:* mnemonic_only True/False (counting only the mnemonic vs. the full normalized token) and min_blocks ∈ {0, 4, 10}.

**Results (champion: sim_fn=cos, mnem=False, min_blocks=0):**

| Metric | openssl | zlib | sqlite3 |
|---|---:|---:|---:|
| Recall@1 (pool 1000) | *0.105* | 0.109 | 0.083 |
| MRR (pool 1000) | 0.116 | 0.123 | **0.093** |
| ROC AUC | 0.549 | 0.523 | 0.491 |
| XA / XO / XM R@1 | *0.000* / 0.365 / *0.000* | | |

*Analysis:*
* ROC AUC ≈ *0.5 (0.49–0.55) — essentially a coin flip*. This baseline told us exactly what we ran it for: the task can't be trivially solved.
* The most telling number here is *XA = 0.000, exactly zero*. The reason is mechanical: the sets of mnemonics for x86, ARM, and MIPS barely overlap at all (`mov`/`ldr`/`lw` are three different worlds). Two histograms that don't share a nonzero coordinate are orthogonal under cosine similarity — the model can never find a cross-arch pair. XO = 0.365, by contrast, is normal: within a single architecture, optimization levels partially preserve the mnemonic distribution.
![alt text](./images/histogram_per_axis_bars.png)
  
* The mnem parameter changed almost nothing (sqlite3 MRR 0.0925 vs 0.0923).
* *plots/score_distribution* for this model looks exactly like what an AUC of 0.5 corresponds to: the green (positive) and red (negative) histograms lie almost perfectly on top of each other — there simply is no dividing boundary.
![alt text](./images/score_distribution_histogram.png)

*Conclusion:* The main thing this baseline taught us is that *cross-architecture similarity is fundamentally out of reach through surface-level statistics*. We need a representation that looks past the architectural surface.

---

## 2. Feature Aggregator (baseline)

*Idea:* The second baseline already uses the features we created: we simply *sum* the 21-dimensional feature vectors of all of a function's blocks into one vector and again measure similarity directly (cosine or L2), *without training*. We check: how much do we get from our feature engineering alone, without a learned component?

*What we tried:* sim_fn ∈ {cos, l2}, n_feat ∈ {7 (Gemini's original), 14, 21}, min_blocks ∈ {0, 10, 15}.

**Results (champion: cos, n_feat=14, min_blocks=10):**

| Metric | openssl | zlib | sqlite3 |
|---|---:|---:|---:|
| Recall@1 (pool 1000) | *0.220* | 0.184 | 0.148 |
| MRR (pool 1000) | 0.269 | 0.243 | **0.204** |
| ROC AUC | 0.887 | 0.908 | 0.856 |
| XA / XO / XM R@1 | 0.132 / 0.536 / 0.085 | | |

*Analysis:*
* Compared to the histogram, the jump is sharp: AUC 0.55 → 0.89, sqlite3 MRR 0.093 → 0.204, and most importantly — *XA came alive* (0.000 → 0.132). This happens precisely because the features (arith, branch, call, degrees, ...) *are built on semantic categories rather than concrete mnemonics* — x86's `add` and MIPS's `addiu` both increment the same thing. In other words, cross-arch invariance is "built into" our extractor, and that's enough to move it off zero.
![alt text](./images/feature_agg_per_axis_bars.png)

* An interesting finding: **n_feat=14 beats n_feat=21** (sqlite3 MRR: 0.204 vs 0.179; 7 features is even lower — 0.153). At first glance this seems paradoxical — more information makes things worse? The reason is that we have no training: the features are on **completely different scales** (`size` is in the hundreds of bytes, `betw` is in the [0,1] range), and in raw cosine similarity the large-scale features dominate. Adding all 21 features introduced exactly these scale-dominant columns (e.g., size, operands) and distorted the similarity measure. This observation points directly to the next step: we need a model that learns the features itself.
* Increasing `min_blocks` improves the openssl number (larger functions are more distinguishable), though this is also partly an artifact of simplifying the evaluation — the pool fills up with "larger" functions.
* *plots/score_distribution*: here the two histograms are already separated (the mass of positives is shifted to the right), but the overlap zone is still very wide — that's why AUC is decent (0.87) while retrieval is weak (0.227): among 1000 candidates there's almost always a negative that lands above the positive in this wide overlap.
![alt text](./images/score_distribution_feature_agg.png)
  
*Conclusion:* Feature engineering managed to learn cross-arch invariance, but the ceiling is low.

---

## 3. MLP Aggregator

*Idea:* The first *trained* model, but with a minimal architecture: we still sum up a function's block features into one 21-dimensional vector, but now it passes through an *MLP* (hidden layers → output embedding dimension), and the whole model is trained with contrastive loss. That is, the previous baseline plus a learned projection. This gives us the answer to: how much do we get from a learned metric alone (what profit did training give us), without changing the representation?

*What we tried:* hidden architectures (from [128] up to [128,256,512,512,512,512]), out_dim ∈ {64, 128, 256}, dropout, batch size, temperature ∈ {0.05, 0.1, 0.15}, normalization variants, min_blocks ∈ {0, 4}, loss ∈ {InfoNCE, triplet (batch-hard)}.

**Results (champion: hiddens=[128], out=64, lr=1e-3, dropout=0, bs=128, min_blocks=4, loss=triplet/batch-hard, margin=0.5):**

| Metric | openssl | zlib | sqlite3 |
|---|---:|---:|---:|
| Recall@1 (pool 1000) | *0.535* | 0.421 | 0.348 |
| MRR (pool 1000) | 0.613 | 0.514 | **0.430** |
| ROC AUC | 0.973 | 0.957 | 0.921 |
| XA / XO / XM R@1 | 0.664 / 0.571 / 0.432 | | |

*Analysis:*
* Again a big jump: sqlite3 MRR 0.204 → *0.430* (openssl R@1 0.220 → 0.535), i.e. the learned projection *doubled* the baseline's result. InfoNCE solves exactly the problem the feature-aggregator had: the network learns for itself which feature carries what weight and combination is informative, and the scale problem disappears. Indirect confirmation of this: here the best result comes from the full `n_feat=21` — with learned weights, "extra" features provide benefit instead of harm.
![alt text](./images/mlp_per_axis_bars.png)

* **min_blocks=4 turned out to be one of the most influential parameters**: runs with min_blocks=0 averaged R@1 of 0.328, while min_blocks=4 runs averaged 0.447. Small functions (1-3 blocks) are basically indistinguishable in the aggregated feature space and pollute training too (false-negative pairs in InfoNCE batches). After this observation, min_blocks=4 was kept as the default in every subsequent model.
* In the temperature sweep, t=0.05 turned out best (0.462 vs 0.446 vs 0.434 on a comparable config) — a lower t forces the network to push hard negatives apart more aggressively. This too was fixed for every following model.
* Depth effect: on the main metric, the *single-layer [128] won*, while the deeper [128,256,512] is only slightly ahead on openssl (R@1 0.551) — i.e. the additional capacity is spent overfitting to OpenSSL's idioms. The bottleneck is clearly elsewhere, not in model capacity.
* *plots/score_distribution*: for the first time a clear *bimodal* picture appears — the mass of positives gathers at 1, negatives at 0, though a noticeable "bridge" remains in the middle. ROC AUC is already 0.985 — from here on AUC can barely measure the difference between models anymore, and all the signal is in pool-1000 retrieval.
![alt text](./images/score_distribution_mlp_agg.png)
  
*Conclusion:* A learned metric is necessary, but the ceiling is now set by *premature aggregation*: by summing blocks we lose information about which blocks make up the function. Next step — move the aggregation to after the learned transformation.

---


## 4. DeepSets

*Idea:* The DeepSets architecture treats a function as a *set of blocks* (ignoring the graph's edges): each block's 21-feature vector first passes individually through a shared MLP (*φ*), then the block representations are combined into one vector via pooling, and finally a second MLP (*ρ*) builds the final embedding. The difference from mlp-aggregator is exactly one thing — *aggregation happens after the learned φ, not before*.

*What we tried:* pooling ∈ {mean, max, sum, mean_max, simple_attention, **mean_max_attention**}, φ/ρ architectures, out_dim, batch size ∈ {128, 512}, normalization (BatchNorm vs LayerNorm — more on this below), dropout.

**Results (champion: pool=mean_max_attention, φ=[128,256,256], ρ=[128,64], bs=128, min_blocks=4):**

| Metric | openssl | zlib | sqlite3 |
|---|---:|---:|---:|
| Recall@1 (pool 1000) | *0.725* | 0.497 | 0.455 |
| MRR (pool 1000) | 0.793 | 0.593 | **0.546** |
| ROC AUC | 0.993 | 0.979 | 0.949 |
| XA / XO / XM R@1 | *0.826* / 0.709 / 0.680 | | |

*Analysis:*
* sqlite3 MRR 0.430 → *0.546* (openssl R@1 0.535 → 0.725). The effect of the per-block φ is exactly what we expected: computing a nonlinear transform before the sum preserves differences between blocks that pre-summing erased (two completely different functions can have the same summed feature vector, but different block composition).
![alt text](./images/deepsets_per_axis_bars.png)
  
* *The choice of pooling turned out to be decisive.* The combination of three poolings — mean + max + attention (concatenated) — outperformed the individual mean/max/attention variants by ~0.10-0.12 on average across runs (average R@1: mean 0.518, max 0.550, mean_max 0.569, *mean_max_attention 0.669*). Intuition: mean captures the function's "overall profile," max captures the most salient blocks (e.g., a crypto loop's core), and attention learns which blocks are discriminative. All three give different information.
* *A BatchNorm bug that code review caught.* In the initial implementation, the incoming normalization looked like this: self.in_norm(X.reshape(-1, D)) — the batch tensor (B, N, D) flattened into (B·N, D) shape, where N is the padded maximum number of blocks. The problem: *zero-padding was also included in BatchNorm's statistics* — the means were artificially pulled toward zero, variances were distorted, and the fraction of padding varied from batch to batch, confusing the statistics further. Downstream masking (* m) couldn't fix this, because statistics were computed before masking. The fix: masked normalization (statistics computed only over valid blocks) / switching to LayerNorm. This bug didn't invalidate previous comparative results (both compared models had the same bug), but training became noticeably more stable after the fix.
* Increasing batch size to 512 helped in-dataset metrics (more in-batch negatives = a stricter training task), but on the main out-of-dataset metric the champion still turned out to be the bs=128 run — an in-dataset gain doesn't automatically transfer to generalization.
* *plots/score_distribution*: the mass of negatives now sits tightly around 0, positives around 1; the overlap remains only as a narrow band. Most of the remaining errors are stuck in exactly this narrow tail — mainly O0↔O3 pairs, where the compiler transforms the code the most.
![alt text](./images/score_distribution_deepsets.png)
  
*Conclusion:* A block-level learned representation plus rich pooling is such a strong combination that it gives very high results *even without using the graph structure*. This naturally raises the question: if a set of blocks gives this much, how much will the graph's topology add? We moved to GNNs to check this.

---

## 5. Structure2vec (Gemini's GNN)

*Idea:* This is the literature baseline — our own PyTorch reimplementation of the Xu et al. (CCS 2017, "Gemini") architecture. Structure2vec runs on the function's ACFG: each block's initial feature vector passes through a linear layer, then over *T iterations* every block aggregates its neighbors' states (aggregation), passes them through a multi-layer MLP, and updates its own state. Finally, the block states are combined via readout into the function's embedding. So unlike DeepSets, here *edges (control flow) are actually used*.

*What we tried:* T ∈ {2, 3, 5} (propagation iterations), hidden ∈ {64, 128, 256}, msg_layers ∈ {2, 3}, aggregation ∈ {sum, mean}, readout ∈ {sum, mean, max}, n_feat ∈ {7, 14, 21} (7 = Gemini's original).

**Results (champion: T=5, hidden=256, msg_layers=3, aggr=sum, readout=sum, lr=5e-4):**

| Metric | openssl | zlib | sqlite3 |
|---|---:|---:|---:|
| Recall@1 (pool 1000) | 0.684 | 0.532 | 0.456 |
| MRR (pool 1000) | 0.759 | 0.615 | **0.544** |
| ROC AUC | 0.994 | 0.971 | 0.952 |
| XA / XO / XM R@1 | 0.804 / 0.672 / 0.624 | | |

*Analysis:*
* *This is one of the most interesting and surprising results in the project.* Structure2vec, which uses more information than DeepSets (blocks + edges vs. only blocks), *did not surpass* it on the main metric (sqlite3 MRR 0.544 vs 0.546; it's even slightly ahead on zlib — 0.615 vs 0.593), and it *fell behind* on every in-dataset axis: 0.684 vs 0.725 overall, 0.804 vs 0.826 XA, 0.624 vs 0.680 XM. In other words, adding edges gave, at best, zero additional benefit.
![alt text](./images/S2V_per_axis_bars.png)
  
* Why? Our interpretation: *CFG topology is not architecturally invariant*. For the same C function, the CFGs of x86 and MIPS differ structurally — MIPS delay slots, ARM conditional execution, x86's complex addressing modes produce different numbers of basic blocks. And going from O0→O3, inlining, loop unrolling, and branch elimination directly rewrite the topology. So message passing mixes in *noise that is correlated exactly with the axes on which we need invariance*. This is confirmed by the fact that the biggest drop is on XM (cross-arch + cross-opt simultaneously) — where topological distortion is at its maximum.
* Regarding parameters: T=5 averaged 0.656, T=2 was 0.572. More iterations helps (a larger receptive field). n_feat: 21 > 14 > 7 — i.e. Gemini's original set of 7 features is clearly insufficient in our setup. aggr=sum averaged 0.661 vs mean 0.627 — sum preserves degree information that mean erases.
* *plots/score_distribution*: resembles DeepSets' picture, but the right tail of the negative distribution is noticeably thicker — i.e. the model more often gives high scores to pairs of different functions. This is exactly the signature of topological "false similarity": structurally similar but semantically different functions (e.g., two completely different loop-containing helpers) end up close to each other.
![alt text](./images/score_distribution_s2v.png)
  
*Conclusion:* Having a GNN by itself gives nothing — what matters is *how* neighbors are aggregated (sum-variants clearly beat mean). The natural next step is to take this logic all the way: theoretically the most expressive, injective aggregation → GIN.

## 6. GIN (Graph Isomorphism Network)

*Idea:* GIN is theoretically the most powerful message-passing GNN: neighbor aggregation is done via *sum* (rather than mean/max, which lose information), and the update is MLP((1+ε)·h_v + Σ h_u). Here we check whether injective aggregation fixes what Structure2vec fell short on.

*What we tried:* T ∈ {2, 3, 5}, hidden ∈ {128, 256}, msg_layers, eps learnable ∈ {True, False}, pooling ∈ {sum, mean, mean_max}.

**Results (champion: T=3, hidden=256, msg_layers=3, pool=sum, eps=True):**

| Metric | openssl | zlib | sqlite3 |
|---|---:|---:|---:|
| Recall@1 (pool 1000) | *0.739* | 0.525 | 0.441 |
| MRR (pool 1000) | 0.799 | 0.607 | **0.526** |
| ROC AUC | 0.992 | 0.975 | 0.950 |
| XA / XO / XM R@1 | *0.838* / 0.734 / 0.692 | | |

*Analysis:*
* In-dataset, GIN is the best graph model and even beats DeepSets (R@1 0.739 vs 0.725, XA 0.838 vs 0.826) — i.e. *injective aggregation recovered the information that Structure2vec was losing*. But it falls behind both on the main out-of-dataset metric (sqlite3 MRR 0.526 vs DeepSets' 0.546 and S2V's 0.544) — *edges added nothing to generalization*. This is the strongest confirmation of our hypothesis: on this task, the marginal value of control-flow topology is ~zero, and it can even become harmful once the architecture adds noise to it.
![alt text](./images/gin_per_axis_bars.png)

* **T=3 is optimal, it drops at T=5** (average R@1: T=3 → 0.691, T=5 → 0.608). This is classic **oversmoothing**: after many message-passing iterations, every block's representation within a graph converges toward the others, and graph embeddings become indistinguishable from each other. Interestingly, Structure2vec prefers T=5 instead — its less expressive update rule accumulates information more slowly and reaches oversmoothing later.
* *A technical problem: OOM.* GIN's implementation used a dense adjacency matrix — memory O(B · N²), where N is the maximum number of blocks in a batch. openssl has functions with 2000+ blocks; one such function, padded across a whole batch, would balloon to 2000×2000 and exceed the T4's 16GB. Fix: a max_blocks cap plus bucketing (batching sorted by number of blocks), so similarly sized graphs land in the same batch.
* *plots/score_distribution*: compared to Structure2vec, the thick right tail of negatives noticeably thins out — sum-aggregation really does reduce "structural false positives."
![alt text](./images/score_distribution_gin.png)
  
*Conclusion:* Even the best GNNs can't beat DeepSets' level on the main metric. Two paths remain — either make the comparison mechanism more sophisticated (GMN), or change *the input representation itself* and feed the model actual instructions instead of block features (SAFE).

---


## 7. GMN (Graph Matching Network)

*Idea:* GMN breaks the siamese paradigm: the embeddings of two graphs are *no longer computed independently*. At every propagation iteration, each block aggregates not only its own graph's neighbors, but also looks at *the other graph's blocks* via cross-graph attention. This is theoretically much more powerful — the model directly learns block correspondence (matching) between x86 and ARM.

*What we tried:* T, hidden dim, cross-attention configuration, batch size.

* GMN *can't cache embeddings*. In every other model, pool-1000 retrieval is computed like this: compute the embeddings of the 1000 candidates once, then compute cosine similarity against the query — i.e. 1000 + Q forward passes. In GMN, the score is defined per pair, so for a single query it requires *1000 full forward passes*. 3000 queries × 1000 = 3 million pair computations would need several days on a T4.
* So GMN's retrieval evaluations (including a separately re-run gmn__eval_from_saved_weights on another account) were only run on *~100 queries*. This means the standard error of Recall@1 is ≈ *±0.05–0.06*, and in the per-axis (XA/XO/XM) breakdown, only a few dozen queries remain in each category.
* Most of the remaining runs (out of 24) either only logged the cheaply computed sqlite mrr_pool1000 (0.37–0.65 range, **even on identical configs** — this spread itself is an indicator of the noise scale), or crashed with OOM (cross-graph attention requires O(N₁·N₂) memory per pair).

**Results (champion: T=5, hidden=256, msg_layers=3, pool=sum, bs=64; evaluated at pool=1000, only ~100 queries, ±0.06):**

| Metric | openssl | zlib | sqlite3 |
|---|---:|---:|---:|
| Recall@1 (pool 1000) | ~0.73 | ~0.54 | ~0.59 |
| MRR (pool 1000) | ~0.76 | ~0.63 | **~0.65** |
| ROC AUC | - (not logged) | - | - |
| XA / XO / XM R@1 | 0.16 / 0.79 / 0.13 (noisy — see above) | | |

For comparison: a separately re-run evaluation on another account (gmn__eval_from_saved_weights, also ~100 queries) showed a completely different picture — openssl R@1 ~0.53, zlib R@1 ~0.69, sqlite3 MRR ~0.56, XA/XO/XM 0.31/0.14/0.11.

*Analysis:*
* The per-axis numbers *contradict each other*: the champion run shows XO 0.79 / XA 0.16, while the re-run evaluation shows the opposite — XA 0.31 / XO 0.14 (in other runs we've also seen XO 0.87 / XA 0.04). These can't all be true at the same time; it simply confirms that a per-axis breakdown on ~100 queries is uninformative.
![alt text](./images/gmn_per_axis_bars.png)
  
* The only thing that can honestly be said: *in our compute budget, GMN doesn't beat SAFE (sqlite3 MRR ~0.652 vs 0.685, and that with ±0.06 error), and its inference cost makes it impractical for a real retrieval task anyway* — in a real scenario (a database of millions of functions), a pairwise model isn't indexable. In the literature too (Marcelli et al., USENIX 2022), GMN's advantage is achieved precisely at the cost of this compute expense.
* *plots/score_distribution*: GMN's score distributions on in-dataset pairs are sharply separated (ROC AUC is high), which again underscores our main methodological conclusion — *binary AUC can no longer distinguish between models at this stage*; the only thing that differentiates them is large-pool retrieval.
![alt text](./images/score_distribution_gmn.png)
  
*Conclusion:* The investigation is incomplete, and that's due to compute resource constraints. Our next step — change the *representation* instead of the architecture.

---

## 8. SAFE (Self-Attentive Function Embeddings)

*Idea:* SAFE drops hand-crafted features and the graph entirely. A function is treated as a *sequence of instructions* (linearized assembly). Each instruction is first turned into a vector using *word2vec* (skip-gram, trained by us on a corpus shared across all architectures), then the sequence goes through a **bi-GRU**, and finally several "hops" of **self-attention** aggregate the GRU's output into the function's embedding.

*The decisive detail — normalization.* Instruction tokenization is *coarse* (extract_insns_angr.py): registers → reg, addresses → mem, immediates → imm. For example, mov eax, dword ptr [rbp - 0x18] → **mov reg mem**. The vocabulary across the whole dataset is only *656 tokens total*. This means x86, ARM, and MIPS tokens *live in the same space*, and word2vec can learn that x86's mov reg mem and MIPS's lw reg mem appear in the same context — i.e. *the embedding space becomes architecturally shared on its own*.

*What we tried:* rnn ∈ {GRU, LSTM}, hidden ∈ {64, 128, 256}, attention_hops ∈ {1, 4, 8}, attention_dim, rnn_layers, embedding_dim ∈ {100, 200}, freeze_embeddings ∈ {True, False}, max_len ∈ {500, 1000}, lr. In total *83 runs* — the largest sweep in the project.


**Results (champion: rnn=GRU, hidden=128, out=64, hops=4, attn_dim=128, layers=1, emb=200, freeze=False, max_len=250, lr=1e-3, bs=128):**

| Metric | openssl | zlib | sqlite3 |
|---|---:|---:|---:|
| Recall@1 (pool 1000) | *0.783* | *0.591* | *0.611* |
| MRR (pool 1000) | *0.840* | *0.678* | ***0.685*** |
| ROC AUC | 0.996 | 0.982 | 0.972 |
| XA / XO / XM R@1 | *0.876* / *0.757* / *0.749* | | |

*Analysis:*
* *The biggest single jump in the project*: the main metric goes 0.546 → *0.685* (+25%), and out-of-dataset R@1 goes from *0.497 → 0.591 (zlib)* and *0.455 → 0.611 (sqlite3)*. In other words, SAFE isn't just better — it *generalizes far better to code it has never seen during training*.
* *XA = 0.876* — the cross-architecture axis is the strongest of all. This directly confirms the shared-vocabulary hypothesis: when x86/ARM/MIPS all speak the same 656-token alphabet, crossing between architectures becomes a "dialect" problem rather than a "foreign language" problem.
![alt text](./images/SAFE_per_axis_bars.png)
  
* freeze_embeddings=False beats `True` (average R@1 0.743 vs 0.718) — word2vec gives a good starting point, but contrastive fine-tuning improves it further.
* attention_hops: 4–8 clearly beats 1. A single hop is forced to summarize the whole function with one "focus"; 4 hops can attend in parallel to the beginning, the loop core, and the cryptographic constants.
* GRU ≈ LSTM (the difference is within noise), so we chose GRU — it's cheaper.
* The effect of max_len split in an interesting way: the max_len=1000 run set the project's in-dataset record (openssl R@1 0.808, XA 0.904), but on the main metric the 250-token champion edged it out slightly (sqlite3 MRR 0.685 vs 0.682). In other words, a function's first ~250 instructions are nearly enough for out-of-dataset similarity; the difference is within noise, and we kept the champion as the one better on sqlite3 MRR.
* *plots/score_distribution*: here the distributions are almost fully separated — the mass of negatives is a dense peak near 0, positives near 1, minimal overlap. *The key thing is that this picture holds up on zlib/sqlite3 too* (AUC 0.982 / 0.972), whereas in the GNNs the out-of-dataset distributions merged noticeably.
![alt text](./images/score_distribution_safe.png)

*Conclusion:*
*Representation beat architecture.* A correctly normalized instruction sequence plus a moderately complex sequence model beats any graph architecture that works on hand-crafted features.

---

## 9. SAFE-rich

*Idea:* Exactly the same SAFE architecture, with one change — *tokenization is rich* (extract_insns_rich_angr.py): actual register names are kept (eax, r0, $t1), small immediates (|imm| ≤ 5000) keep their exact value, and memory operands keep their structure ([base+idx+disp]). Vocabulary: *656 → 93,926 tokens*.

*The hypothesis we were testing* (the same one formulated in the data section) had two parts: (a) coarse normalization loses information within a single architecture (mov reg reg can't distinguish mov eax, ebx from mov esp, ebp), so rich tokens might improve the in-dataset result; (b) but the vocabularies would become nearly disjoint across architectures, hurting cross-arch/out-of-dataset generalization.

*What we tried:* 32 runs — hidden ∈ {128, 256}, freeze ∈ {True, False}, lr ∈ {5e-4, 1e-3}, hops, max_len.

**Results (champion: hidden=256, hops=4, attn_dim=64, freeze=True, lr=5e-4, max_len=250):**

| Metric | openssl | zlib | sqlite3 |
|---|---:|---:|---:|
| Recall@1 (pool 1000) | 0.771 | 0.464 | 0.461 |
| MRR (pool 1000) | 0.833 | 0.569 | **0.561** |
| ROC AUC | *0.9967* (highest in the project!) | 0.982 | 0.964 |
| XA / XO / XM R@1 | 0.866 / 0.758 / 0.729 | | |

*Analysis*
* *In-dataset AUC is the project's highest (0.9967), but the main metric dropped*: sqlite3 MRR *0.685 → 0.561*, out-of-dataset R@1 — zlib *0.591 → 0.464*, sqlite3 *0.611 → 0.461*. That's an *18–25% relative drop on unseen libraries*, while the drop on openssl is minimal (0.783 → 0.771).
![alt text](./images/SAFE_rich_per_axis_bars.png)

* This is *an exact confirmation of both parts of the hypothesis*: rich tokens introduce *exactly the information that is tied to the architecture and the specific library*. eax only exists on x86; $t1 only on MIPS. A specific immediate (e.g., 0x67452301 — an MD5 constant) is unique to that particular openssl function, but it never appears in zlib.
* That's why: *the model gets stronger in-dataset* (it memorizes literals and register conventions → AUC 0.9967), *but weaker at generalization*. This is the classic *memorization vs. generalization* trade-off, though in a setting where it has a semantic cause.
* Most of the 93,926-token vocabulary is *rare* (long tail): many tokens appear only a few times in training, word2vec can't learn quality embeddings for them, and at test time they're either OOV or noise. Indirect confirmation of this: here **freeze=True beats `False`** (0.724 vs 0.711) — the opposite of SAFE-coarse. On a small vocabulary, fine-tuning improved the embeddings; on a rich one, fine-tuning causes overfitting on rare tokens, so it's better not to touch the embeddings at all.
* *plots/score_distribution*: on openssl, the separating boundary is sharper than any other model (almost two clean peaks). *But on zlib/sqlite3's distributions, the mass of positives noticeably "slides" toward the center* — i.e. on unseen libraries the model gives even true pairs a low score. This is exactly the visual signature of overfitting, and exactly why in-dataset AUC alone should never be trusted.
![alt text](./images/score_distribution_safe_rich.png)

*Conclusion:*
*The coarseness of normalization is the regulator that controls architectural invariance.* Deliberately removing information (registers, addresses, large constants) is *the key feature of this task*.


## 10. HBinSim (hierarchical model) - incomplete investigation

*Idea:* SAFE sees a function as a flat sequence and loses the graph; GNNs see the graph, but only hand-crafted features within a block. HBinSim combines both *hierarchically*:
1. *Block level:* the instructions of each basic block (coarse tokens, vocab 656) pass through a bi-GRU + attention → a learned block vector.
2. *Function level:* these learned block-vectors become the node features of the ACFG, on which a Structure2vec GNN runs.
That is, the 21 hand-crafted features per block are replaced by a *learned* block representation.

**Results (the only completed run: block_hidden=64, graph_hidden=128, T=5, msg_layers=2, bs=32, max_insns=50, max_blocks=100):**

| Metric | openssl | zlib | sqlite3 |
|---|---:|---:|---:|
| Recall@1 (pool 1000) | 0.602 | 0.484 | 0.409 |
| MRR (pool 1000) | 0.689 | 0.576 | **0.499** |
| ROC AUC | 0.988 | 0.970 | 0.943 |
| XA / XO / XM R@1 | 0.706 / 0.635 / 0.515 | | |
 
*Analysis (and limitations):*
* The result is *sharply weaker than SAFE* (sqlite3 MRR 0.499 vs 0.685; openssl R@1 0.602 vs 0.783) and also weaker than DeepSets (0.499 vs 0.546). But *this conclusion is about our resources, not about the model.* We only have *2 runs total*, and the second one crashed with OOM — meaning only one is actually complete.
![alt text](./images/Hbinsim_per_axis_bars.png)

* Why so few? HBinSim is the most expensive model: one batch is a B × N_blocks × L_insns tensor — i.e. 32 × 100 × 50 instruction tokens, and each of them has to pass through a bi-GRU. On a T4, this forced us into the following limits: batch_size=32 (instead of SAFE's 128!), max_insns=50, and max_blocks=100.
* Both limits *directly cut into the model*:
  * bs=32 is devastating for InfoNCE — there are only 31 negatives in a batch (instead of SAFE's 127), meaning the training signal weakens by an order of magnitude. We already measured the importance of this parameter on DeepSets, and the effect was large.
  * max_insns=50 and max_blocks=100 mean that *large functions get truncated* — exactly the functions that are the most discriminative.
* *plots/score_distribution*: a picture on par with DeepSets — bimodal, but with a noticeable "bridge" in the middle; the overlap grows even more out-of-dataset.
![alt text](./images/score_distribution_Hbinsin.png)

* *Conclusion:* HBinSim's idea (a learned block representation + graph) is conceptually the most complete, but we *couldn't fairly test it*. Its low score should be read as "insufficient budget," not "a bad model." If we continue the project, the first step will be gradient accumulation / mixed precision, to increase the effective batch size.
---
## 11. Comparison of All Models

Each model's *champion* run. The champion is selected the same way everywhere: *highest sqlite3 MRR pool=1000* (the main metric — the shaded column). openssl - test split, zlib/sqlite3 - out-of-dataset; all numbers pool=1000, 3000 queries (GMN - ~100 queries):

| # | Model | Representation | sqlite3 MRR ↑ | zlib MRR | sqlite3 R@1 | zlib R@1 | openssl R@1 | openssl MRR | XA | XO | XM | ROC AUC |
|---|---|---|---:|---:|---:|---:|---:|---:|---:|---:|---:|---:|
| 1 | Opcode Histogram | opcode counts | **0.093** | 0.123 | 0.083 | 0.109 | 0.105 | 0.116 | 0.000 | 0.365 | 0.000 | 0.549 |
| 2 | Feature Aggregator | hand feature (sum) | **0.204** | 0.243 | 0.148 | 0.184 | 0.220 | 0.269 | 0.132 | 0.536 | 0.085 | 0.887 |
| 3 | MLP Aggregator | hand feature + MLP | **0.430** | 0.514 | 0.348 | 0.421 | 0.535 | 0.613 | 0.664 | 0.571 | 0.432 | 0.973 |
| 4 | DeepSets | set of blocks | **0.546** | 0.593 | 0.455 | 0.497 | 0.725 | 0.793 | 0.826 | 0.709 | 0.680 | 0.993 |
| 5 | Structure2vec | ACFG (GNN) | **0.544** | 0.615 | 0.456 | 0.532 | 0.684 | 0.759 | 0.804 | 0.672 | 0.624 | 0.994 |
| 6 | GIN | ACFG (GNN) | **0.526** | 0.607 | 0.441 | 0.525 | 0.739 | 0.799 | 0.838 | 0.734 | 0.692 | 0.992 |
| 7 | GMN | ACFG + cross-attn | **~0.65**\* | ~0.63\* | ~0.59\* | ~0.54\* | ~0.73\* | ~0.76\* | - | - | - | - |
| 8 | *SAFE (coarse)* | *instructions, vocab 656* | ***0.685*** | *0.678* | *0.611* | *0.591* | *0.783* | *0.840* | *0.876* | *0.757* | *0.749* | 0.996 |
| 9 | SAFE-rich | instructions, vocab 93,926 | **0.561** | 0.569 | 0.461 | 0.464 | 0.771 | 0.833 | 0.866 | 0.758 | 0.729 | *0.9967* |
| 10 | HBinSim | hierarchical (insn → block → graph) | **0.65**\*\* | 0.70 | 0.58 | 0.62 | 0.79 | 0.841 | - | - | - | - |

\* GMN - measured on ~100 queries, with ±0.06 error; per-axis numbers are unreliable, ROC AUC wasn't logged (see §7).


### score_distribution - comparison table

<!-- insert screenshots pulled from W&B here -->
| Model | score_distribution |
|---|---|
| Baseline Opcode Histogram | images/score_distribution_histogram.png |
| Baseline feature agg | images/score_distribution_feature_agg.png |
| MLP Aggregator | images/score_distribution_mlp_agg.png |
| DeepSets | images/score_distribution_deepsets.png |
| Structure2vec | images/score_distribution_s2v.png |
| GIN | images/score_distribution_gin.png |
| GMN | images/score_distribution_gmn.png |
| SAFE | images/score_distribution_safe.png |
| SAFE-rich | images/score_distribution_safe_rich.png |
| Hbinsim | images/score_distribution_Hbinsin.png |
| SAFE-transformer | images/score_distribution_safe_transfromer.png |

Looking at these screenshots side by side ties the whole story together into one picture:
1. *Histogram*: the two distributions practically coincide (AUC ~0.5).
2. *Feature/MLP aggregator*: bimodality appears, but a thick "bridge" remains in the middle.
3. *DeepSets / GIN*: negatives compress toward 0, positives toward 1; the bridge thins.
4. *SAFE*: almost clean separation — *and, most importantly, the same picture holds on zlib/sqlite3 too*.
5. *SAFE-rich*: the sharpest separation on openssl, while out-of-dataset the mass of positives spreads back toward the center — the visual signature of overfitting.
---

## 12. Conclusions

*1. Representation beat architecture.*
On the main metric (sqlite3 MRR), even the best graph models don't reach a model that *doesn't use the graph at all*: S2V 0.544 and GIN 0.526 vs. DeepSets 0.546. Meanwhile, a simple bi-GRU on the right tokens (SAFE, 0.685) clearly beats all of them. On our task, control-flow topology has ~zero marginal value — *because it isn't itself invariant* to architecture and optimization (MIPS delay slots, O3's inlining/unrolling rewrite the CFG).

*2. The coarseness of normalization (reduction to a small sample) is the regulator of invariance.*
SAFE vs. SAFE-rich is the project's most similar experiment: only the tokenization changed. The rich vocabulary raised in-dataset AUC (0.9967 — the highest), but dropped out-of-dataset retrieval by 18–25% (sqlite3 MRR 0.685 → 0.561, zlib R@1 0.591 → 0.464). *Deliberately removing* information (registers, addresses, large constants) is a *feature, not a bug*, for this task.

*3. In-dataset AUC is a deceptive metric for this task.*
Starting from model 3 onward, openssl's ROC AUC is 0.97+ and varies by only *0.024* across the trained models — while the main metric (sqlite3 MRR) ranges from 0.43 to 0.69. Moreover, *the model with the highest AUC (SAFE-rich) falls sharply behind the champion on the main metric.* The only solid metric is *MRR/Recall on a large pool, on an unseen library — in our case, sqlite3 MRR pool=1000*.

*4. Compute resources shape conclusions.*
The low scores of GMN and HBinSim are *not a verdict on the models* — they're a result of the fact that neither could be fairly evaluated/trained on our T4s (cross-graph attention can't be cached; we were forced to run the hierarchical model at `bs=32`).

*5. Small engineering details produce large effects.*
The min_blocks=4 filter: +0.12 R@1. pool=mean_max_attention instead of a single mean: +0.15. Masked normalization on padded blocks (the BatchNorm bug): training stability. Batch size 32 → 128+ for InfoNCE: a qualitative difference. *Most of these bugs and details were caught by code review, not by metrics* — because even the buggy model "worked," just a bit worse.

*6. The data pipeline is half the project.*
angr's slow analysis, cross-compilation build systems, the IDA↔angr distribution shift, Kaggle session timeouts — these problems took up most of the time. Writing the model code, by comparison, went quickly.

---

## 13. Reproduction

# 1. Data generation (Docker - angr + cross-compilers)
`cd container && docker build -t binsim .`
`docker run -v $(pwd)/../data:/data binsim bash /data_compiled/compile.sh`   # 48 binaries
`docker run -v $(pwd)/../data:/data binsim python extract_acfg_angr.py`      # ACFG (21 features)
`docker run -v $(pwd)/../data:/data binsim python extract_insns_angr.py`     # coarse tokens (vocab 656)
`docker run -v $(pwd)/../data:/data binsim python extract_insns_rich_angr.py` # rich tokens (vocab 93,926)

# 2. Training - notebooks (Kaggle T4 / Colab; path autodetect + pickle cache)
#    every notebook contains a fixed eval harness (identical for every model)

W&B projects (all runs are public):
binsim_baseline_opcode-histogram · binsim_baseline_feature-aggregator · binsim_mlp-aggregator · binsim_deepsets · binsim_gnn · binsim_gmn · binsim_safe · binsim_safe_rich · binsim_hbinsim

---

## References

* Xu et al., Neural Network-based Graph Embedding for Cross-Platform Binary Code Similarity Detection (Gemini), CCS 2017 - [arxiv.org/pdf/1708.06525](https://arxiv.org/pdf/1708.06525)
* Massarelli et al., SAFE: Self-Attentive Function Embeddings for Binary Similarity, DIMVA 2019 - [arxiv.org/pdf/1811.05296](https://arxiv.org/pdf/1811.05296)
* Li et al., Graph Matching Networks for Learning the Similarity of Graph Structured Objects, ICML 2019 - [arxiv.org/pdf/1904.12787](https://arxiv.org/pdf/1904.12787)
* Xu et al., How Powerful are Graph Neural Networks? (GIN), ICLR 2019.
* Marcelli et al., How Machine Learning Is Solving the Binary Function Similarity Problem, USENIX Security 2022.
