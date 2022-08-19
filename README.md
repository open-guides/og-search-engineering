# The Open Guide to Search Engineering


## Table of Contents

* [Introduction](#introduction)
* [Legend](#legend)
* [Why read this?](#why-read-this)
* [Principles](#principles)
* [Theory: The search problem](#theory-the-search-problem)
* [Theory: The search pipeline](#theory-the-search-pipeline)
* [Indexing pipeline operation](#indexing-pipeline-operation)
* [Serving systems](#serving-systems)
* [Quality, evaluation, and improvement](#quality-evaluation-and-improvement)
* [Practical advice](#practical-advice)
* [Services and tools](#services-and-tools)
* [Datasets](#datasets)
* [References](#references)
* [Credits](#credits)
* [License](#license)

## Introduction

**What every software engineer should know about search**

[This guide is based on [this](https://medium.com/startup-grind/what-every-software-engineer-should-know-about-search-27d1df99f80d) popular blog post, also discussed [on Hacker News](https://news.ycombinator.com/item?id=15231302).]

Ask a software engineer: “[How would you add search functionality to your product?](https://stackoverflow.com/questions/34314/how-do-i-implement-search-functionality-in-a-website)” or “[How do I build a search engine?](https://www.quora.com/How-to-build-a-search-engine-from-scratch)” You’ll probably immediately hear something like: “Oh, we’d just launch an ElasticSearch cluster. Search is easy these days.”

But is it? Numerous current products [still](https://github.com/isaacs/github/issues/908) [have](https://www.reddit.com/r/Windows10/comments/4jbxgo/can_we_talk_about_how_bad_windows_10_search_sucks/d365mce/) [suboptimal](https://www.reddit.com/r/spotify/comments/2apwpd/the_search_function_sucks_let_me_explain/) [search](https://medium.com/@RohitPaulK/github-issues-suck-723a5b80a1a3#.yp8ui3g9i) [experiences](https://thenextweb.com/opinion/2016/01/11/netflix-search-sucks-flixed-fixes-it/). Any true search expert will tell you that few engineers have a very deep understanding of how search engines work, knowledge that’s often needed to improve search quality.

Even though many open source software packages exist, and the research is vast, the knowledge around building solid search experiences is limited to a select few. Ironically, [searching online](https://www.google.com/search?q=building+a+search+engine) for search-related expertise doesn’t yield any recent, thoughtful overviews.

## Legend

* ❗ “Serious” gotcha: consequences of ignorance can be deadly
* 🔹 Especially notable idea or piece of technology
* ☁️ Cloud/SaaS
* 🍺 Open source / free software
* 🦏 JavaScript
* 🐍 Python
* ☕ Java
* 🇨 C/C++

## Why read this?

Think of this post as a collection of insights and resources that could help you to build search experiences. It can’t be a complete reference, of course, but hopefully we can improve it based on feedback (please comment or reach out!).

I’ll point at some of the most popular approaches, algorithms, techniques, and tools, based on my work on general purpose and niche search experiences of varying sizes at Google, Airbnb and several startups.

❗️ Not appreciating or understanding the scope and complexity of search problems can lead to bad user experiences, wasted engineering effort, and product failure.

If you’re impatient or already know a lot of this, you might find it useful to jump ahead to the [**services and tools**](#services-and-tools) section.

## Principles

Most of what we cover here has four underlying principles:

1. 🔹 **Search is an inherently messy problem:**

    * Queries are highly variable. The search problems are **highly variable** based on product needs.
    * Think about how different Facebook search (searching a graph of people).
    * YouTube search (searching individual videos).
    * Or how different both of those are are from Kayak ([air travel planning is a really hairy problem](http://www.demarcken.org/carl/papers/ITA-software-travel-complexity/ITA-software-travel-complexity.pdf)).
    * Google Maps (making sense of geo-spacial data).
    * Pinterest (pictures of a brunch you might cook one day).

2. **Quality, metrics, and processes matter a lot:**

    * There is no magic bullet (like PageRank) nor a magic ranking formula that makes for a good approach. Processes are always evolving collection of techniques and processes that solve aspects of the problem and improve overall experience, usually gradually and continuously.
    * ❗️In other words, search is not just just about building software that does **ranking** or **retrieval** (which we will discuss below) for a specific domain. Search systems are usually an evolving pipeline of components that are tuned and evolve over time and that build up to a cohesive experience.
    * In particular, the key to success in search is building processes for evaluation and tuning into the product and development cycles. A search system architect should **think about processes and metrics, not just technologies**.

3. **Use existing technologies first:**
    * As in most engineering problems, don’t reinvent the wheel yourself. When possible, use existing services or open source tools. If an existing SaaS (such as [Algolia](https://www.algolia.com/) or managed Elasticsearch) fits your constraints and you can afford to pay for it, use it. This solution will likely will be the best choice for your product at first, even if down the road you need to customize, enhance, or replace it.

4. **Even if you buy, know the details:**
    * ❗️ Even if you are using an existing open source or commercial solution, you should have some sense of the complexity of the search problem and where there are likely to be pitfalls.

## Theory: The search problem

Search is different for every product, and choices depend on many technical details of the requirements. It helps to identify the key parameters of your search problem:

* **Corpus size:** How big is the corpus (the complete set of documents that need to be searched)? Is it thousands or billions of documents?
8 **Media:** Are you searching through text, images, graphical relationships, or geospatial data?
* 🔹 **Corpus control and quality**: Are the sources for the documents under your control, or coming from a (potentially adversarial) third party? Are all the documents ready to be indexed or need to be cleaned up and selected?
* **Indexing speed:** Do you need real-time indexing, or is building indices in batch is fine?
* **Query language:** Are the queries structured, or you need to support unstructured ones?
* **Query structure**: Are your queries textual, images, sounds? Street addresses, record ids, people’s faces?
* **Context-dependence**: Do the results depend on who the user is, what is their history with the product, their geographical location, time of the day etc?
* **Suggest support**: Do you need to support incomplete queries?
* **Latency:** What are the serving latency requirements? 100 milliseconds or 100 seconds?
* **Access control:** Is it entirely public or should users only see a restricted subset of the documents?
* **Compliance:** Are there compliance or organizational limitations?
* **Internationalization:** Do you need to support documents with multilingual character sets or Unicode? Do you need to support a multilingual corpus? Multilingual queries?
    * 🔹 In general, use **[UTF-8](https://en.wikipedia.org/wiki/UTF-8)** unless you really know what you’re doing.

Thinking through these points up front can help you make significant choices designing and building individual search system components.

## Theory: The search pipeline

Now let’s go through a list of search sub-problems. These are usually solved by separate subsystems that form a pipeline. What that means is that a given subsystem consumes the output of previous subsystems, and produces input for the following subsystems.

This leads to an important property of the ecosystem: once you change how an upstream subsystem works, you need to evaluate the effect of the change and possibly change the behavior downstream.

Here are the most important problems you need to solve:

### Index selection

Given a set of documents (e.g. the entirety of the Internet, all the Twitter posts, all the pictures on Instagram), select a potentially smaller subset of documents that may be worthy for consideration as search results and only include those in the index, discarding the rest. This is done to keep your indexes compact, and is almost orthogonal to selecting the documents to show to the user. Examples of particular classes of documents that don’t make the cut may include:

### Spam

Oh, all the different shapes and sizes of search spam! A giant topic in itself, worthy of a separate guide. In the meantime, consult [this web spam taxonomy overview](http://airweb.cse.lehigh.edu/2005/gyongyi.pdf).

### Undesirable documents

Domain constraints might require filtering: [porn](https://www.researchgate.net/profile/Gabriel_Sanchez-Perez/publication/262371199_Explicit_image_detection_using_YCbCr_space_color_model_as_skin_detection/links/549839cf0cf2519f5a1dd966.pdf), illegal content, etc. The techniques are similar to spam filtering, probably with extra heuristics.

### Duplicates

Or near-duplicates and redundant documents. Can be done with [Locality-sensitive hashing](https://en.wikipedia.org/wiki/Locality-sensitive_hashing), [similarity measures](https://en.wikipedia.org/wiki/Similarity_measure), clustering techniques or even [clickthrough data](https://www.microsoft.com/en-us/research/wp-content/uploads/2011/02/RadlinskiBennettYilmaz_WSDM2011.pdf). A [good overview](http://infolab.stanford.edu/~ullman/mmds/ch3.pdf) of techniques.

### Low-utility documents

The definition of utility depends highly on the problem domain, so it’s hard to recommend the approaches here. Some ideas are: it might be possible to build a utility function for your documents; heuristics might work, or example an image that contains only black pixels is not a useful document; utility might be learned from user behavior.

### Index construction

For most search systems, document retrieval is performed using an [**inverted index**](https://en.wikipedia.org/wiki/Inverted_index) — often just called the **index.**

* The index is a mapping of **search terms** to documents. A search term could be a word, an image feature or any other document derivative useful for query-to-document matching. The list of the documents for a given term is called a [**posting list**](https://en.wikipedia.org/wiki/Inverted_index). It can be sorted by some metric, like document quality.
* Figure out whether you need to index the data in **batches** or in **real time**.
  * ❗️Many companies with large corpora of documents use a batch-oriented indexing approach, but then find this is unsuited to a product where users expect results to be current.
* With text documents, term extraction usually involves using NLP techniques, such as stop lists, [stemming](https://en.wikipedia.org/wiki/Stemming) and [entity extraction](https://en.wikipedia.org/wiki/Named-entity_recognition); for images or videos computer vision methods are used etc.
* In addition, documents are mined for statistical and meta information, such as references to other documents (used in the famous [**PageRank**](http://ilpubs.stanford.edu:8090/422/1/1999-66.pdf) ranking signal), [topics](https://gofishdigital.com/semantic-topic-modeling/), counts of term occurrences, document size, entities A mentioned etc. That information can be later used in ranking signal construction or document clustering. Some larger systems might contain several indexes, e.g. for documents of different types.
* Index formats. The actual structure and layout of the index is a complex topic, since it can be optimized in many ways. For instance there are [posting lists compression methods](https://nlp.stanford.edu/IR-book/html/htmledition/postings-file-compression-1.html), one could target [mmap()able data representation](https://deplinenoise.wordpress.com/2013/03/31/fast-mmapable-data-structures/) or use[ LSM-tree](https://en.wikipedia.org/wiki/Log-structured_merge-tree) for continuously updated index.

### Query analysis and document retrieval

Most popular search systems allow non-structured queries. That means the system has to extract structure out of the query itself. In the case of an inverted index, you need to extract search terms using [NLP](https://en.wikipedia.org/wiki/Natural_language_processing) techniques.

The extracted terms can be used to retrieve relevant documents. Unfortunately, most queries are not very well formulated, so it pays to do additional query expansion and rewriting, like:

* [Term re-weighting](http://orion.lcg.ufrj.br/Dr.Dobbs/books/book5/chap11.htm).
* [Spell checking](http://norvig.com/spell-correct.html). Historical query logs are very useful as a dictionary.
* [Synonym matching](http://nlp.stanford.edu/IR-book/html/htmledition/query-expansion-1.html). [Another survey](https://www.iro.umontreal.ca/~nie/IFT6255/carpineto-Survey-QE.pdf).
* [Named entity recognition](https://en.wikipedia.org/wiki/Named-entity_recognition). A good approach is to use [HMM-based language modeling](http://www.aclweb.org/anthology/P02-1060).
* Query classification. Detect queries of particular type. For example, Google Search detects queries that contain a geographical entity, a porny query, or a query about something in the news. The retrieval algorithm can then make a decision about which corpora or indexes to look at.
* Expansion through [personalization](https://en.wikipedia.org/wiki/Personalized_search) or [local context](http://searchengineland.com/future-search-engines-context-217550). Useful for queries like “gas stations around me”.

### Ranking

Given a list of documents (retrieved in the previous step), their signals, and a processed query, create an optimal ordering (ranking) for those documents.

Originally, most ranking models in use were hand-tuned weighted combinations of all the document signals. Signal sets might include PageRank, clickthrough data, topicality information and [others](http://backlinko.com/google-ranking-factors).

To further complicate things, many of those signals, such as PageRank, or ones generated by [statistical language models](http://times.cs.uiuc.edu/czhai/pub/slmir-now.pdf) contain parameters that greatly affect the performance of a signal. Those have to be hand-tuned too.

Lately, 🔹 [learning to rank](https://en.wikipedia.org/wiki/Learning_to_rank), signal-based discriminative supervised approaches are becoming more and more popular. Some popular examples of LtR are [McRank](https://papers.nips.cc/paper/3270-mcrank-learning-to-rank-using-multiple-classification-and-gradient-boosting.pdf) and [LambdaRank](https://www.microsoft.com/en-us/research/wp-content/uploads/2016/02/lambdarank.pdf) from Microsoft, and [MatrixNet](https://yandex.com/company/technologies/matrixnet/) from Yandex.

A new, [vector space based approach](https://arxiv.org/abs/1708.02702) for semantic retrieval and ranking is gaining popularity lately. The idea is to learn individual low-dimensional vector document representations, then build a model which maps queries into the same vector space.

Then, retrieval is just finding several documents that are closest by some metric (e.g. Euclidean distance) to the query vector. Ranking is the distance itself. If the mapping of both the documents and queries is built well, the documents are chosen not by a fact of presence of some simple pattern (like a word), but how close the documents are to the query by *meaning*.

## Indexing pipeline operation

Usually, each of the above pieces of the pipeline must be operated on a regular basis to keep the search index and search experience current.

❗️Operating a search pipeline can be complex and involve a lot of moving pieces. Not only is the data moving through the pipeline, but the code for each module and the formats and assumptions embedded in the data will change over time.

A pipeline can be run in “batch” or based on a regular or occasional basis (if indexing speed does not need to be real time) or in a streamed way (if real-time indexing is needed) or based on certain triggers.

Some complex search engines (like Google) have several layers of pipelines operating on different time scales — for example, a page that changes often (like [cnn.com](http://cnn.com/)) is indexed with a higher frequency than a static page that hasn’t changed in years.

## Serving systems

Ultimately, the goal of a search system is to accept queries, and use the index to return appropriately ranked results. While this subject can be incredibly complex and technical, we mention a few of the key aspects to this part of the system.

* **Performance:** users notice when the system they interact with is laggy. ❗️Google has done [extensive research](http://services.google.com/fh/files/blogs/google_delayexp.pdf), and they have noticed that number of searches falls 0.6%, when serving is slowed by 300ms. They recommend to serve results under 200 ms for most of your queries. A good article [on the topic](http://highscalability.com/latency-everywhere-and-it-costs-you-sales-how-crush-it). This is the hard part: the system needs to collect documents from, possibly, many computers, than merge them into possible a very long list and then sort that list in the ranking order. To complicate things further, ranking might be query-dependent, so, while sorting, the system is not just comparing 2 numbers, but performing computation.
* **🔹Caching results**: is often necessary to achieve decent performance. ❗️ But caches are just one large gotcha. The might show stale results when indices are updated or some results are blacklisted. Purging caches is a can of worms of itself: a search system might not have the capacity to serve the entire query stream with an empty (cold) cache, so the [cache needs to be pre-warmed](https://stackoverflow.com/questions/22756092/what-does-it-mean-by-cold-cache-and-warm-cache-concept) before the queries start arriving. Overall, caches complicate a system’s performance profile. Choosing a cache size and a replacement algorithm is also a [challenge](https://en.wikipedia.org/wiki/Cache_performance_measurement_and_metric).
* **Availability**: is often defined by an uptime/(uptime + downtime) metric. When the index is distributed, in order to serve any search results, the system often needs to query all the shards for their share of results. ❗️That means, that if one shard is unavailable, the entire search system is compromised. The more machines are involved in serving the index — the higher the probability of one of them becoming defunct and bringing the whole system down.
* **Managing multiple indices:** Indices for large systems may separated into shards (pieces) or divided by media type or indexing cadence (fresh versus long-term indices). Results can then be merged.
* **Merging results of different kinds**: e.g. Google showing results from Maps, News etc.

## Quality, evaluation, and improvement

So you’ve launched your indexing pipeline and search servers, and it’s all running nicely. Unfortunately the road to a solid search experience only begins with running infrastructure.

Next, you’ll need to build a set of processes around continuous search quality evaluation and improvement. In fact, this is actually most of the work and the hardest problem you’ll have to solve.

**🔹What is quality?** First, you’ll need to determine (and get your boss or the product lead to agree), what **quality** means in your case:

* Self-reported user satisfaction (includes UX)
* Perceived relevance of the returned results (not including UX)
* Satisfaction relative to competitors
* Satisfaction relative performance of the previous version of the search engine (e.g. last week)
* [User engagement](http://blog.popcornmetrics.com/5-user-engagement-metrics-for-growth/)

**Metrics:** Some of these concepts can be quite hard to quantify. On the other hand, it’s incredibly useful to be able to express how well a search engine is performing in a single number, a **quality metric**.

Continuously computing such a metric for your (and your competitors’) system you can both track your progress and explain how well you are doing to your boss. Here are some classical ways to quantify quality, that can help you construct your magic quality metric formula:

* [**Precision**](https://en.wikipedia.org/wiki/Information_retrieval#Precision) and  [**recall**](https://en.wikipedia.org/wiki/Information_retrieval#Recall) measure how well the retrieved set of documents corresponds to the set you expected to see.
* [**F score**](https://en.wikipedia.org/wiki/F1_score) (specifically **F1 score**) is a single number, that represents both precision and recall well.
* [**Mean Average Precision**](http://fastml.com/what-you-wanted-to-know-about-mean-average-precision/) (**MAP**) allows to quantify the relevance of the top returned results.
* [🔹**Normalized Discounted Cumulative Gain**](https://en.wikipedia.org/wiki/Discounted_cumulative_gain) (**nDCG**) is like MAP, but weights the relevance of the result by its position.
* [**Long and short clicks**](http://www.blindfiveyearold.com/short-clicks-versus-long-clicks) — Allow to quantify how useful the results are to the real users.
* [A good detailed overview](https://arxiv.org/pdf/1302.2318.pdf).

**🔹Human evaluations:** Quality metrics might seem like statistical calculations, but they can’t all be done by automated calculations. Ultimately, metrics need to represent subjective human evaluation, and this is where a “human in the loop” comes into play.

❗️Skipping human evaluation is probably the most widespread reason of sub-par search experiences.

Usually, at early stages the developers themselves evaluate the results manually. At later point [**human raters**](http://static.googleusercontent.com/media/www.google.com/en//insidesearch/howsearchworks/assets/searchqualityevaluatorguidelines.pdf) (or assessors) may get involved. Raters typically use custom tools to look at returned search results and provide feedback on the quality of the results.

Subsequently, you can use the feedback signals to guide development, help make launch decisions or even feed them back into the index selection, retrieval or ranking systems.

Here is the list of some other types of human-driven evaluation, that can be done on a search system:

* **Basic user evaluation:** The user ranks their satisfaction with the whole experience
* **Comparative evaluation:** Compare with other search results (compare with search results from earlier versions of the system or competitors)
* **Retrieval evaluation:** The query analysis and retrieval quality is often evaluated using manually constructed query-document sets. A user is shown a query and the list of the retrieved documents. She can then mark all the documents that are relevant to the query, and the ones that are not. The resulting pairs of (query, [relevant docs]) are called a “**golden set**”. Golden sets are remarkably useful. For one, an engineer can set up automatic retrieval regression tests using those sets. The selection signal from golden sets can also be fed back as ground truth to term re-weighting and other query re-writing models.
* **Ranking evaluation:** Raters are presented with a query and two documents side-by-side. The rater must choose the document that fits the query better. This creates a partial ordering on the documents for a given query. That ordering can be later be compared to the output of the ranking system. The usual ranking quality measures used are MAP and nDCG.

### Evaluation datasets

One should start thinking about the datasets used for evaluation (like “golden sets” mentioned above) early in the search experience design process. How you collect and update them? How you push them to the production eval pipeline? Is there a built-in bias?

**Live experiments:** After your search engine catches on and gains enough users, you might want to start conducting [live search experiments](https://googleblog.blogspot.co.uk/2008/08/search-experiments-large-and-small.html) on a portion of your traffic. The basic idea is to turn some optimization on for a group of people, and then compare the outcome with that of a “control” group — a similar sample of your users that did not have the experiment feature on for them. How you would measure the outcome is, once again, very product specific: it could be clicks on results, clicks on ads etc.

**Evaluation cycle time:** How fast you improve your search quality is directly related to how fast you can complete the above cycle of measurement and improvement. It is essential from the beginning to ask yourself, “how fast can we measure and improve our performance?”

Will it take days, hours, minutes or seconds to make changes and see if they improve quality? ❗️Running evaluation should also be as easy as possible for the engineers and should not take too much hands-on time.

## Practical advice

This guide is not meant as a tutorial, but here is a rough outline of a recommended approach to building a search experience right now:

* If you can afford it and it fits your needs, just buy an existing SaaS solution (some good ones are listed below). An existing service fits if:

  * Your experience is a “connected” one (your service or app has internet connection).
  * Does it support all the functionality you need out of box? This post gives a pretty good idea of what functions would you want. To name a few, I’d at least consider: support for the media you are searching; real-time indexing support; query flexibility, including context-dependent queries.
  * Given the size of the corpus and the expected [QpS](https://en.wikipedia.org/wiki/Queries_per_second), can you afford to pay for it for the next 12 months?
  * Can the service support your expected traffic within the required latency limits? In case when you are querying the service from an app, make sure that the given service is accessible quickly enough from where your users are.

* If a hosted solution does not fit your needs or resources, you probably want to use one of the open source libraries or tools. In case of connected apps or websites, I’d choose ElasticSearch right now. For embedded experiences, there are multiple tools below.

* You most likely want to do index selection and clean up your documents (say extract relevant text from HTML pages) before uploading them to the search index. This will decrease the index size and make getting to good results easier. If your corpus fits on a single machine, just write a script (or several) to do that. If not, consider [Spark](https://spark.apache.org/).


## Services and tools

### ☁️ SaaS

* ☁️ 🔹[**Algolia**](https://www.algolia.com/) — a proprietary SaaS that indexes a client’s website and provides an API to search the website’s pages. They also have an API to submit your own documents, support context dependent searches and serve results really fast. If I were building a web search experience right now and could afford it, I’d probably use Algolia first — and buy myself time to build a comparable search experience.

* Cloud-based **ElasticSearch** providers: ☁️ [**AWS ElasticSearch Cloud**](https://aws.amazon.com/elasticsearch-service/), ☁️[**elastic.co**](https://www.elastic.co/) and ☁️ [**Qbox**](https://qbox.io/).
* ☁️[**Azure Search**](https://azure.microsoft.com/en-us/services/search/) — a SaaS solution from Microsoft. Accessible through a REST API, it can scale to billions of documents. Has a Lucene query interface to simplify migrations from Lucene-based solutions.
* ☁️[**Swiftype**](https://swiftype.com/) — an enterprise SaaS that indexes your company’s internal services, like Salesforce, G Suite, Dropbox and the intranet site.

### Tools and libraries

* 🍺☕🔹[**Lucene**](https://lucene.apache.org/) is the most popular IR library. Implements query analysis, index retrieval and ranking. Either of the components can be replaced by an alternative implementation. There is also a C port — 🍺[Lucy](https://lucy.apache.org/).
* 🍺☕🔹[**Solr**](http://lucene.apache.org/solr/) is a complete search server, based on Lucene. It’s a part of the [Hadoop](http://hadoop.apache.org/) ecosystem of tools.
* 🍺☕🔹[**Hadoop**](http://hadoop.apache.org/) is the most widely used open source MapReduce system, originally designed as a indexing pipeline framework for Solr. It has been gradually loosing ground to 🍺[**Spark**](http://spark.apache.org/) as the batch data processing framework used for indexing. ☁️[EMR](https://aws.amazon.com/emr/) is a proprietary implementation of MapReduce on AWS.
* 🍺☕🔹 [**ElasticSearch**](https://www.elastic.co/products/elasticsearch) is also based on Lucene ([feature comparison with Solr](http://solr-vs-elasticsearch.com/)). It has been getting more attention lately, so much that a lot of people think of ES when they hear “search”, and for good reasons: it’s well supported, has [extensive API](https://www.elastic.co/guide/en/elasticsearch/reference/current/docs.html), [integrates with Hadoop](https://github.com/elastic/elasticsearch-hadoop) and [scales well](https://www.elastic.co/guide/en/elasticsearch/guide/current/distributed-cluster.html). There are open source and [Enterprise](https://www.elastic.co/cloud/enterprise) versions. ES is also available as a SaaS on Can scale to billions of documents, but scaling to that point can be very challenging, so typical scenario would involve orders of magnitude smaller corpus.
* 🍺🇨 [**Xapian**](https://xapian.org/) — a C++-based IR library. Relatively compact, so good for embedding into desktop or mobile applications.
* 🍺🇨 [**Sphinx**](http://sphinxsearch.com/) — an full-text search server. Has a SQL-like query language. Can also act as a [storage engine for MySQL](https://mariadb.com/kb/en/mariadb/sphinx-storage-engine/) or used as a library.
* 🍺☕ [**Nutch**](https://nutch.apache.org/) — a web crawler. Can be used in conjunction with Solr. It’s also the tool behind [🍺Common Crawl](http://commoncrawl.org/).
* 🍺🦏 [**Lunr**](https://lunrjs.com/) — a compact embedded search library for web apps on the client-side.
* 🍺🦏 [**searchkit**](https://github.com/searchkit/searchkit) — a library of web UI components to use with ElasticSearch.
* 🍺🦏 [**Norch**](https://github.com/fergiemcdowall/norch) — a [LevelDB](https://github.com/google/leveldb)-based search engine library for Node.js.
* 🍺🐍 [**Whoosh**](https://bitbucket.org/mchaput/whoosh/wiki/Home) — a fast, full-featured search library implemented in pure Python.
* OpenStreetMaps has its own 🍺[deck of search software](http://wiki.openstreetmap.org/wiki/Search_engines).

## Datasets

A few fun or useful data sets to try building a search engine or evaluating search engine quality:

* 🍺🔹 [**Commoncrawl**](http://commoncrawl.org/) — a regularly-updated open web crawl data. There is a [mirror on AWS](https://aws.amazon.com/public-datasets/common-crawl/), accessible for free within the service.
* 🍺🔹 [**Openstreetmap data dump**](http://wiki.openstreetmap.org/wiki/Downloading_data) is a very rich source of data for someone building a geospacial search engine.
* 🍺 [**Google Books N-grams**](http://commondatastorage.googleapis.com/books/syntactic-ngrams/index.html) can be very useful for building language models.
* 🍺 [**Wikipedia dumps**](https://dumps.wikimedia.org/) are a classic source to build, among other things, an entity graph out of. There is a [wide range of helper tools](https://www.mediawiki.org/wiki/Alternative_parsers) available.
* [**IMDb dumps**](http://www.imdb.com/interfaces) are a fun dataset to build a small toy search engine for.

## References

* [Modern Information Retrieval](https://www.amazon.com/dp/0321416910) by R. Baeza-Yates and B. Ribeiro-Neto is a good, deep academic treatment of the subject. This is a good overview for someone completely new to the topic.
* [Information Retrieval](https://www.amazon.com/dp/0262528878/) by S. Büttcher, C. Clarke and G. Cormack is another academic textbook with a wide coverage and is more up-to-date. Covers learn-to-rank and does a pretty good job at discussing theory of search systems evaluation. Also is a good overview.
* [Learning to Rank](https://www.amazon.com/dp/3642142664/) by T-Y Liu is a best theoretical treatment of LtR. Pretty thin on practical aspects though. Someone considering building an LtR system should probably check this out.
* [Managing Gigabytes](https://www.amazon.com/dp/1558605703) — published in 1999, is still a definitive reference for anyone embarking on building an efficient index of a significant size.
* [Text Retrieval and Search Engines](https://www.coursera.org/learn/text-retrieval) — a MOOC from Coursera. A decent overview of basics.
* [Indexing the World Wide Web: The Journey So Far](https://research.google.com/pubs/pub37043.html) ([PDF](https://pdfs.semanticscholar.org/28d8/288bff1b1fc693e6d80c238de9fe8b5e8160.pdf)), an overview of web search from 2012, by Ankit Jain and Abhishek Das of Google.
* [Why Writing Your Own Search Engine is Hard](http://queue.acm.org/detail.cfm?id=988407) a classic article from 2004 from Anna Patterson.
* https://github.com/harpribot/awesome-information-retrieval — a curated list of search-related resources.
* A [great blog](https://medium.com/@dtunkelang) on everything search by [Daniel Tunkelang](https://www.cs.cmu.edu/~quixote/).
* Some good slides on [search engine evaluation](https://web.stanford.edu/class/cs276/handouts/lecture8-evaluation_2014-one-per-page.pdf).
* UX article on [best practices for search](http://www.uxbooth.com/articles/best-practices-for-search/).

This concludes my humble attempt to make a somewhat-useful “map” for an aspiring search engine engineer. Did I miss something important? I’m pretty sure I did — you know, [the margin is too narrow](https://www.brainyquote.com/quotes/quotes/p/pierredefe204944.html) to contain this enormous topic. Let me know if you think that something should be here and is not — you can reach [me](https://www.linkedin.com/in/grigorev/) at[ forwidur@gmail.com](mailto:forwidur@gmail.com) or at [@forwidur](https://twitter.com/forwidur).

> P.S. — This post is part of a open, collaborative effort to build an online reference, the Open Guide to Practical AI, which we’ll release in draft form soon. See [this popular guide](https://github.com/open-guides/og-aws) for an example of what’s coming. If you’d like to get updates on or help with with this effort, sign up [here](https://upscri.be/d29cfe/).

## Credits

Original author: [**Max Grigorev**](https://twitter.com/forwidur)

Contribution and review:
[**Leo Polovets**](https://twitter.com/lpolovets), [**Abhishek Das**](https://www.linkedin.com/in/abhishek-das-3280053/)

Editor: 
[**Joshua Levy**](https://twitter.com/ojoshe)

## License

[![Creative Commons License](https://i.creativecommons.org/l/by-sa/4.0/88x31.png)](http://creativecommons.org/licenses/by-sa/4.0/)

This work is licensed under a [Creative Commons Attribution-ShareAlike 4.0 International License](http://creativecommons.org/licenses/by-sa/4.0/).
