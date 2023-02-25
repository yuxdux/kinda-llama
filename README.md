# KINDA-LLAMA
An open-source replication and extension of the [Meta AI's LLAMA](https://research.facebook.com/file/1574548786327032/LLaMA--Open-and-Efficient-Foundation-Language-Models.pdf) dataset. The project is general-purpose, but we also specifically aim for compatibility with [RWKV](https://github.com/BlinkDL/RWKV-LM) [checkpoints](https://huggingface.co/BlinkDL/rwkv-4-pile-14b/tree/main) and [tokenizer](https://github.com/BlinkDL/RWKV-LM/blob/main/RWKV-v4neo/20B_tokenizer.json).

## My overview on LLAMA dataset
... keeping in mind three possible goals, namely<br>
(G.1) pure replication of LLAMA<br>
(G.2) A superset of LLAMA for better scientific performance - they notice in the paper that they might lose to Minerva in the benchmarks due to not enough technical books and papers and<br>
(G.3) A "LLAMA minus Pile-V1" which would allow continuing training from the checkpoints we already have.<br>

Notably, (G.3) would require deduplication against Pile-V1 with some fast string-hashing algorithm ([Pile used simple sha256 hash](https://github.com/EleutherAI/the-pile/blob/master/processing_scripts/dedupe_train.py) but there might be better solutions)

Thus, the overview:

1. CommonCrawl is taken from [here](https://commoncrawl.org/the-data/get-started/) and [processed with this](https://github.com/facebookresearch/cc_net) (likely a decent amount of CPU compute needed, as it is written in python, we could profile and accelerate the hotspots).
2. [C4 is on huggingface](https://huggingface.co/datasets/allenai/c4) - we might think about which exact (noclean, clean) version to use.
3. Github - they used bigquery dump (it requires google account to access), we could use more extensive dumps such as https://huggingface.co/datasets/codeparrot/github-code (free) or https://huggingface.co/datasets/bigcode/the-stack (free, requires a form sign-in). Pile-V1 included 95 GiB Github section, so we need to deduplicate against it.
4. Wikipedia - they use latest summer 2022 dumps of Wikipedia in many languages(bg, ca, cs, da, de, en, es, fr, hr, hu, it, nl, pl, pt, ro, ru, sl, sr, sv, uk), while Pile-V1 used just an older dump of english wikipedia. They sample Wikipedia two times. Clearly this one is important enough to be shown several times, so I think we don't need deduplication here. The [dumps are available](https://dumps.wikimedia.org/backup-index.html) and we can process them with one of these scripts https://github.com/shyamupa/wikidump_preprocessing https://github.com/singletongue/wikipedia-utils https://github.com/siznax/wptools (if you know a better tool, comment).
5. Books - they use a mix of public domain books from Gutenberg project and a copy of Pile-V1 books section, thus we need to take a different set of quality books from libgen and clean+tokenize them with Pile-derived script to avoid duplication. They also implement deduplication at a book level with 90% threshold which wasn't the case with Pile. We will need fast custom code for this (comment if you know a good codebase to start from).
6. ArXiv - they use [ArXiv Latex dump](https://info.arxiv.org/help/bulk_data_s3.html) with extensive postprocessing (removal of intro pages and bibliography, **latex macro expansion**). It overlaps with Pile-V1 ArXiv subsection, but Pile-V1 lacks papers submitted in the last 3 years and it didn't use special preprocessing. Given success of Galactica LM with its multi-epoch training on scientific literature, we likely would be better served by avoiding deduplication here and just copying what LLAMA did for data processing. At a glance I don't see an exactly equivalent Arxiv script, so we might need to develop our own from one of these: https://github.com/EleutherAI/pile-arxiv https://github.com/mattbierbaum/arxiv-public-datasets https://github.com/amacfie/mathtext
7. Stackexchange - freely available from web archive. Pile-V1 has stackexchange data too, but LLAMA likely has a superset of it due to later date. LLAMA's preprocessing is very simple, could be implemented within [this codebase from Pile's authors](https://github.com/EleutherAI/stackexchange-dataset)

Important note: **in an attempt to enhance number representation, LLAMA authors split all numbers into individual digits**. We likely would be better off doing this as well, otherwise models hardly learn mathematics. This could be implemented without changing the legacy 20B_tokenizer.json to keep compatibility with available checkpoints.

Technically, the most complicated parts of the dataset are likely these: ArXiv, Wikipedia, Books. I expect some of the subdatasets to be very large and to have small compute hotspots we might want to rewrite in something other than interpreted python to execute it in time on volunteer hardware.

Regarding the storage requirements, we can use smart sharding to avoid having to store and transmit 3x data for different versions of the dataset. For example, the Pile-V1 is already sharded https://the-eye.eu/public/AI/pile/train/ and we might use similar data format with addition of labeling the shards as belonging to (G.1) (G.2) or (G.3) sets.

## Preliminary complexity estimate

In short, there are many pieces available to imitate and surpass LLAMA dataset, but there is no 100% complete workflow and some programming will be required to build it to completion, in addition to compute donation to execute it and produce the dataset.

## Possible extensions

If our goal were to surpass the LLAMA dataset, we might think about creating these addons:<br>
A.1. Add more scientific papers - pubmed and scihub (pubmed was included in Pile-V1 though)<br>
A.2. Add more science and engineering literature from libgen, OCR-ed if necessary. Some of it doesn't need OCR and could be quickly included with a pipeline similar to 5. ("Books")<br>
A.3. Add DM-math from Pile-V1<br>
A.4. Add more code from github - code modeling seems to help LMs.<br>
A.5. Add more dialogue data<br>
A.6. Add <work></work> tokens with reasoning chains like in [Galactica](https://arxiv.org/abs/2211.09085) (requires source of reasoning)<br>
