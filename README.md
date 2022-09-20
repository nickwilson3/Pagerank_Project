# Pagerank Project

In this project, you will create a simple search engine for the website <https://www.lawfareblog.com>.
This website provides legal analysis on US national security issues.

**Due date:** Sunday, 18 September at midnight

**Computation:**
This project has low computational requirements.
You are not required to complete it on the lambda server (although you are welcome to if you'd like).

## Background

**Data:**

The `data` folder contains two files that store example "web graphs".
The file `small.csv.gz` contains the example graph from the *Deeper Inside Pagerank* paper.
This is a small graph, so we can manually inspect the contents of this file with the following command:
```
$ zcat data/small.csv.gz
source,target
1,2
1,3
3,1
3,2
3,5
4,5
4,6
5,6
5,4
6,4
```

> **Recall:**
> The `cat` terminal command outputs the contents of a file to stdout, and the `zcat` command first decompressed a gzipped file and then outputs the decompressed contents.

As you can see, the graph is stored as a CSV file.
The first line is a header,
and each subsequent line stores a single edge in the graph.
The first column contains the source node of the edge and the second column the target node.
The file is assumed to be sorted alphabetically.

The second data file `lawfareblog.csv.gz` contains the link structure for the lawfare blog.
Let's take a look at the first 10 of these lines:
```
$ zcat data/lawfareblog.csv.gz | head
source,target
www.lawfareblog.com/,www.lawfareblog.com/topic/interrogation
www.lawfareblog.com/,www.lawfareblog.com/upcoming-events
www.lawfareblog.com/,www.lawfareblog.com/
www.lawfareblog.com/,www.lawfareblog.com/our-comments-policy
www.lawfareblog.com/,www.lawfareblog.com/litigation-documents-related-appointment-matthew-whitaker-acting-attorney-general
www.lawfareblog.com/,www.lawfareblog.com/topic/lawfare-research-paper-series
www.lawfareblog.com/,www.lawfareblog.com/topic/book-reviews
www.lawfareblog.com/,www.lawfareblog.com/documents-related-mueller-investigation
www.lawfareblog.com/,www.lawfareblog.com/topic/international-law-loac
```
You can see that in this file, the node names are URLs.
Semantically, each line corresponds to an HTML `<a>` tag that is contained in the source webpage and links to the target webpage.

We can use the following command to count the total number of links in the file:
```
$ zcat data/lawfareblog.csv.gz | wc -l
1610789
```
Since every link corresponds to a non-zero entry in the `P` matrix,
this is also the value of `nnz(P)`.
(Technically, we should subtract 1 from this value since the `wc -l` command also counts the header line, not just the data lines.)

To get the dimensions of `P`, we need to count the total number of nodes in the graph.
The following command achieves this by: decompressing the file, extracting the first column, removing all duplicate lines, then counting the results.
```
$ zcat data/lawfareblog.csv.gz | cut -f1 -d, | uniq | wc -l
25761
```
This matrix is large enough that computing matrix products for dense matrices takes several minutes on a single CPU.
Fortunately, however, the matrix is very sparse.
The following python code computes the fraction of entries in the matrix with non-zero values:
```
>>> 1610788 / (25760**2)
0.0024274297384360172
```
Thus, by using sparse matrix operations, we will be able to speed up the code significantly.

**Code:**

The `pagerank.py` file contains code for loading the graph CSV files and searching through their nodes for key phrases.
For example, you can perform a search for all nodes (i.e. urls) that mention the string `corona` with the following command:
```
$ python3 pagerank.py --data=data/lawfareblog.csv.gz --verbose --search_query=corona
```

> **NOTE:**
> It will take about 10 seconds to load and parse the data files.
> All the other computation happens essentially instantly.

Currently, the pagerank of the nodes is not currently being calculated correctly, and so the webpages are returned in an arbitrary order.
Your task in this assignment will be to fix these calculations in order to have the most important results (i.e. highest pagerank results) returned first.

## Task 1: the power method

Implement the `WebGraph.power_method` function in `pagerank.py` for computing the pagerank vector by fixing the `FIXME` annotation.

**Part 1:**

To check that your implementation is working,
you should run the program on the `data/small.csv.gz` graph.
For my implementation, I get the following output.
```
$ python3 pagerank.py --data=data/small.csv.gz --verbose
DEBUG:root:computing indices
DEBUG:root:computing values
DEBUG:root:i=0 residual=2.5629e-01
DEBUG:root:i=1 residual=1.1841e-01
DEBUG:root:i=2 residual=7.0701e-02
DEBUG:root:i=3 residual=3.1815e-02
DEBUG:root:i=4 residual=2.0497e-02
DEBUG:root:i=5 residual=1.0108e-02
DEBUG:root:i=6 residual=6.3716e-03
DEBUG:root:i=7 residual=3.4228e-03
DEBUG:root:i=8 residual=2.0879e-03
DEBUG:root:i=9 residual=1.1750e-03
DEBUG:root:i=10 residual=7.0131e-04
DEBUG:root:i=11 residual=4.0321e-04
DEBUG:root:i=12 residual=2.3800e-04
DEBUG:root:i=13 residual=1.3812e-04
DEBUG:root:i=14 residual=8.1083e-05
DEBUG:root:i=15 residual=4.7251e-05
DEBUG:root:i=16 residual=2.7704e-05
DEBUG:root:i=17 residual=1.6164e-05
DEBUG:root:i=18 residual=9.4778e-06
DEBUG:root:i=19 residual=5.5066e-06
DEBUG:root:i=20 residual=3.2042e-06
DEBUG:root:i=21 residual=1.8612e-06
DEBUG:root:i=22 residual=1.1283e-06
DEBUG:root:i=23 residual=6.1907e-07
INFO:root:rank=0 pagerank=6.6270e-01 url=4
INFO:root:rank=1 pagerank=5.2179e-01 url=6
INFO:root:rank=2 pagerank=4.1434e-01 url=5
INFO:root:rank=3 pagerank=2.3175e-01 url=2
INFO:root:rank=4 pagerank=1.8590e-01 url=3
INFO:root:rank=5 pagerank=1.6917e-01 url=1
```
Yours likely won't be identical (due to weird floating point issues), but it should be similar.
In particular, the ranking of the nodes/urls should be the same order.

> **NOTE:**
> The `--verbose` flag causes all of the lines beginning with `DEBUG` to be printed.
> By default, only lines beginning with `INFO` are printed.

**Part 2:**

The `pagerank.py` file has an option `--search_query`, which takes a string as a parameter.
If this argument is used, then the program returns all nodes that match the query string sorted according to their pagerank.
Essentially, this gives us the most important pages related to our query.

Again, you may not get the exact same results as me,
but you should get similar results to the examples I've shown below.
Verify that you do in fact get similar results.

```
$ python3 pagerank.py --data=data/lawfareblog.csv.gz --search_query='corona'
INFO:root:rank=0 pagerank=1.0038e-03 url=www.lawfareblog.com/lawfare-podcast-united-nations-and-coronavirus-crisis
INFO:root:rank=1 pagerank=8.9224e-04 url=www.lawfareblog.com/house-oversight-committee-holds-day-two-hearing-government-coronavirus-response
INFO:root:rank=2 pagerank=7.0390e-04 url=www.lawfareblog.com/britains-coronavirus-response
INFO:root:rank=3 pagerank=6.9153e-04 url=www.lawfareblog.com/prosecuting-purposeful-coronavirus-exposure-terrorism
INFO:root:rank=4 pagerank=6.7041e-04 url=www.lawfareblog.com/israeli-emergency-regulations-location-tracking-coronavirus-carriers
INFO:root:rank=5 pagerank=6.6256e-04 url=www.lawfareblog.com/why-congress-conducting-business-usual-face-coronavirus
INFO:root:rank=6 pagerank=6.5046e-04 url=www.lawfareblog.com/congressional-homeland-security-committees-seek-ways-support-state-federal-responses-coronavirus
INFO:root:rank=7 pagerank=6.3620e-04 url=www.lawfareblog.com/paper-hearing-experts-debate-digital-contact-tracing-and-coronavirus-privacy-concerns
INFO:root:rank=8 pagerank=6.1248e-04 url=www.lawfareblog.com/house-subcommittee-voices-concerns-over-us-management-coronavirus
INFO:root:rank=9 pagerank=6.0187e-04 url=www.lawfareblog.com/livestream-house-oversight-committee-holds-hearing-government-coronavirus-response

$ python3 pagerank.py --data=data/lawfareblog.csv.gz --search_query='trump'
INFO:root:rank=0 pagerank=5.7826e-03 url=www.lawfareblog.com/trump-asks-supreme-court-stay-congressional-subpeona-tax-returns
INFO:root:rank=1 pagerank=5.2338e-03 url=www.lawfareblog.com/document-trump-revokes-obama-executive-order-counterterrorism-strike-casualty-reporting
INFO:root:rank=2 pagerank=5.1297e-03 url=www.lawfareblog.com/trump-administrations-worrying-new-policy-israeli-settlements
INFO:root:rank=3 pagerank=4.6599e-03 url=www.lawfareblog.com/dc-circuit-overrules-district-courts-due-process-ruling-qasim-v-trump
INFO:root:rank=4 pagerank=4.5934e-03 url=www.lawfareblog.com/donald-trump-and-politically-weaponized-executive-branch
INFO:root:rank=5 pagerank=4.3071e-03 url=www.lawfareblog.com/how-trumps-approach-middle-east-ignores-past-future-and-human-condition
INFO:root:rank=6 pagerank=4.0935e-03 url=www.lawfareblog.com/why-trump-cant-buy-greenland
INFO:root:rank=7 pagerank=3.7591e-03 url=www.lawfareblog.com/oral-argument-summary-qassim-v-trump
INFO:root:rank=8 pagerank=3.4509e-03 url=www.lawfareblog.com/dc-circuit-court-denies-trump-rehearing-mazars-case
INFO:root:rank=9 pagerank=3.4484e-03 url=www.lawfareblog.com/second-circuit-rules-mazars-must-hand-over-trump-tax-returns-new-york-prosecutors

$ python3 pagerank.py --data=data/lawfareblog.csv.gz --search_query='iran'
INFO:root:rank=0 pagerank=4.5746e-03 url=www.lawfareblog.com/praise-presidents-iran-tweets
INFO:root:rank=1 pagerank=4.4174e-03 url=www.lawfareblog.com/how-us-iran-tensions-could-disrupt-iraqs-fragile-peace
INFO:root:rank=2 pagerank=2.6928e-03 url=www.lawfareblog.com/cyber-command-operational-update-clarifying-june-2019-iran-operation
INFO:root:rank=3 pagerank=1.9391e-03 url=www.lawfareblog.com/aborted-iran-strike-fine-line-between-necessity-and-revenge
INFO:root:rank=4 pagerank=1.5452e-03 url=www.lawfareblog.com/parsing-state-departments-letter-use-force-against-iran
INFO:root:rank=5 pagerank=1.5357e-03 url=www.lawfareblog.com/iranian-hostage-crisis-and-its-effect-american-politics
INFO:root:rank=6 pagerank=1.5258e-03 url=www.lawfareblog.com/announcing-united-states-and-use-force-against-iran-new-lawfare-e-book
INFO:root:rank=7 pagerank=1.4221e-03 url=www.lawfareblog.com/us-names-iranian-revolutionary-guard-terrorist-organization-and-sanctions-international-criminal
INFO:root:rank=8 pagerank=1.1788e-03 url=www.lawfareblog.com/iran-shoots-down-us-drone-domestic-and-international-legal-implications
INFO:root:rank=9 pagerank=1.1463e-03 url=www.lawfareblog.com/israel-iran-syria-clash-and-law-use-force
```

**Part 3:**

The webgraph of lawfareblog.com (i.e. the `P` matrix) naturally contains a lot of structure.
For example, essentially all pages on the domain have links to the root page <https://lawfareblog.com/> and other "non-article" pages like <https://www.lawfareblog.com/topics> and <https://www.lawfareblog.com/subscribe-lawfare>.
These pages therefore have a large pagerank.
We can get a list of the pages with the largest pagerank by running

```
$ python3 pagerank.py --data=data/lawfareblog.csv.gz
INFO:root:rank=0 pagerank=2.8741e-01 url=www.lawfareblog.com/lawfare-job-board
INFO:root:rank=1 pagerank=2.8741e-01 url=www.lawfareblog.com/masthead
INFO:root:rank=2 pagerank=2.8741e-01 url=www.lawfareblog.com/litigation-documents-related-appointment-matthew-whitaker-acting-attorney-general
INFO:root:rank=3 pagerank=2.8741e-01 url=www.lawfareblog.com/documents-related-mueller-investigation
INFO:root:rank=4 pagerank=2.8741e-01 url=www.lawfareblog.com/topics
INFO:root:rank=5 pagerank=2.8741e-01 url=www.lawfareblog.com/about-lawfare-brief-history-term-and-site
INFO:root:rank=6 pagerank=2.8741e-01 url=www.lawfareblog.com/snowden-revelations
INFO:root:rank=7 pagerank=2.8741e-01 url=www.lawfareblog.com/support-lawfare
INFO:root:rank=8 pagerank=2.8741e-01 url=www.lawfareblog.com/upcoming-events
INFO:root:rank=9 pagerank=2.8741e-01 url=www.lawfareblog.com/our-comments-policy
```

Most of these pages are not very interesting, however, because they are not articles,
and usually when we are performing a web search, we only want articles.

This raises the question: How can we find the most important articles filtering out the non-article pages?
The answer is to modify the `P` matrix by removing all links to non-article pages.

This raises another question: How do we know if a link is a non-article page?
Unfortunately, this is a hard question to answer with 100% accuracy,
but there are many methods that get us most of the way there.
One easy to implement method is to compute what's called the "in-link ratio" of each node (i.e. the total number of edges with the node as a target divided by the total number of nodes),
and then remove nodes from the search results with too-high of a ratio.
The intuition is that non-article pages often appear in the menu of a webpage, and so have links from almost all of the other webpages;
but article-webpages are unlikely to appear on a menu and so will only have a small number of links from other webpages.
The `--filter_ratio` parameter causes the code to remove all pages that have an in-link ratio larger than the provided value.

Using this option, we can estimate the most important articles on the domain with the following command:
```
$ python3 pagerank.py --data=data/lawfareblog.csv.gz --filter_ratio=0.2
INFO:root:rank=0 pagerank=3.4696e-01 url=www.lawfareblog.com/trump-asks-supreme-court-stay-congressional-subpeona-tax-returns
INFO:root:rank=1 pagerank=2.9521e-01 url=www.lawfareblog.com/livestream-nov-21-impeachment-hearings-0
INFO:root:rank=2 pagerank=2.9040e-01 url=www.lawfareblog.com/opening-statement-david-holmes
INFO:root:rank=3 pagerank=1.5179e-01 url=www.lawfareblog.com/lawfare-podcast-ben-nimmo-whack-mole-game-disinformation
INFO:root:rank=4 pagerank=1.5099e-01 url=www.lawfareblog.com/todays-headlines-and-commentary-1963
INFO:root:rank=5 pagerank=1.5099e-01 url=www.lawfareblog.com/todays-headlines-and-commentary-1964
INFO:root:rank=6 pagerank=1.5071e-01 url=www.lawfareblog.com/lawfare-podcast-week-was-impeachment
INFO:root:rank=7 pagerank=1.4957e-01 url=www.lawfareblog.com/todays-headlines-and-commentary-1962
INFO:root:rank=8 pagerank=1.4367e-01 url=www.lawfareblog.com/cyberlaw-podcast-mistrusting-google
INFO:root:rank=9 pagerank=1.4240e-01 url=www.lawfareblog.com/lawfare-podcast-bonus-edition-gordon-sondland-vs-committee-no-bull
```
Notice that the urls in this list look much more like articles than the urls in the previous list.

When Google calculates their `P` matrix for the web,
they use a similar (but much more complicated) process to modify the `P` matrix in order to reduce spam results.
The exact formula they use is a jealously guarded secret that they update continuously.

In the case above, notice that we have accidentally removed the blog's most popular article (<www.lawfareblog.com/snowden-revelations>).
The blog editors believed that Snowden's revelations about NSA spying are so important that they directly put a link to the article on the menu.
So every single webpage in the domain links to the Snowden article,
and our "anti-spam" `--filter-ratio` argument removed this article from the list.
In general, it is a challenging open problem to remove spam from pagerank results,
and all current solutions rely on careful human tuning and still have lots of false positives and false negatives.

**Part 4:**

Recall from the reading that the runtime of pagerank depends heavily on the eigengap of the `\bar\bar P` matrix,
and that this eigengap is bounded by the alpha parameter.

Run the following four commands:
```
$ python3 pagerank.py --data=data/lawfareblog.csv.gz --verbose 
$ python3 pagerank.py --data=data/lawfareblog.csv.gz --verbose --alpha=0.99999
$ python3 pagerank.py --data=data/lawfareblog.csv.gz --verbose --filter_ratio=0.2
$ python3 pagerank.py --data=data/lawfareblog.csv.gz --verbose --filter_ratio=0.2 --alpha=0.99999
```
You should notice that the last command takes considerably more iterations to compute the pagerank vector.
(My code takes 685 iterations for this call, and about 10 iterations for all the others.)

This raises the question: Why does the second command (with the `--alpha` option but without the `--filter_ratio`) option not take a long time to run?
The answer is that the `P` graph for <https://www.lawfareblog.com> naturally has a large eigengap and so is fast to compute for all alpha values,
but the modified graph does not have a large eigengap and so requires a small alpha for fast convergence.

Changing the value of alpha also gives us very different pagerank rankings.
For example, 
```
$ python3 pagerank.py --data=data/lawfareblog.csv.gz --verbose --filter_ratio=0.2
INFO:root:rank=0 pagerank=3.4696e-01 url=www.lawfareblog.com/trump-asks-supreme-court-stay-congressional-subpeona-tax-returns
INFO:root:rank=1 pagerank=2.9521e-01 url=www.lawfareblog.com/livestream-nov-21-impeachment-hearings-0
INFO:root:rank=2 pagerank=2.9040e-01 url=www.lawfareblog.com/opening-statement-david-holmes
INFO:root:rank=3 pagerank=1.5179e-01 url=www.lawfareblog.com/lawfare-podcast-ben-nimmo-whack-mole-game-disinformation
INFO:root:rank=4 pagerank=1.5099e-01 url=www.lawfareblog.com/todays-headlines-and-commentary-1963
INFO:root:rank=5 pagerank=1.5099e-01 url=www.lawfareblog.com/todays-headlines-and-commentary-1964
INFO:root:rank=6 pagerank=1.5071e-01 url=www.lawfareblog.com/lawfare-podcast-week-was-impeachment
INFO:root:rank=7 pagerank=1.4957e-01 url=www.lawfareblog.com/todays-headlines-and-commentary-1962
INFO:root:rank=8 pagerank=1.4367e-01 url=www.lawfareblog.com/cyberlaw-podcast-mistrusting-google
INFO:root:rank=9 pagerank=1.4240e-01 url=www.lawfareblog.com/lawfare-podcast-bonus-edition-gordon-sondland-vs-committee-no-bull

$ python3 pagerank.py --data=data/lawfareblog.csv.gz --verbose --filter_ratio=0.2 --alpha=0.99999
INFO:root:rank=0 pagerank=7.0149e-01 url=www.lawfareblog.com/covid-19-speech-and-surveillance-response
INFO:root:rank=1 pagerank=7.0149e-01 url=www.lawfareblog.com/lawfare-live-covid-19-speech-and-surveillance
INFO:root:rank=2 pagerank=1.0552e-01 url=www.lawfareblog.com/cost-using-zero-days
INFO:root:rank=3 pagerank=3.1755e-02 url=www.lawfareblog.com/lawfare-podcast-former-congressman-brian-baird-and-daniel-schuman-how-congress-can-continue-function
INFO:root:rank=4 pagerank=2.2040e-02 url=www.lawfareblog.com/events
INFO:root:rank=5 pagerank=1.6027e-02 url=www.lawfareblog.com/water-wars-increased-us-focus-indo-pacific
INFO:root:rank=6 pagerank=1.6026e-02 url=www.lawfareblog.com/water-wars-drill-maybe-drill
INFO:root:rank=7 pagerank=1.6023e-02 url=www.lawfareblog.com/water-wars-disjointed-operations-south-china-sea
INFO:root:rank=8 pagerank=1.6020e-02 url=www.lawfareblog.com/water-wars-song-oil-and-fire
INFO:root:rank=9 pagerank=1.6020e-02 url=www.lawfareblog.com/water-wars-sinking-feeling-philippine-china-relations
```

Which of these rankings is better is entirely subjective,
and the only way to know if you have the "best" alpha for your application is to try several variations and see what is best.
If large alphas are good for your application, you can see that there is a trade-off between quality answers and algorithmic runtime.
We'll be exploring this trade-off more formally in class over the rest of the semester.

## Task 2: the personalization vector

The most interesting applications of pagerank involve the personalization vector.
Implement the `WebGraph.make_personalization_vector` function so that it outputs a personalization vector tuned for the input query.
The pseudocode for the function is:
```
for each index in the personalization vector:
    get the url for the index (see the _index_to_url function)
    check if the url satisfies the input query (see the url_satisfies_query function)
    if so, set the corresponding index to one
normalize the vector
```

**Part 1:**

The command line argument `--personalization_vector_query` will use the function you created above to augment your search with a custom personalization vector.
If you've implemented the function correctly,
you should get results similar to:
```
$ python3 pagerank.py --data=data/lawfareblog.csv.gz --filter_ratio=0.2 --personalization_vector_query='corona'
INFO:root:rank=0 pagerank=6.3127e-01 url=www.lawfareblog.com/covid-19-speech-and-surveillance-response
INFO:root:rank=1 pagerank=6.3124e-01 url=www.lawfareblog.com/lawfare-live-covid-19-speech-and-surveillance
INFO:root:rank=2 pagerank=1.5947e-01 url=www.lawfareblog.com/chinatalk-how-party-takes-its-propaganda-global
INFO:root:rank=3 pagerank=1.2209e-01 url=www.lawfareblog.com/brexit-not-immune-coronavirus
INFO:root:rank=4 pagerank=1.2209e-01 url=www.lawfareblog.com/rational-security-my-corona-edition
INFO:root:rank=5 pagerank=9.3360e-02 url=www.lawfareblog.com/trump-cant-reopen-country-over-state-objections
INFO:root:rank=6 pagerank=9.1920e-02 url=www.lawfareblog.com/prosecuting-purposeful-coronavirus-exposure-terrorism
INFO:root:rank=7 pagerank=9.1920e-02 url=www.lawfareblog.com/britains-coronavirus-response
INFO:root:rank=8 pagerank=7.7770e-02 url=www.lawfareblog.com/lawfare-podcast-united-nations-and-coronavirus-crisis
INFO:root:rank=9 pagerank=7.2888e-02 url=www.lawfareblog.com/house-oversight-committee-holds-day-two-hearing-government-coronavirus-response
```

Notice that these results are significantly different than when using the `--search_query` option:
```
$ python3 pagerank.py --data=data/lawfareblog.csv.gz --filter_ratio=0.2 --search_query='corona'
INFO:root:rank=0 pagerank=8.1320e-03 url=www.lawfareblog.com/house-oversight-committee-holds-day-two-hearing-government-coronavirus-response
INFO:root:rank=1 pagerank=7.7908e-03 url=www.lawfareblog.com/lawfare-podcast-united-nations-and-coronavirus-crisis
INFO:root:rank=2 pagerank=5.2262e-03 url=www.lawfareblog.com/livestream-house-oversight-committee-holds-hearing-government-coronavirus-response
INFO:root:rank=3 pagerank=3.9584e-03 url=www.lawfareblog.com/britains-coronavirus-response
INFO:root:rank=4 pagerank=3.8114e-03 url=www.lawfareblog.com/prosecuting-purposeful-coronavirus-exposure-terrorism
INFO:root:rank=5 pagerank=3.3973e-03 url=www.lawfareblog.com/paper-hearing-experts-debate-digital-contact-tracing-and-coronavirus-privacy-concerns
INFO:root:rank=6 pagerank=3.3633e-03 url=www.lawfareblog.com/cyberlaw-podcast-how-israel-fighting-coronavirus
INFO:root:rank=7 pagerank=3.3557e-03 url=www.lawfareblog.com/israeli-emergency-regulations-location-tracking-coronavirus-carriers
INFO:root:rank=8 pagerank=3.2160e-03 url=www.lawfareblog.com/congress-needs-coronavirus-failsafe-its-too-late
INFO:root:rank=9 pagerank=3.1036e-03 url=www.lawfareblog.com/why-congress-conducting-business-usual-face-coronavirus
```

Which results are better?
Again, that depends on what you mean by "better."
With the `--personalization_vector_query` option,
a webpage is important only if other coronavirus webpages also think it's important;
with the `--search_query` option,
a webpage is important if any other webpage thinks it's important.
You'll notice that in the later example, many of the webpages are about Congressional proceedings related to the coronavirus.
From a strictly coronavirus perspective, these are not very important webpages.
But in the broader context of national security, these are very important webpages.

Google engineers spend TONs of time fine-tuning their pagerank personalization vectors to remove spam webpages.
Exactly how they do this is another one of their secrets that they don't publicly talk about.

**Part 2:**

Another use of the `--personalization_vector_query` option is that we can find out what webpages are related to the coronavirus but don't directly mention the coronavirus.
This can be used to map out what types of topics are similar to the coronavirus.

For example, the following query ranks all webpages by their `corona` importance,
but removes webpages mentioning `corona` from the results.
```
$ python3 pagerank.py --data=data/lawfareblog.csv.gz --filter_ratio=0.2 --personalization_vector_query='corona' --search_query='-corona'
INFO:root:rank=0 pagerank=6.3127e-01 url=www.lawfareblog.com/covid-19-speech-and-surveillance-response
INFO:root:rank=1 pagerank=6.3124e-01 url=www.lawfareblog.com/lawfare-live-covid-19-speech-and-surveillance
INFO:root:rank=2 pagerank=1.5947e-01 url=www.lawfareblog.com/chinatalk-how-party-takes-its-propaganda-global
INFO:root:rank=3 pagerank=9.3360e-02 url=www.lawfareblog.com/trump-cant-reopen-country-over-state-objections
INFO:root:rank=4 pagerank=7.0277e-02 url=www.lawfareblog.com/fault-lines-foreign-policy-quarantined
INFO:root:rank=5 pagerank=6.9713e-02 url=www.lawfareblog.com/lawfare-podcast-mom-and-dad-talk-clinical-trials-pandemic
INFO:root:rank=6 pagerank=6.4944e-02 url=www.lawfareblog.com/limits-world-health-organization
INFO:root:rank=7 pagerank=5.9492e-02 url=www.lawfareblog.com/chinatalk-dispatches-shanghai-beijing-and-hong-kong
INFO:root:rank=8 pagerank=5.1245e-02 url=www.lawfareblog.com/us-moves-dismiss-case-against-company-linked-ira-troll-farm
INFO:root:rank=9 pagerank=5.1245e-02 url=www.lawfareblog.com/livestream-house-armed-services-holds-hearing-national-security-challenges-north-and-south-america
```
You can see that there are many urls about concepts that are obviously related like "covid", "clinical trials", and "quarantine",
but this algorithm also finds articles about Chinese propaganda and Trump's policy decisions.
Both of these articles are highly relevant to coronavirus discussions,
but a simple keyword search for corona or related terms would not find these articles.

<!--
**Part 3:**

Select another topic related to national security.
You should experiment with a national security topic other than the coronavirus.
For example, find out what articles are important to the `iran` topic but do not contain the word `iran`.
Your goal should be to discover what topics that www.lawfareblog.com considers to be related to the national security topic you choose.
-->

## Submission

1. Create a new repo on github (not a fork of this repo).

1. Run the following commands, and paste their output into the code blocks below.
   
   Task 1, part 1:
   ```
   $ python3 pagerank.py --data=data/small.csv.gz --verbose
   
   DEBUG:root:computing indices
DEBUG:root:computing values
DEBUG:root:i=0 residual=0.2562914192676544
DEBUG:root:i=1 residual=0.11841227114200592
DEBUG:root:i=2 residual=0.07070134580135345
DEBUG:root:i=3 residual=0.03181542828679085
DEBUG:root:i=4 residual=0.020496590062975883
DEBUG:root:i=5 residual=0.01010835450142622
DEBUG:root:i=6 residual=0.006371544674038887
DEBUG:root:i=7 residual=0.0034227892756462097
DEBUG:root:i=8 residual=0.002087961183860898
DEBUG:root:i=9 residual=0.0011749734403565526
DEBUG:root:i=10 residual=0.0007012754795141518
DEBUG:root:i=11 residual=0.00040320929838344455
DEBUG:root:i=12 residual=0.00023798426263965666
DEBUG:root:i=13 residual=0.00013812065299134701
DEBUG:root:i=14 residual=8.108324982458726e-05
DEBUG:root:i=15 residual=4.7269360948121175e-05
DEBUG:root:i=16 residual=2.7704918466042727e-05
DEBUG:root:i=17 residual=1.6170568414963782e-05
DEBUG:root:i=18 residual=9.479118489252869e-06
DEBUG:root:i=19 residual=5.4782999541203026e-06
DEBUG:root:i=20 residual=3.2123323308042018e-06
DEBUG:root:i=21 residual=1.8802053318722756e-06
DEBUG:root:i=22 residual=1.1228398761886638e-06
DEBUG:root:i=23 residual=6.322027275018627e-07
INFO:root:rank=0 pagerank=6.6270e-01 url=4
INFO:root:rank=1 pagerank=5.2179e-01 url=6
INFO:root:rank=2 pagerank=4.1434e-01 url=5
INFO:root:rank=3 pagerank=2.3175e-01 url=2
INFO:root:rank=4 pagerank=1.8590e-01 url=3
INFO:root:rank=5 pagerank=1.6917e-01 url=1

   ```

   Task 1, part 2:
   ```
   $ python3 pagerank.py --data=data/lawfareblog.csv.gz --search_query='corona'
   
   INFO:root:rank=0 pagerank=1.0038e-03 url=www.lawfareblog.com/lawfare-podcast-united-nations-and-coronavirus-crisis
INFO:root:rank=1 pagerank=8.9228e-04 url=www.lawfareblog.com/house-oversight-committee-holds-day-two-hearing-government-coronavirus-response
INFO:root:rank=2 pagerank=7.0394e-04 url=www.lawfareblog.com/britains-coronavirus-response
INFO:root:rank=3 pagerank=6.9157e-04 url=www.lawfareblog.com/prosecuting-purposeful-coronavirus-exposure-terrorism
INFO:root:rank=4 pagerank=6.7045e-04 url=www.lawfareblog.com/israeli-emergency-regulations-location-tracking-coronavirus-carriers
INFO:root:rank=5 pagerank=6.6260e-04 url=www.lawfareblog.com/why-congress-conducting-business-usual-face-coronavirus
INFO:root:rank=6 pagerank=6.5050e-04 url=www.lawfareblog.com/congressional-homeland-security-committees-seek-ways-support-state-federal-responses-coronavirus
INFO:root:rank=7 pagerank=6.3623e-04 url=www.lawfareblog.com/paper-hearing-experts-debate-digital-contact-tracing-and-coronavirus-privacy-concerns
INFO:root:rank=8 pagerank=6.1252e-04 url=www.lawfareblog.com/house-subcommittee-voices-concerns-over-us-management-coronavirus
INFO:root:rank=9 pagerank=6.0191e-04 url=www.lawfareblog.com/livestream-house-oversight-committee-holds-hearing-government-coronavirus-response


   $ python3 pagerank.py --data=data/lawfareblog.csv.gz --search_query='trump'
   
   INFO:root:rank=0 pagerank=5.7827e-03 url=www.lawfareblog.com/trump-asks-supreme-court-stay-congressional-subpeona-tax-returns
INFO:root:rank=1 pagerank=5.2340e-03 url=www.lawfareblog.com/document-trump-revokes-obama-executive-order-counterterrorism-strike-casualty-reporting
INFO:root:rank=2 pagerank=5.1298e-03 url=www.lawfareblog.com/trump-administrations-worrying-new-policy-israeli-settlements
INFO:root:rank=3 pagerank=4.6601e-03 url=www.lawfareblog.com/dc-circuit-overrules-district-courts-due-process-ruling-qasim-v-trump
INFO:root:rank=4 pagerank=4.5935e-03 url=www.lawfareblog.com/donald-trump-and-politically-weaponized-executive-branch
INFO:root:rank=5 pagerank=4.3073e-03 url=www.lawfareblog.com/how-trumps-approach-middle-east-ignores-past-future-and-human-condition
INFO:root:rank=6 pagerank=4.0936e-03 url=www.lawfareblog.com/why-trump-cant-buy-greenland
INFO:root:rank=7 pagerank=3.7592e-03 url=www.lawfareblog.com/oral-argument-summary-qassim-v-trump
INFO:root:rank=8 pagerank=3.4510e-03 url=www.lawfareblog.com/dc-circuit-court-denies-trump-rehearing-mazars-case
INFO:root:rank=9 pagerank=3.4486e-03 url=www.lawfareblog.com/second-circuit-rules-mazars-must-hand-over-trump-tax-returns-new-york-prosecutors


   $ python3 pagerank.py --data=data/lawfareblog.csv.gz --search_query='iran'
   
   INFO:root:rank=0 pagerank=4.5748e-03 url=www.lawfareblog.com/praise-presidents-iran-tweets
INFO:root:rank=1 pagerank=4.4176e-03 url=www.lawfareblog.com/how-us-iran-tensions-could-disrupt-iraqs-fragile-peace
INFO:root:rank=2 pagerank=2.6929e-03 url=www.lawfareblog.com/cyber-command-operational-update-clarifying-june-2019-iran-operation
INFO:root:rank=3 pagerank=1.9392e-03 url=www.lawfareblog.com/aborted-iran-strike-fine-line-between-necessity-and-revenge
INFO:root:rank=4 pagerank=1.5453e-03 url=www.lawfareblog.com/parsing-state-departments-letter-use-force-against-iran
INFO:root:rank=5 pagerank=1.5358e-03 url=www.lawfareblog.com/iranian-hostage-crisis-and-its-effect-american-politics
INFO:root:rank=6 pagerank=1.5258e-03 url=www.lawfareblog.com/announcing-united-states-and-use-force-against-iran-new-lawfare-e-book
INFO:root:rank=7 pagerank=1.4222e-03 url=www.lawfareblog.com/us-names-iranian-revolutionary-guard-terrorist-organization-and-sanctions-international-criminal
INFO:root:rank=8 pagerank=1.1788e-03 url=www.lawfareblog.com/iran-shoots-down-us-drone-domestic-and-international-legal-implications
INFO:root:rank=9 pagerank=1.1463e-03 url=www.lawfareblog.com/israel-iran-syria-clash-and-law-use-force
   ```

   Task 1, part 3:
   ```
   $ python3 pagerank.py --data=data/lawfareblog.csv.gz
   
   INFO:root:rank=0 pagerank=2.8741e-01 url=www.lawfareblog.com/about-lawfare-brief-history-term-and-site
INFO:root:rank=1 pagerank=2.8741e-01 url=www.lawfareblog.com/lawfare-job-board
INFO:root:rank=2 pagerank=2.8741e-01 url=www.lawfareblog.com/masthead
INFO:root:rank=3 pagerank=2.8741e-01 url=www.lawfareblog.com/litigation-documents-resources-related-travel-ban
INFO:root:rank=4 pagerank=2.8741e-01 url=www.lawfareblog.com/subscribe-lawfare
INFO:root:rank=5 pagerank=2.8741e-01 url=www.lawfareblog.com/litigation-documents-related-appointment-matthew-whitaker-acting-attorney-general
INFO:root:rank=6 pagerank=2.8741e-01 url=www.lawfareblog.com/documents-related-mueller-investigation
INFO:root:rank=7 pagerank=2.8741e-01 url=www.lawfareblog.com/our-comments-policy
INFO:root:rank=8 pagerank=2.8741e-01 url=www.lawfareblog.com/upcoming-events
INFO:root:rank=9 pagerank=2.8741e-01 url=www.lawfareblog.com/topics

   $ python3 pagerank.py --data=data/lawfareblog.csv.gz --filter_ratio=0.2
   
   INFO:root:rank=0 pagerank=3.4697e-01 url=www.lawfareblog.com/trump-asks-supreme-court-stay-congressional-subpeona-tax-returns
INFO:root:rank=1 pagerank=2.9522e-01 url=www.lawfareblog.com/livestream-nov-21-impeachment-hearings-0
INFO:root:rank=2 pagerank=2.9040e-01 url=www.lawfareblog.com/opening-statement-david-holmes
INFO:root:rank=3 pagerank=1.5179e-01 url=www.lawfareblog.com/lawfare-podcast-ben-nimmo-whack-mole-game-disinformation
INFO:root:rank=4 pagerank=1.5100e-01 url=www.lawfareblog.com/todays-headlines-and-commentary-1964
INFO:root:rank=5 pagerank=1.5100e-01 url=www.lawfareblog.com/todays-headlines-and-commentary-1963
INFO:root:rank=6 pagerank=1.5072e-01 url=www.lawfareblog.com/lawfare-podcast-week-was-impeachment
INFO:root:rank=7 pagerank=1.4958e-01 url=www.lawfareblog.com/todays-headlines-and-commentary-1962
INFO:root:rank=8 pagerank=1.4367e-01 url=www.lawfareblog.com/cyberlaw-podcast-mistrusting-google
INFO:root:rank=9 pagerank=1.4240e-01 url=www.lawfareblog.com/lawfare-podcast-bonus-edition-gordon-sondland-vs-committee-no-bull

   ```

   Task 1, part 4:
   ```
   $ python3 pagerank.py --data=data/lawfareblog.csv.gz --verbose 
   DEBUG:root:computing indices
DEBUG:root:computing values
DEBUG:root:i=0 residual=1.3793749809265137
DEBUG:root:i=1 residual=0.11642683297395706
DEBUG:root:i=2 residual=0.07496178895235062
DEBUG:root:i=3 residual=0.031702104955911636
DEBUG:root:i=4 residual=0.0174466110765934
DEBUG:root:i=5 residual=0.008526231162250042
DEBUG:root:i=6 residual=0.00444182800129056
DEBUG:root:i=7 residual=0.0022433267440646887
DEBUG:root:i=8 residual=0.001149666146375239
DEBUG:root:i=9 residual=0.0005811726441606879
DEBUG:root:i=10 residual=0.00029266104684211314
DEBUG:root:i=11 residual=0.00014553753135260195
DEBUG:root:i=12 residual=7.151532918214798e-05
DEBUG:root:i=13 residual=3.476878555375151e-05
DEBUG:root:i=14 residual=1.5952729881973937e-05
DEBUG:root:i=15 residual=6.454707545344718e-06
DEBUG:root:i=16 residual=2.470087110850727e-06
DEBUG:root:i=17 residual=8.236708595177333e-07
INFO:root:rank=0 pagerank=2.8741e-01 url=www.lawfareblog.com/about-lawfare-brief-history-term-and-site
INFO:root:rank=1 pagerank=2.8741e-01 url=www.lawfareblog.com/lawfare-job-board
INFO:root:rank=2 pagerank=2.8741e-01 url=www.lawfareblog.com/masthead
INFO:root:rank=3 pagerank=2.8741e-01 url=www.lawfareblog.com/litigation-documents-resources-related-travel-ban
INFO:root:rank=4 pagerank=2.8741e-01 url=www.lawfareblog.com/subscribe-lawfare
INFO:root:rank=5 pagerank=2.8741e-01 url=www.lawfareblog.com/litigation-documents-related-appointment-matthew-whitaker-acting-attorney-general
INFO:root:rank=6 pagerank=2.8741e-01 url=www.lawfareblog.com/documents-related-mueller-investigation
INFO:root:rank=7 pagerank=2.8741e-01 url=www.lawfareblog.com/our-comments-policy
INFO:root:rank=8 pagerank=2.8741e-01 url=www.lawfareblog.com/upcoming-events
INFO:root:rank=9 pagerank=2.8741e-01 url=www.lawfareblog.com/topics

   $ python3 pagerank.py --data=data/lawfareblog.csv.gz --verbose --alpha=0.99999
   
   DEBUG:root:computing indices
DEBUG:root:computing values
DEBUG:root:i=0 residual=1.3846429586410522
DEBUG:root:i=1 residual=0.07087274640798569
DEBUG:root:i=2 residual=0.018819572404026985
DEBUG:root:i=3 residual=0.0069561367854475975
DEBUG:root:i=4 residual=0.002734419424086809
DEBUG:root:i=5 residual=0.0010338547872379422
DEBUG:root:i=6 residual=0.00037713360507041216
DEBUG:root:i=7 residual=0.00013517311890609562
DEBUG:root:i=8 residual=4.816500950255431e-05
DEBUG:root:i=9 residual=1.7141897842520848e-05
DEBUG:root:i=10 residual=6.1014479797449894e-06
DEBUG:root:i=11 residual=2.172939502997906e-06
DEBUG:root:i=12 residual=7.724702300038189e-07
INFO:root:rank=0 pagerank=2.8859e-01 url=www.lawfareblog.com/litigation-documents-related-appointment-matthew-whitaker-acting-attorney-general
INFO:root:rank=1 pagerank=2.8859e-01 url=www.lawfareblog.com/lawfare-job-board
INFO:root:rank=2 pagerank=2.8859e-01 url=www.lawfareblog.com/documents-related-mueller-investigation
INFO:root:rank=3 pagerank=2.8859e-01 url=www.lawfareblog.com/litigation-documents-resources-related-travel-ban
INFO:root:rank=4 pagerank=2.8859e-01 url=www.lawfareblog.com/subscribe-lawfare
INFO:root:rank=5 pagerank=2.8859e-01 url=www.lawfareblog.com/topics
INFO:root:rank=6 pagerank=2.8859e-01 url=www.lawfareblog.com/about-lawfare-brief-history-term-and-site
INFO:root:rank=7 pagerank=2.8859e-01 url=www.lawfareblog.com/our-comments-policy
INFO:root:rank=8 pagerank=2.8859e-01 url=www.lawfareblog.com/upcoming-events
INFO:root:rank=9 pagerank=2.8859e-01 url=www.lawfareblog.com/masthead
   
   $ python3 pagerank.py --data=data/lawfareblog.csv.gz --verbose --filter_ratio=0.2
   
   DEBUG:root:computing indices
DEBUG:root:computing values
DEBUG:root:i=0 residual=1.2609769105911255
DEBUG:root:i=1 residual=0.4985710680484772
DEBUG:root:i=2 residual=0.13418613374233246
DEBUG:root:i=3 residual=0.0692228302359581
DEBUG:root:i=4 residual=0.023409796878695488
DEBUG:root:i=5 residual=0.010187179781496525
DEBUG:root:i=6 residual=0.00490697892382741
DEBUG:root:i=7 residual=0.002280232962220907
DEBUG:root:i=8 residual=0.0010745070176199079
DEBUG:root:i=9 residual=0.0005251269903965294
DEBUG:root:i=10 residual=0.00026976881781592965
DEBUG:root:i=11 residual=0.00014569450286217034
DEBUG:root:i=12 residual=8.226578938774765e-05
DEBUG:root:i=13 residual=4.813347550225444e-05
DEBUG:root:i=14 residual=2.8801283406210132e-05
DEBUG:root:i=15 residual=1.7420436051907018e-05
DEBUG:root:i=16 residual=1.0539955837884918e-05
DEBUG:root:i=17 residual=6.396006028808188e-06
DEBUG:root:i=18 residual=3.848239430226386e-06
DEBUG:root:i=19 residual=2.298697609148803e-06
DEBUG:root:i=20 residual=1.3677324659511214e-06
DEBUG:root:i=21 residual=8.154062811627227e-07
INFO:root:rank=0 pagerank=3.4697e-01 url=www.lawfareblog.com/trump-asks-supreme-court-stay-congressional-subpeona-tax-returns
INFO:root:rank=1 pagerank=2.9522e-01 url=www.lawfareblog.com/livestream-nov-21-impeachment-hearings-0
INFO:root:rank=2 pagerank=2.9040e-01 url=www.lawfareblog.com/opening-statement-david-holmes
INFO:root:rank=3 pagerank=1.5179e-01 url=www.lawfareblog.com/lawfare-podcast-ben-nimmo-whack-mole-game-disinformation
INFO:root:rank=4 pagerank=1.5100e-01 url=www.lawfareblog.com/todays-headlines-and-commentary-1964
INFO:root:rank=5 pagerank=1.5100e-01 url=www.lawfareblog.com/todays-headlines-and-commentary-1963
INFO:root:rank=6 pagerank=1.5072e-01 url=www.lawfareblog.com/lawfare-podcast-week-was-impeachment
INFO:root:rank=7 pagerank=1.4958e-01 url=www.lawfareblog.com/todays-headlines-and-commentary-1962
INFO:root:rank=8 pagerank=1.4367e-01 url=www.lawfareblog.com/cyberlaw-podcast-mistrusting-google
INFO:root:rank=9 pagerank=1.4240e-01 url=www.lawfareblog.com/lawfare-podcast-bonus-edition-gordon-sondland-vs-committee-no-bull
   
   
   $ python3 pagerank.py --data=data/lawfareblog.csv.gz --verbose --filter_ratio=0.2 --alpha=0.99999
   
   DEBUG:root:computing indices
DEBUG:root:computing values
DEBUG:root:i=0 residual=1.2809287309646606
DEBUG:root:i=1 residual=0.563285768032074
DEBUG:root:i=2 residual=0.3834463059902191
DEBUG:root:i=3 residual=0.2186862975358963
DEBUG:root:i=4 residual=0.1411007195711136
DEBUG:root:i=5 residual=0.10877297818660736
DEBUG:root:i=6 residual=0.0929727703332901
DEBUG:root:i=7 residual=0.08236368745565414
DEBUG:root:i=8 residual=0.07349645346403122
DEBUG:root:i=9 residual=0.06571436673402786
DEBUG:root:i=10 residual=0.05918193981051445
DEBUG:root:i=11 residual=0.05423394963145256
DEBUG:root:i=12 residual=0.05114201083779335
DEBUG:root:i=13 residual=0.049983762204647064
DEBUG:root:i=14 residual=0.05056798458099365
DEBUG:root:i=15 residual=0.05246279016137123
DEBUG:root:i=16 residual=0.0551128052175045
DEBUG:root:i=17 residual=0.05795999616384506
DEBUG:root:i=18 residual=0.06051959842443466
DEBUG:root:i=19 residual=0.06241799145936966
DEBUG:root:i=20 residual=0.0634097307920456
DEBUG:root:i=21 residual=0.06338091194629669
DEBUG:root:i=22 residual=0.062340348958969116
DEBUG:root:i=23 residual=0.06039580702781677
DEBUG:root:i=24 residual=0.05772067606449127
DEBUG:root:i=25 residual=0.05451766029000282
DEBUG:root:i=26 residual=0.05098824203014374
DEBUG:root:i=27 residual=0.047311000525951385
DEBUG:root:i=28 residual=0.04363010451197624
DEBUG:root:i=29 residual=0.04005277529358864
DEBUG:root:i=30 residual=0.03665163740515709
DEBUG:root:i=31 residual=0.03347083553671837
DEBUG:root:i=32 residual=0.030532417818903923
DEBUG:root:i=33 residual=0.027842501178383827
DEBUG:root:i=34 residual=0.025396129116415977
DEBUG:root:i=35 residual=0.02318161353468895
DEBUG:root:i=36 residual=0.021183300763368607
DEBUG:root:i=37 residual=0.019383732229471207
DEBUG:root:i=38 residual=0.017764877527952194
DEBUG:root:i=39 residual=0.016309145838022232
DEBUG:root:i=40 residual=0.014999980106949806
DEBUG:root:i=41 residual=0.013821776024997234
DEBUG:root:i=42 residual=0.012760649435222149
DEBUG:root:i=43 residual=0.011803793720901012
DEBUG:root:i=44 residual=0.010939855128526688
DEBUG:root:i=45 residual=0.010158701799809933
DEBUG:root:i=46 residual=0.009451248683035374
DEBUG:root:i=47 residual=0.008809510618448257
DEBUG:root:i=48 residual=0.008226433768868446
DEBUG:root:i=49 residual=0.007695662323385477
DEBUG:root:i=50 residual=0.0072118379175662994
DEBUG:root:i=51 residual=0.006769917439669371
DEBUG:root:i=52 residual=0.006365581881254911
DEBUG:root:i=53 residual=0.005995078943669796
DEBUG:root:i=54 residual=0.005654919892549515
DEBUG:root:i=55 residual=0.005342131480574608
DEBUG:root:i=56 residual=0.0050540491938591
DEBUG:root:i=57 residual=0.004788342863321304
DEBUG:root:i=58 residual=0.004542799200862646
DEBUG:root:i=59 residual=0.0043155597522854805
DEBUG:root:i=60 residual=0.004104919265955687
DEBUG:root:i=61 residual=0.0039094300009310246
DEBUG:root:i=62 residual=0.003727702423930168
DEBUG:root:i=63 residual=0.0035584955476224422
DEBUG:root:i=64 residual=0.0034007669892162085
DEBUG:root:i=65 residual=0.003253525123000145
DEBUG:root:i=66 residual=0.003115906147286296
DEBUG:root:i=67 residual=0.002987051848322153
DEBUG:root:i=68 residual=0.0028663193807005882
DEBUG:root:i=69 residual=0.002753013977780938
DEBUG:root:i=70 residual=0.0026465721894055605
DEBUG:root:i=71 residual=0.0025464233476668596
DEBUG:root:i=72 residual=0.00245215673930943
DEBUG:root:i=73 residual=0.0023632629308849573
DEBUG:root:i=74 residual=0.0022793507669121027
DEBUG:root:i=75 residual=0.0022000810131430626
DEBUG:root:i=76 residual=0.002125096507370472
DEBUG:root:i=77 residual=0.0020541141275316477
DEBUG:root:i=78 residual=0.0019868274684995413
DEBUG:root:i=79 residual=0.0019229983445256948
DEBUG:root:i=80 residual=0.0018623856594786048
DEBUG:root:i=81 residual=0.0018047919729724526
DEBUG:root:i=82 residual=0.0017499927198514342
DEBUG:root:i=83 residual=0.0016978007042780519
DEBUG:root:i=84 residual=0.001648083794862032
DEBUG:root:i=85 residual=0.001600663410499692
DEBUG:root:i=86 residual=0.0015553995035588741
DEBUG:root:i=87 residual=0.0015121459728106856
DEBUG:root:i=88 residual=0.0014708059607073665
DEBUG:root:i=89 residual=0.0014312493149191141
DEBUG:root:i=90 residual=0.0013933753361925483
DEBUG:root:i=91 residual=0.0013570921728387475
DEBUG:root:i=92 residual=0.0013222922571003437
DEBUG:root:i=93 residual=0.0012889101635664701
DEBUG:root:i=94 residual=0.001256857649423182
DEBUG:root:i=95 residual=0.0012260567164048553
DEBUG:root:i=96 residual=0.0011964496225118637
DEBUG:root:i=97 residual=0.0011679684976115823
DEBUG:root:i=98 residual=0.0011405327823013067
DEBUG:root:i=99 residual=0.0011141204740852118
DEBUG:root:i=100 residual=0.0010886711534112692
DEBUG:root:i=101 residual=0.0010641429107636213
DEBUG:root:i=102 residual=0.0010404628701508045
DEBUG:root:i=103 residual=0.001017606002278626
DEBUG:root:i=104 residual=0.0009955442510545254
DEBUG:root:i=105 residual=0.0009742035181261599
DEBUG:root:i=106 residual=0.0009535751305520535
DEBUG:root:i=107 residual=0.0009336223592981696
DEBUG:root:i=108 residual=0.0009143262286670506
DEBUG:root:i=109 residual=0.0008956189849413931
DEBUG:root:i=110 residual=0.0008775112801231444
DEBUG:root:i=111 residual=0.0008599554421380162
DEBUG:root:i=112 residual=0.0008429353474639356
DEBUG:root:i=113 residual=0.0008264316129498184
DEBUG:root:i=114 residual=0.0008104113512672484
DEBUG:root:i=115 residual=0.0007948672864586115
DEBUG:root:i=116 residual=0.0007797663565725088
DEBUG:root:i=117 residual=0.0007651016931049526
DEBUG:root:i=118 residual=0.0007508271373808384
DEBUG:root:i=119 residual=0.000736961723305285
DEBUG:root:i=120 residual=0.0007234770455397666
DEBUG:root:i=121 residual=0.0007103522657416761
DEBUG:root:i=122 residual=0.000697571667842567
DEBUG:root:i=123 residual=0.0006851415382698178
DEBUG:root:i=124 residual=0.0006730229943059385
DEBUG:root:i=125 residual=0.000661214638967067
DEBUG:root:i=126 residual=0.0006497159483842552
DEBUG:root:i=127 residual=0.0006384956068359315
DEBUG:root:i=128 residual=0.0006275559426285326
DEBUG:root:i=129 residual=0.0006169050466269255
DEBUG:root:i=130 residual=0.0006064882036298513
DEBUG:root:i=131 residual=0.0005963204312138259
DEBUG:root:i=132 residual=0.0005863972473889589
DEBUG:root:i=133 residual=0.0005767198745161295
DEBUG:root:i=134 residual=0.0005672430852428079
DEBUG:root:i=135 residual=0.0005579989519901574
DEBUG:root:i=136 residual=0.0005489474860951304
DEBUG:root:i=137 residual=0.0005401145899668336
DEBUG:root:i=138 residual=0.0005314694717526436
DEBUG:root:i=139 residual=0.0005230205715633929
DEBUG:root:i=140 residual=0.0005147609044797719
DEBUG:root:i=141 residual=0.0005066748126409948
DEBUG:root:i=142 residual=0.0004987556603737175
DEBUG:root:i=143 residual=0.0004910103743895888
DEBUG:root:i=144 residual=0.00048342315130867064
DEBUG:root:i=145 residual=0.00047598540550097823
DEBUG:root:i=146 residual=0.00046870546066202223
DEBUG:root:i=147 residual=0.0004615831421688199
DEBUG:root:i=148 residual=0.00045459659304469824
DEBUG:root:i=149 residual=0.0004477547772694379
DEBUG:root:i=150 residual=0.0004410400870256126
DEBUG:root:i=151 residual=0.0004344591579865664
DEBUG:root:i=152 residual=0.00042801626841537654
DEBUG:root:i=153 residual=0.000421688862843439
DEBUG:root:i=154 residual=0.00041548715671524405
DEBUG:root:i=155 residual=0.00040940303006209433
DEBUG:root:i=156 residual=0.00040343115688301623
DEBUG:root:i=157 residual=0.00039757706690579653
DEBUG:root:i=158 residual=0.00039182015461847186
DEBUG:root:i=159 residual=0.00038617680547758937
DEBUG:root:i=160 residual=0.0003806350287050009
DEBUG:root:i=161 residual=0.0003751949407160282
DEBUG:root:i=162 residual=0.00036985395126976073
DEBUG:root:i=163 residual=0.0003646106633823365
DEBUG:root:i=164 residual=0.00035945544368587434
DEBUG:root:i=165 residual=0.0003543953353073448
DEBUG:root:i=166 residual=0.0003494191914796829
DEBUG:root:i=167 residual=0.0003445349575486034
DEBUG:root:i=168 residual=0.0003397348918952048
DEBUG:root:i=169 residual=0.00033501788857392967
DEBUG:root:i=170 residual=0.0003303826379124075
DEBUG:root:i=171 residual=0.0003258231736253947
DEBUG:root:i=172 residual=0.0003213426098227501
DEBUG:root:i=173 residual=0.0003169376286678016
DEBUG:root:i=174 residual=0.00031259676325134933
DEBUG:root:i=175 residual=0.000308340007904917
DEBUG:root:i=176 residual=0.0003041541203856468
DEBUG:root:i=177 residual=0.0003000321739818901
DEBUG:root:i=178 residual=0.0002959810954052955
DEBUG:root:i=179 residual=0.0002919910184573382
DEBUG:root:i=180 residual=0.00028807317721657455
DEBUG:root:i=181 residual=0.0002842079848051071
DEBUG:root:i=182 residual=0.0002804100513458252
DEBUG:root:i=183 residual=0.00027667314861901104
DEBUG:root:i=184 residual=0.00027299521025270224
DEBUG:root:i=185 residual=0.0002693750720936805
DEBUG:root:i=186 residual=0.00026581509155221283
DEBUG:root:i=187 residual=0.00026230356888845563
DEBUG:root:i=188 residual=0.00025885284412652254
DEBUG:root:i=189 residual=0.0002554590464569628
DEBUG:root:i=190 residual=0.00025210180319845676
DEBUG:root:i=191 residual=0.0002488067257218063
DEBUG:root:i=192 residual=0.00024556374410167336
DEBUG:root:i=193 residual=0.00024236431636381894
DEBUG:root:i=194 residual=0.0002392169408267364
DEBUG:root:i=195 residual=0.00023611949291080236
DEBUG:root:i=196 residual=0.0002330642892047763
DEBUG:root:i=197 residual=0.0002300576597917825
DEBUG:root:i=198 residual=0.00022709427867084742
DEBUG:root:i=199 residual=0.00022417739091906697
DEBUG:root:i=200 residual=0.00022130212164483964
DEBUG:root:i=201 residual=0.000218468441744335
DEBUG:root:i=202 residual=0.0002156790142180398
DEBUG:root:i=203 residual=0.0002129263011738658
DEBUG:root:i=204 residual=0.0002102184807881713
DEBUG:root:i=205 residual=0.00020754475553985685
DEBUG:root:i=206 residual=0.0002049119648290798
DEBUG:root:i=207 residual=0.00020231645612511784
DEBUG:root:i=208 residual=0.0001997604122152552
DEBUG:root:i=209 residual=0.00019723695004358888
DEBUG:root:i=210 residual=0.0001947526034200564
DEBUG:root:i=211 residual=0.00019230232283007354
DEBUG:root:i=212 residual=0.0001898846385302022
DEBUG:root:i=213 residual=0.000187507743248716
DEBUG:root:i=214 residual=0.00018516113050282001
DEBUG:root:i=215 residual=0.0001828431704780087
DEBUG:root:i=216 residual=0.00018056572298519313
DEBUG:root:i=217 residual=0.00017831676814239472
DEBUG:root:i=218 residual=0.00017609779024496675
DEBUG:root:i=219 residual=0.0001739105355227366
DEBUG:root:i=220 residual=0.00017175757966469973
DEBUG:root:i=221 residual=0.00016962781955953687
DEBUG:root:i=222 residual=0.00016753270756453276
DEBUG:root:i=223 residual=0.00016546310507692397
DEBUG:root:i=224 residual=0.00016342326125595719
DEBUG:root:i=225 residual=0.0001614115317352116
DEBUG:root:i=226 residual=0.00015942516620270908
DEBUG:root:i=227 residual=0.00015746788994874805
DEBUG:root:i=228 residual=0.00015553671983070672
DEBUG:root:i=229 residual=0.0001536299241706729
DEBUG:root:i=230 residual=0.00015175141743384302
DEBUG:root:i=231 residual=0.00014989795454312116
DEBUG:root:i=232 residual=0.00014806764374952763
DEBUG:root:i=233 residual=0.00014626377378590405
DEBUG:root:i=234 residual=0.0001444838708266616
DEBUG:root:i=235 residual=0.00014272700354922563
DEBUG:root:i=236 residual=0.00014099445252213627
DEBUG:root:i=237 residual=0.00013928281259723008
DEBUG:root:i=238 residual=0.00013759742432739586
DEBUG:root:i=239 residual=0.0001359327870886773
DEBUG:root:i=240 residual=0.00013428731472231448
DEBUG:root:i=241 residual=0.00013266786118038
DEBUG:root:i=242 residual=0.00013106588448863477
DEBUG:root:i=243 residual=0.00012948649236932397
DEBUG:root:i=244 residual=0.0001279286661883816
DEBUG:root:i=245 residual=0.00012639032502193004
DEBUG:root:i=246 residual=0.00012487253115978092
DEBUG:root:i=247 residual=0.00012337281077634543
DEBUG:root:i=248 residual=0.0001218973338836804
DEBUG:root:i=249 residual=0.00012043455353705212
DEBUG:root:i=250 residual=0.0001189926260849461
DEBUG:root:i=251 residual=0.0001175726720248349
DEBUG:root:i=252 residual=0.0001161648178822361
DEBUG:root:i=253 residual=0.00011478265514597297
DEBUG:root:i=254 residual=0.00011341453500790522
DEBUG:root:i=255 residual=0.00011206133058294654
DEBUG:root:i=256 residual=0.00011072734923800454
DEBUG:root:i=257 residual=0.00010941256186924875
DEBUG:root:i=258 residual=0.00010811303945956752
DEBUG:root:i=259 residual=0.00010683064465411007
DEBUG:root:i=260 residual=0.00010556263441685587
DEBUG:root:i=261 residual=0.00010431098053231835
DEBUG:root:i=262 residual=0.00010307957563782111
DEBUG:root:i=263 residual=0.00010186032159253955
DEBUG:root:i=264 residual=0.00010065481183119118
DEBUG:root:i=265 residual=9.946776845026761e-05
DEBUG:root:i=266 residual=9.829259215621278e-05
DEBUG:root:i=267 residual=9.713583858683705e-05
DEBUG:root:i=268 residual=9.599184704711661e-05
DEBUG:root:i=269 residual=9.486191265750676e-05
DEBUG:root:i=270 residual=9.37460717977956e-05
DEBUG:root:i=271 residual=9.264351683668792e-05
DEBUG:root:i=272 residual=9.155725274467841e-05
DEBUG:root:i=273 residual=9.048206266015768e-05
DEBUG:root:i=274 residual=8.942343993112445e-05
DEBUG:root:i=275 residual=8.837210043566301e-05
DEBUG:root:i=276 residual=8.733809227123857e-05
DEBUG:root:i=277 residual=8.631807577330619e-05
DEBUG:root:i=278 residual=8.530668856110424e-05
DEBUG:root:i=279 residual=8.431196329183877e-05
DEBUG:root:i=280 residual=8.332644210895523e-05
DEBUG:root:i=281 residual=8.235250425059348e-05
DEBUG:root:i=282 residual=8.139281999319792e-05
DEBUG:root:i=283 residual=8.044490095926449e-05
DEBUG:root:i=284 residual=7.95064406702295e-05
DEBUG:root:i=285 residual=7.858289609430358e-05
DEBUG:root:i=286 residual=7.766643102513626e-05
DEBUG:root:i=287 residual=7.676488894503564e-05
DEBUG:root:i=288 residual=7.587243453599513e-05
DEBUG:root:i=289 residual=7.499045022996143e-05
DEBUG:root:i=290 residual=7.412215927615762e-05
DEBUG:root:i=291 residual=7.326059858314693e-05
DEBUG:root:i=292 residual=7.241279672598466e-05
DEBUG:root:i=293 residual=7.15746427886188e-05
DEBUG:root:i=294 residual=7.074469613144174e-05
DEBUG:root:i=295 residual=6.992680573603138e-05
DEBUG:root:i=296 residual=6.911661330377683e-05
DEBUG:root:i=297 residual=6.831959763076156e-05
DEBUG:root:i=298 residual=6.752939953003079e-05
DEBUG:root:i=299 residual=6.674851465504616e-05
DEBUG:root:i=300 residual=6.597929314011708e-05
DEBUG:root:i=301 residual=6.521786417579278e-05
DEBUG:root:i=302 residual=6.446688348660246e-05
DEBUG:root:i=303 residual=6.372267671395093e-05
DEBUG:root:i=304 residual=6.299097731243819e-05
DEBUG:root:i=305 residual=6.22655643383041e-05
DEBUG:root:i=306 residual=6.154928269097582e-05
DEBUG:root:i=307 residual=6.084038250264712e-05
DEBUG:root:i=308 residual=6.0140955611132085e-05
DEBUG:root:i=309 residual=5.945054363110103e-05
DEBUG:root:i=310 residual=5.876611976418644e-05
DEBUG:root:i=311 residual=5.8091030950890854e-05
DEBUG:root:i=312 residual=5.742695429944433e-05
DEBUG:root:i=313 residual=5.6767814385239035e-05
DEBUG:root:i=314 residual=5.611626693280414e-05
DEBUG:root:i=315 residual=5.547347973333672e-05
DEBUG:root:i=316 residual=5.483692802954465e-05
DEBUG:root:i=317 residual=5.4211184760788456e-05
DEBUG:root:i=318 residual=5.358945782063529e-05
DEBUG:root:i=319 residual=5.297775351209566e-05
DEBUG:root:i=320 residual=5.2369934564922005e-05
DEBUG:root:i=321 residual=5.17721964570228e-05
DEBUG:root:i=322 residual=5.1180653827032074e-05
DEBUG:root:i=323 residual=5.059864270151593e-05
DEBUG:root:i=324 residual=5.001969839213416e-05
DEBUG:root:i=325 residual=4.944805914419703e-05
DEBUG:root:i=326 residual=4.888329567620531e-05
DEBUG:root:i=327 residual=4.832754348171875e-05
DEBUG:root:i=328 residual=4.777670619660057e-05
DEBUG:root:i=329 residual=4.723239908344112e-05
DEBUG:root:i=330 residual=4.669166082749143e-05
DEBUG:root:i=331 residual=4.616142177837901e-05
DEBUG:root:i=332 residual=4.5634395064553246e-05
DEBUG:root:i=333 residual=4.5116674300516024e-05
DEBUG:root:i=334 residual=4.46023877884727e-05
DEBUG:root:i=335 residual=4.4096188503317535e-05
DEBUG:root:i=336 residual=4.359498780104332e-05
DEBUG:root:i=337 residual=4.309757423470728e-05
DEBUG:root:i=338 residual=4.2609281081240624e-05
DEBUG:root:i=339 residual=4.212408748571761e-05
DEBUG:root:i=340 residual=4.164765414316207e-05
DEBUG:root:i=341 residual=4.117452772334218e-05
DEBUG:root:i=342 residual=4.07077168347314e-05
DEBUG:root:i=343 residual=4.024584995931946e-05
DEBUG:root:i=344 residual=3.9789800212020054e-05
DEBUG:root:i=345 residual=3.933750485884957e-05
DEBUG:root:i=346 residual=3.889457002514973e-05
DEBUG:root:i=347 residual=3.845261744572781e-05
DEBUG:root:i=348 residual=3.801717684837058e-05
DEBUG:root:i=349 residual=3.758699676836841e-05
DEBUG:root:i=350 residual=3.716169885592535e-05
DEBUG:root:i=351 residual=3.674226172734052e-05
DEBUG:root:i=352 residual=3.6325429391581565e-05
DEBUG:root:i=353 residual=3.591417771531269e-05
DEBUG:root:i=354 residual=3.5507866414263844e-05
DEBUG:root:i=355 residual=3.510791066219099e-05
DEBUG:root:i=356 residual=3.471046147751622e-05
DEBUG:root:i=357 residual=3.4319924452574924e-05
DEBUG:root:i=358 residual=3.3931130019482225e-05
DEBUG:root:i=359 residual=3.3547530620126054e-05
DEBUG:root:i=360 residual=3.316853326396085e-05
DEBUG:root:i=361 residual=3.279683369328268e-05
DEBUG:root:i=362 residual=3.2426312827738e-05
DEBUG:root:i=363 residual=3.20615254167933e-05
DEBUG:root:i=364 residual=3.170006675645709e-05
DEBUG:root:i=365 residual=3.1341140129370615e-05
DEBUG:root:i=366 residual=3.098652814514935e-05
DEBUG:root:i=367 residual=3.0637784220743924e-05
DEBUG:root:i=368 residual=3.0293558666016906e-05
DEBUG:root:i=369 residual=2.995552131324075e-05
DEBUG:root:i=370 residual=2.961474092444405e-05
DEBUG:root:i=371 residual=2.9284423362696543e-05
DEBUG:root:i=372 residual=2.8952184948138893e-05
DEBUG:root:i=373 residual=2.86289541691076e-05
DEBUG:root:i=374 residual=2.8306756576057523e-05
DEBUG:root:i=375 residual=2.7988471629214473e-05
DEBUG:root:i=376 residual=2.7672840587911196e-05
DEBUG:root:i=377 residual=2.7362420951249078e-05
DEBUG:root:i=378 residual=2.7054886231780984e-05
DEBUG:root:i=379 residual=2.6752177291200496e-05
DEBUG:root:i=380 residual=2.6451112717040814e-05
DEBUG:root:i=381 residual=2.615185803733766e-05
DEBUG:root:i=382 residual=2.5858938897727057e-05
DEBUG:root:i=383 residual=2.5569823264959268e-05
DEBUG:root:i=384 residual=2.5282084607169963e-05
DEBUG:root:i=385 residual=2.4999648303491995e-05
DEBUG:root:i=386 residual=2.4720335204619914e-05
DEBUG:root:i=387 residual=2.4441582354484126e-05
DEBUG:root:i=388 residual=2.416684765194077e-05
DEBUG:root:i=389 residual=2.3894897822174244e-05
DEBUG:root:i=390 residual=2.3629043425899e-05
DEBUG:root:i=391 residual=2.336370016564615e-05
DEBUG:root:i=392 residual=2.3102234990801662e-05
DEBUG:root:i=393 residual=2.2843218175694346e-05
DEBUG:root:i=394 residual=2.258568929391913e-05
DEBUG:root:i=395 residual=2.2334837922244333e-05
DEBUG:root:i=396 residual=2.20843794522807e-05
DEBUG:root:i=397 residual=2.1836998712387867e-05
DEBUG:root:i=398 residual=2.1591717086266726e-05
DEBUG:root:i=399 residual=2.134899841621518e-05
DEBUG:root:i=400 residual=2.1113213733769953e-05
DEBUG:root:i=401 residual=2.0873996618320234e-05
DEBUG:root:i=402 residual=2.064129694190342e-05
DEBUG:root:i=403 residual=2.0410423530847766e-05
DEBUG:root:i=404 residual=2.018347367993556e-05
DEBUG:root:i=405 residual=1.9956703908974305e-05
DEBUG:root:i=406 residual=1.9732951841433533e-05
DEBUG:root:i=407 residual=1.951310878212098e-05
DEBUG:root:i=408 residual=1.9297496692161076e-05
DEBUG:root:i=409 residual=1.9079665435128845e-05
DEBUG:root:i=410 residual=1.886559766717255e-05
DEBUG:root:i=411 residual=1.865748708951287e-05
DEBUG:root:i=412 residual=1.8446024114382453e-05
DEBUG:root:i=413 residual=1.8241002180729993e-05
DEBUG:root:i=414 residual=1.8039398128166795e-05
DEBUG:root:i=415 residual=1.7835634935181588e-05
DEBUG:root:i=416 residual=1.7638911231188104e-05
DEBUG:root:i=417 residual=1.7442314856452867e-05
DEBUG:root:i=418 residual=1.7244448827113956e-05
DEBUG:root:i=419 residual=1.7053544070222415e-05
DEBUG:root:i=420 residual=1.6863272321643308e-05
DEBUG:root:i=421 residual=1.667503238422796e-05
DEBUG:root:i=422 residual=1.6488424080307595e-05
DEBUG:root:i=423 residual=1.6304235032293946e-05
DEBUG:root:i=424 residual=1.6124195099109784e-05
DEBUG:root:i=425 residual=1.5942092431942e-05
DEBUG:root:i=426 residual=1.5765841453685425e-05
DEBUG:root:i=427 residual=1.559055453981273e-05
DEBUG:root:i=428 residual=1.5415629604831338e-05
DEBUG:root:i=429 residual=1.5244248061208054e-05
DEBUG:root:i=430 residual=1.507440538262017e-05
DEBUG:root:i=431 residual=1.4907756849424914e-05
DEBUG:root:i=432 residual=1.474164855608251e-05
DEBUG:root:i=433 residual=1.457678445149213e-05
DEBUG:root:i=434 residual=1.4413960343517829e-05
DEBUG:root:i=435 residual=1.4253871086111758e-05
DEBUG:root:i=436 residual=1.4094122889218852e-05
DEBUG:root:i=437 residual=1.3937602489022538e-05
DEBUG:root:i=438 residual=1.3783455869997852e-05
DEBUG:root:i=439 residual=1.362865532428259e-05
DEBUG:root:i=440 residual=1.3478947948897257e-05
DEBUG:root:i=441 residual=1.3326177395356353e-05
DEBUG:root:i=442 residual=1.3179163943277672e-05
DEBUG:root:i=443 residual=1.3032953575020656e-05
DEBUG:root:i=444 residual=1.2888053788628895e-05
DEBUG:root:i=445 residual=1.2743099432555027e-05
DEBUG:root:i=446 residual=1.2601353773789015e-05
DEBUG:root:i=447 residual=1.2462138329283334e-05
DEBUG:root:i=448 residual=1.2325113857514225e-05
DEBUG:root:i=449 residual=1.2187208085379098e-05
DEBUG:root:i=450 residual=1.205128228320973e-05
DEBUG:root:i=451 residual=1.1917309166165069e-05
DEBUG:root:i=452 residual=1.1783678019128274e-05
DEBUG:root:i=453 residual=1.1653241926978808e-05
DEBUG:root:i=454 residual=1.152361073764041e-05
DEBUG:root:i=455 residual=1.139728101406945e-05
DEBUG:root:i=456 residual=1.127015730162384e-05
DEBUG:root:i=457 residual=1.1144874406454619e-05
DEBUG:root:i=458 residual=1.101862835639622e-05
DEBUG:root:i=459 residual=1.090035311790416e-05
DEBUG:root:i=460 residual=1.0777873285405803e-05
DEBUG:root:i=461 residual=1.0658224709914066e-05
DEBUG:root:i=462 residual=1.0539751201577019e-05
DEBUG:root:i=463 residual=1.0422270861454308e-05
DEBUG:root:i=464 residual=1.030757721309783e-05
DEBUG:root:i=465 residual=1.0193395610258449e-05
DEBUG:root:i=466 residual=1.0081096661451738e-05
DEBUG:root:i=467 residual=9.966545803763438e-06
DEBUG:root:i=468 residual=9.857409168034792e-06
DEBUG:root:i=469 residual=9.745364877744578e-06
DEBUG:root:i=470 residual=9.63919228524901e-06
DEBUG:root:i=471 residual=9.532192052574828e-06
DEBUG:root:i=472 residual=9.426649739907589e-06
DEBUG:root:i=473 residual=9.320484423369635e-06
DEBUG:root:i=474 residual=9.217823389917612e-06
DEBUG:root:i=475 residual=9.113819942285772e-06
DEBUG:root:i=476 residual=9.014381248562131e-06
DEBUG:root:i=477 residual=8.914938007364981e-06
DEBUG:root:i=478 residual=8.815082765067928e-06
DEBUG:root:i=479 residual=8.719725883565843e-06
DEBUG:root:i=480 residual=8.62204069562722e-06
DEBUG:root:i=481 residual=8.523898941348307e-06
DEBUG:root:i=482 residual=8.430986781604588e-06
DEBUG:root:i=483 residual=8.337911822309252e-06
DEBUG:root:i=484 residual=8.244614946306683e-06
DEBUG:root:i=485 residual=8.15321345726261e-06
DEBUG:root:i=486 residual=8.062536835495848e-06
DEBUG:root:i=487 residual=7.973941137606744e-06
DEBUG:root:i=488 residual=7.887157153163571e-06
DEBUG:root:i=489 residual=7.800698767823633e-06
DEBUG:root:i=490 residual=7.712767001066823e-06
DEBUG:root:i=491 residual=7.6273540798865724e-06
DEBUG:root:i=492 residual=7.541502327512717e-06
DEBUG:root:i=493 residual=7.459529570041923e-06
DEBUG:root:i=494 residual=7.375418590527261e-06
DEBUG:root:i=495 residual=7.294529495993629e-06
DEBUG:root:i=496 residual=7.21162541594822e-06
DEBUG:root:i=497 residual=7.1323247539112344e-06
DEBUG:root:i=498 residual=7.053663011902245e-06
DEBUG:root:i=499 residual=6.9741622610308696e-06
DEBUG:root:i=500 residual=6.898417268530466e-06
DEBUG:root:i=501 residual=6.822542218287708e-06
DEBUG:root:i=502 residual=6.744119673385285e-06
DEBUG:root:i=503 residual=6.670498805760872e-06
DEBUG:root:i=504 residual=6.597599167434964e-06
DEBUG:root:i=505 residual=6.525076059915591e-06
DEBUG:root:i=506 residual=6.451755325542763e-06
DEBUG:root:i=507 residual=6.380418653861852e-06
DEBUG:root:i=508 residual=6.310235676210141e-06
DEBUG:root:i=509 residual=6.2398471527558286e-06
DEBUG:root:i=510 residual=6.1706273299932946e-06
DEBUG:root:i=511 residual=6.10296137892874e-06
DEBUG:root:i=512 residual=6.035129445081111e-06
DEBUG:root:i=513 residual=5.968372533970978e-06
DEBUG:root:i=514 residual=5.9023755056841765e-06
DEBUG:root:i=515 residual=5.837246590090217e-06
DEBUG:root:i=516 residual=5.772832992079202e-06
DEBUG:root:i=517 residual=5.70771817365312e-06
DEBUG:root:i=518 residual=5.6455101002939045e-06
DEBUG:root:i=519 residual=5.581242476182524e-06
DEBUG:root:i=520 residual=5.5194586821016856e-06
DEBUG:root:i=521 residual=5.4606002777291e-06
DEBUG:root:i=522 residual=5.3987305363989435e-06
DEBUG:root:i=523 residual=5.339476956578437e-06
DEBUG:root:i=524 residual=5.27975589648122e-06
DEBUG:root:i=525 residual=5.221965238888515e-06
DEBUG:root:i=526 residual=5.163279183761915e-06
DEBUG:root:i=527 residual=5.107664492243202e-06
DEBUG:root:i=528 residual=5.048700586485211e-06
DEBUG:root:i=529 residual=4.995838935428765e-06
DEBUG:root:i=530 residual=4.939173322782153e-06
DEBUG:root:i=531 residual=4.885442194790812e-06
DEBUG:root:i=532 residual=4.8304918891517445e-06
DEBUG:root:i=533 residual=4.776177775056567e-06
DEBUG:root:i=534 residual=4.723845904663904e-06
DEBUG:root:i=535 residual=4.672079739975743e-06
DEBUG:root:i=536 residual=4.618900220521027e-06
DEBUG:root:i=537 residual=4.5689848775509745e-06
DEBUG:root:i=538 residual=4.5192509787739255e-06
DEBUG:root:i=539 residual=4.468935003387742e-06
DEBUG:root:i=540 residual=4.419533524924191e-06
DEBUG:root:i=541 residual=4.36980917584151e-06
DEBUG:root:i=542 residual=4.323180746723665e-06
DEBUG:root:i=543 residual=4.274755610822467e-06
DEBUG:root:i=544 residual=4.227171302773058e-06
DEBUG:root:i=545 residual=4.1826638152997475e-06
DEBUG:root:i=546 residual=4.1351645450049546e-06
DEBUG:root:i=547 residual=4.087383331352612e-06
DEBUG:root:i=548 residual=4.0425820770906284e-06
DEBUG:root:i=549 residual=3.997784006060101e-06
DEBUG:root:i=550 residual=3.954427029384533e-06
DEBUG:root:i=551 residual=3.909859060513554e-06
DEBUG:root:i=552 residual=3.86733017876395e-06
DEBUG:root:i=553 residual=3.824732175417012e-06
DEBUG:root:i=554 residual=3.7832239740964724e-06
DEBUG:root:i=555 residual=3.741764885489829e-06
DEBUG:root:i=556 residual=3.697850843309425e-06
DEBUG:root:i=557 residual=3.6577810078597395e-06
DEBUG:root:i=558 residual=3.61927618541813e-06
DEBUG:root:i=559 residual=3.5791549635177944e-06
DEBUG:root:i=560 residual=3.5383004615141544e-06
DEBUG:root:i=561 residual=3.500159891700605e-06
DEBUG:root:i=562 residual=3.461761025391752e-06
DEBUG:root:i=563 residual=3.422477220738074e-06
DEBUG:root:i=564 residual=3.3854621506179683e-06
DEBUG:root:i=565 residual=3.346774292367627e-06
DEBUG:root:i=566 residual=3.311339014544501e-06
DEBUG:root:i=567 residual=3.27099451169488e-06
DEBUG:root:i=568 residual=3.238585804865579e-06
DEBUG:root:i=569 residual=3.201000481567462e-06
DEBUG:root:i=570 residual=3.1676349863118958e-06
DEBUG:root:i=571 residual=3.1334193408838473e-06
DEBUG:root:i=572 residual=3.0983530905359657e-06
DEBUG:root:i=573 residual=3.0660196443932364e-06
DEBUG:root:i=574 residual=3.028825176443206e-06
DEBUG:root:i=575 residual=2.993406269524712e-06
DEBUG:root:i=576 residual=2.96461371362966e-06
DEBUG:root:i=577 residual=2.9287409688549815e-06
DEBUG:root:i=578 residual=2.895996658480726e-06
DEBUG:root:i=579 residual=2.8644042231462663e-06
DEBUG:root:i=580 residual=2.8334361559245735e-06
DEBUG:root:i=581 residual=2.8022016067552613e-06
DEBUG:root:i=582 residual=2.771329718598281e-06
DEBUG:root:i=583 residual=2.7403596050135093e-06
DEBUG:root:i=584 residual=2.710224862312316e-06
DEBUG:root:i=585 residual=2.682766535144765e-06
DEBUG:root:i=586 residual=2.6514296678215032e-06
DEBUG:root:i=587 residual=2.623000227686134e-06
DEBUG:root:i=588 residual=2.5921451651811367e-06
DEBUG:root:i=589 residual=2.5645358618930914e-06
DEBUG:root:i=590 residual=2.5339475087093888e-06
DEBUG:root:i=591 residual=2.5076201382034924e-06
DEBUG:root:i=592 residual=2.4801761355774943e-06
DEBUG:root:i=593 residual=2.4524804302927805e-06
DEBUG:root:i=594 residual=2.4278667751786998e-06
DEBUG:root:i=595 residual=2.4003227281355066e-06
DEBUG:root:i=596 residual=2.3755412712489488e-06
DEBUG:root:i=597 residual=2.345230313949287e-06
DEBUG:root:i=598 residual=2.3197821974463295e-06
DEBUG:root:i=599 residual=2.2938127131055808e-06
DEBUG:root:i=600 residual=2.2683250335830962e-06
DEBUG:root:i=601 residual=2.2453186829807237e-06
DEBUG:root:i=602 residual=2.2234935386222787e-06
DEBUG:root:i=603 residual=2.19465141526598e-06
DEBUG:root:i=604 residual=2.169742629121174e-06
DEBUG:root:i=605 residual=2.147511622752063e-06
DEBUG:root:i=606 residual=2.121161287504947e-06
DEBUG:root:i=607 residual=2.0990883058402687e-06
DEBUG:root:i=608 residual=2.0749355371663114e-06
DEBUG:root:i=609 residual=2.053302068816265e-06
DEBUG:root:i=610 residual=2.0326308458606945e-06
DEBUG:root:i=611 residual=2.012214054047945e-06
DEBUG:root:i=612 residual=1.986934421438491e-06
DEBUG:root:i=613 residual=1.96305677491182e-06
DEBUG:root:i=614 residual=1.9435819922364317e-06
DEBUG:root:i=615 residual=1.9203707779524848e-06
DEBUG:root:i=616 residual=1.9002058024852886e-06
DEBUG:root:i=617 residual=1.8817426052919473e-06
DEBUG:root:i=618 residual=1.860031943579088e-06
DEBUG:root:i=619 residual=1.838025468714477e-06
DEBUG:root:i=620 residual=1.8171671172240167e-06
DEBUG:root:i=621 residual=1.7985312297241762e-06
DEBUG:root:i=622 residual=1.7785631598599139e-06
DEBUG:root:i=623 residual=1.75648597178224e-06
DEBUG:root:i=624 residual=1.736808940222545e-06
DEBUG:root:i=625 residual=1.7192304540003533e-06
DEBUG:root:i=626 residual=1.7010549981932854e-06
DEBUG:root:i=627 residual=1.6807225620141253e-06
DEBUG:root:i=628 residual=1.663842226662382e-06
DEBUG:root:i=629 residual=1.6447642110506422e-06
DEBUG:root:i=630 residual=1.6247403209490585e-06
DEBUG:root:i=631 residual=1.6087982430690317e-06
DEBUG:root:i=632 residual=1.5922298643999966e-06
DEBUG:root:i=633 residual=1.5758547533550882e-06
DEBUG:root:i=634 residual=1.5566650972687057e-06
DEBUG:root:i=635 residual=1.540898210805608e-06
DEBUG:root:i=636 residual=1.5202102758848923e-06
DEBUG:root:i=637 residual=1.503880866948748e-06
DEBUG:root:i=638 residual=1.4898055269441102e-06
DEBUG:root:i=639 residual=1.4704871773574268e-06
DEBUG:root:i=640 residual=1.4583395113731967e-06
DEBUG:root:i=641 residual=1.439593916074955e-06
DEBUG:root:i=642 residual=1.422935156369931e-06
DEBUG:root:i=643 residual=1.4127629128779517e-06
DEBUG:root:i=644 residual=1.3922415291744983e-06
DEBUG:root:i=645 residual=1.3786627732770285e-06
DEBUG:root:i=646 residual=1.3727486702919123e-06
DEBUG:root:i=647 residual=1.3531531521948637e-06
DEBUG:root:i=648 residual=1.3303941841513733e-06
DEBUG:root:i=649 residual=1.3167472161512705e-06
DEBUG:root:i=650 residual=1.3036830068813288e-06
DEBUG:root:i=651 residual=1.2914330227431492e-06
DEBUG:root:i=652 residual=1.281239860873029e-06
DEBUG:root:i=653 residual=1.2620846518984763e-06
DEBUG:root:i=654 residual=1.247077534571872e-06
DEBUG:root:i=655 residual=1.233281182067003e-06
DEBUG:root:i=656 residual=1.22477422337397e-06
DEBUG:root:i=657 residual=1.206897309202759e-06
DEBUG:root:i=658 residual=1.1914451079064747e-06
DEBUG:root:i=659 residual=1.1808957651737728e-06
DEBUG:root:i=660 residual=1.1664624253171496e-06
DEBUG:root:i=661 residual=1.151899823526037e-06
DEBUG:root:i=662 residual=1.1403910775698023e-06
DEBUG:root:i=663 residual=1.1302741995677934e-06
DEBUG:root:i=664 residual=1.117987835641543e-06
DEBUG:root:i=665 residual=1.1107479167549172e-06
DEBUG:root:i=666 residual=1.0924062507911003e-06
DEBUG:root:i=667 residual=1.0795660045914701e-06
DEBUG:root:i=668 residual=1.0675724979591905e-06
DEBUG:root:i=669 residual=1.0567736126176897e-06
DEBUG:root:i=670 residual=1.04483399354649e-06
DEBUG:root:i=671 residual=1.0313269740436226e-06
DEBUG:root:i=672 residual=1.0184187431150349e-06
DEBUG:root:i=673 residual=1.00933755220467e-06
DEBUG:root:i=674 residual=1.0012264510805835e-06
DEBUG:root:i=675 residual=9.892829666569014e-07
INFO:root:rank=0 pagerank=7.0147e-01 url=www.lawfareblog.com/covid-19-speech-and-surveillance-response
INFO:root:rank=1 pagerank=7.0147e-01 url=www.lawfareblog.com/lawfare-live-covid-19-speech-and-surveillance
INFO:root:rank=2 pagerank=1.0537e-01 url=www.lawfareblog.com/cost-using-zero-days
INFO:root:rank=3 pagerank=3.1757e-02 url=www.lawfareblog.com/lawfare-podcast-former-congressman-brian-baird-and-daniel-schuman-how-congress-can-continue-function
INFO:root:rank=4 pagerank=2.2269e-02 url=www.lawfareblog.com/events
INFO:root:rank=5 pagerank=1.6050e-02 url=www.lawfareblog.com/water-wars-increased-us-focus-indo-pacific
INFO:root:rank=6 pagerank=1.6049e-02 url=www.lawfareblog.com/water-wars-drill-maybe-drill
INFO:root:rank=7 pagerank=1.6045e-02 url=www.lawfareblog.com/water-wars-disjointed-operations-south-china-sea
INFO:root:rank=8 pagerank=1.6042e-02 url=www.lawfareblog.com/water-wars-song-oil-and-fire
INFO:root:rank=9 pagerank=1.6042e-02 url=www.lawfareblog.com/water-wars-sinking-feeling-philippine-china-relations
   ```

   Task 2, part 1:
   ```
   $ python3 pagerank.py --data=data/lawfareblog.csv.gz --filter_ratio=0.2 --personalization_vector_query='corona'
   
   INFO:root:rank=0 pagerank=6.2063e-01 url=www.lawfareblog.com/covid-19-speech-and-surveillance-response
INFO:root:rank=1 pagerank=6.2062e-01 url=www.lawfareblog.com/lawfare-live-covid-19-speech-and-surveillance
INFO:root:rank=2 pagerank=2.6287e-01 url=www.lawfareblog.com/rational-security-my-corona-edition
INFO:root:rank=3 pagerank=1.3901e-01 url=www.lawfareblog.com/chinatalk-how-party-takes-its-propaganda-global
INFO:root:rank=4 pagerank=8.2203e-02 url=www.lawfareblog.com/livestream-house-oversight-committee-holds-hearing-government-coronavirus-response
INFO:root:rank=5 pagerank=7.0874e-02 url=www.lawfareblog.com/lawfare-podcast-mom-and-dad-talk-clinical-trials-pandemic
INFO:root:rank=6 pagerank=6.8888e-02 url=www.lawfareblog.com/house-oversight-committee-holds-day-two-hearing-government-coronavirus-response
INFO:root:rank=7 pagerank=6.8328e-02 url=www.lawfareblog.com/trump-cant-reopen-country-over-state-objections
INFO:root:rank=8 pagerank=6.6056e-02 url=www.lawfareblog.com/water-wars-coronavirus-spreads-risk-conflict-around-south-china-sea
INFO:root:rank=9 pagerank=6.5243e-02 url=www.lawfareblog.com/lawfare-podcast-united-nations-and-coronavirus-crisis
   ```

   Task 2, part 2:
   ```
   $ python3 pagerank.py --data=data/lawfareblog.csv.gz --filter_ratio=0.2 --personalization_vector_query='corona' --search_query='-corona'
   
   INFO:root:rank=0 pagerank=6.2063e-01 url=www.lawfareblog.com/covid-19-speech-and-surveillance-response
INFO:root:rank=1 pagerank=6.2062e-01 url=www.lawfareblog.com/lawfare-live-covid-19-speech-and-surveillance
INFO:root:rank=2 pagerank=1.3901e-01 url=www.lawfareblog.com/chinatalk-how-party-takes-its-propaganda-global
INFO:root:rank=3 pagerank=7.0874e-02 url=www.lawfareblog.com/lawfare-podcast-mom-and-dad-talk-clinical-trials-pandemic
INFO:root:rank=4 pagerank=6.8328e-02 url=www.lawfareblog.com/trump-cant-reopen-country-over-state-objections
INFO:root:rank=5 pagerank=6.2202e-02 url=www.lawfareblog.com/us-moves-dismiss-case-against-company-linked-ira-troll-farm
INFO:root:rank=6 pagerank=6.2202e-02 url=www.lawfareblog.com/livestream-house-armed-services-holds-hearing-national-security-challenges-north-and-south-america
INFO:root:rank=7 pagerank=6.2202e-02 url=www.lawfareblog.com/livestream-house-armed-services-committee-holds-hearing-priorities-missile-defense
INFO:root:rank=8 pagerank=6.2202e-02 url=www.lawfareblog.com/livestream-house-foreign-affairs-committee-holds-hearing-crisis-idlib
INFO:root:rank=9 pagerank=5.2615e-02 url=www.lawfareblog.com/limits-world-health-organization
   ```

1. Ensure that all your changes to the `pagerank.py` and `README.md` files are committed to your repo and pushed to github.

1. Get at least 5 stars on your repo.
   (You may trade stars with other students in the class.)

   > **NOTE:**
   > 
   > Recruiters use github profiles to determine who to hire,
   > and pagerank is used to rank user profiles and projects.
   > Links in this graph correspond to who has starred/followed who's repo.
   > By getting more stars on your repo, you'll be increasing your github pagerank, which increases the likelihood that recruiters will hire you.
   > To see an example, [perform a search for `data mining`](https://github.com/search?q=data+mining).
   > Notice that the results are returned "approximately" ranked by the number of stars,
   > but because "some stars count more than others" the results are not exactly ranked by the number of stars.
   > (I asked you not to fork this repo because forks are ranked lower than non-forks.)
   >
   > In some sense, we are doing a "dual problem" to data mining by getting these stars.
   > Recruiters are using data mining to find out who the best people to recruit are,
   > and we are hacking their data mining algorithms by making those algorithms select you instead of someone else.
   >
   > If you're interested in exploring this idea further, here's a python tutorial for extracting GitHub's social graph: <https://www.oreilly.com/library/view/mining-the-social/9781449368180/ch07.html> ; if you're interested in learning more about how recruiters use github profiles, read this Hacker News post: <https://news.ycombinator.com/item?id=19413348>.

1. Submit the url of your repo to sakai.

   Each part is worth 2 points, for 12 points overall.
