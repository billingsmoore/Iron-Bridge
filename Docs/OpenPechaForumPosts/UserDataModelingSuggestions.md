# User Data Modeling Suggestions

This document is intended to summarize and respond to an ongoing discussion between myself, Jacob Moore[^1], and Tashi Dhondup regarding topic modeling of user translations.
This document has been written in such a way as to be useful as a reference point for future discussion of this project and 
may be useful for additional topic modeling in the future.

## Statement of Objective

The goal we are trying to accomplish is to perform topic modeling on the translations produced by users to better understand what sort of translations are being produced. 
That is, to understand the topics about which users generally want to create translations. Additionally, we want to be able to train an accurate classifier to identify the topics 
of future translations as they are made.

## Current Procedure

The current procedure being used for this project is as follows. A test dataset was separated from the initial dataset of user translations.
This test dataset was assigned a 'true' topic label determined by keywords in the translated text. For example, a text containing the word "laptop" might be labeled "Technology".

Then, an unsupervised clustering algorithm was used to create topic clusters in the remainder of the dataset using the BertTopic[^2] pipeline. The cluster labels assigned by this unsupervised model
were then treated as predicted labels and evaluated against the 'true' labels in the test dataset.

## Concerns Regarding the Current Procedure

The procedure described above is very unusual for an unsupervised topic model. In general, supervised models are evaluated for accuracy, precision, etc. against previously labeled data,
whereas unsupervised models are evaluated using a distinct set of metrics. Unsupervised modeling is typically used to help establish meaningful categories in the data prior to any form of programatic labeling.

Comparing unsupervised models against previously labeled data tends to bias efforts toward reproducing the labels that are being treated as 'true', even when those labels are only intended to be preliminary.
This can produce unsatisfactory results in the final analysis.

Additionally, while keyword-based approaches to topic labeling have their merits, they tend to be less reliable and less informative than more complex approaches. Consider a simple, potentially problematic example. Finding the keyword "apple" in a given text provides no way of distinguishing whether the text is about technology ("I want to buy an Apple computer") or produce ("I need to buy an apple for lunch").

Achieving high accuracy (a metric that is not meaningful in unsupervised modeling) on test data whose 'true' labels have been generated by keyword matching, necessarily means that our model performs no better than keyword labeling, simply because the model is learning to reproduce those keyword labels.

## Suggested Procedure

### Unsupervised Clustering

I recommend that data be clustered using BertTopic to produce a first look at potential categories for classification. The hyperparameters for the projection and clustering steps in that pipeline can then be altered based on whether or not the clusters are sufficiently detailed.

The simplest way to assess this clustering is to visually inspect the clusters for a general cohesiveness. More formally, clusters can be evaluated with any of a number of clustering-specific metrics. I typically prefer Silhouette Score as a simple starting point, as it captures cohesiveness within clusters, as well as separation between clusters in a relatively straightforward way. However, in my easy_text_clustering optimization tool, I've implemented a combined metric using Silhouette Score, the Calinski-Harabasz index, and the Davies-Bouldin index. These can be weighted in whatever way is preferred.[^3]

Data which are treated as outliers or noise by initial clustering can be sorted into the established clusters relatively easily using the 'reduce_outliers' method provided by BertTopic, as in the following code from the BertTopic documentation[^4]:

```python
from bertopic import BERTopic

# Train your BERTopic model
topic_model = BERTopic()
topics, probs = topic_model.fit_transform(docs)

# Reduce outliers
new_topics = topic_model.reduce_outliers(docs, topics)
```

Once satisfactory clustering has been achieved, large clusters could be extracted, and clustering models could be trained on just that subset of the data. This would allow greater detail and specificity in the topics found, as well as provide an ontological hierarchy for those topics. For example, a cluster labeled "Technology" could then be modeled with sub-clusters corresponding to "Advances in Technology", "Purchasing Devices", etc. 

A hierarchy of this kind would allow supervised classification models to be trained at different levels in that hierarchy. This is beneficial because classification models tend to perform better with relatively small numbers of categories to distinguish.[^5]

### Supervised Classification

Once a satisfactory set of labels have been generated, a supervised classification model can be trained on those labels.

A common approach here is to use a BERT model for classification. However, anecdotally, I find that BERT models frequently perform only slightly better than non-language model approaches and are significantly more computationally expensive.

Instead, a boosted random forest model (i.e. XGBoost[^6]) can often provide comparable performance. However, where performance is not sufficient, these models can easily be paired with a larger BERT (or comparable) language model by passing data whose confidence scores from the random forest model are too low on to the larger model for classification in an architecture known as "Baby Bear"[^7].

[^1]: Also referred to as "Jack" in other documentation regarding this project.
[^2]: Grootendorst, Maarten. "BERTopic: Neural topic modeling with a class-based TF-IDF procedure." *arXiv preprint arXiv:2203.05794* (2022).
[^3]: https://pypi.org/project/easy-text-clustering/
[^4]: https://maartengr.github.io/BERTopic/getting_started/outlier_reduction/outlier_reduction.html
[^5]: Norris, Spencer (2019, July 15). The latest advances in classification with too many labels. Open Data Science. https://opendatascience.com/the-latest-advances-in-classification-with-too-many-labels/
[^6]: https://xgboost.readthedocs.io/en/latest/index.html
[^7]: Khalili, Leila, Yao You, and John Bohannon. "BabyBear: Cheap inference triage for expensive language models." *arXiv preprint arXiv:2205.11747* (2022). [Available online](https://arxiv.org/abs/2205.11747).