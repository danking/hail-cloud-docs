# How To Cloud With Hail

## Prerequisites

Read [How to Cloud](./how-to-cloud.md) first.

You should already have Hail installed on your laptop; see the [Hail
installation documentation](https://hail.is/docs/0.2/getting_started.html). You
should already have completed the [GWAS
Tutorial](https://hail.is/docs/0.2/tutorials-landing.html) on your laptop. You
do not need to have read the
[Overview](https://hail.is/docs/0.2/overview/index.html), but if you find
yourself confused by Hail expressions or unsure how to use Hail to solve your
problem, read through that.

## Start Small

The cloud has a reputation for easily burning lots of money. You don't want to
be the person who spent ten thousand dollars one night without thinking about
it. Luckily, it's easy to not be that person!

Always start small. For Hail, this means using a two worker dataproc cluster and
experimenting on a small fraction of the data. For genetic data, make sure your
scripts work on chromosome 22 (the smallest one) before you try running on the
entire genome! If you have a matrix table you can limit to chromosome 22 with
`filter_rows`. Hail will make sure not to load data for other chromosomes.

```
import hail as hl

mt = hl.read_matrix_table('gs://....')
mt = mt.filter_rows(mt.locus.contig == '22')
```

Hail's `hl.balding_nichols_model` creates a random genotype dataset with
configurable numbers of rows and columns. You can use these datasets for
experimentation.

As you'll see later, the smallest Hail cluster costs about 3 dollars per hour
(that's pretty cheap compared to your salary!). Each time you think you need to
double the size of your cluster ask yourself: am I prepared to spend twice as
much money per hour?

Before running a big job, re-read [Estimating Cost](#estimating-cost) and make a
cost estimate.

## Hail on the Cloud

This document will focus entirely on running Hail in Google Cloud Platform
(GCP). Hail can also run in other Cloud-based Spark clusters, on-premises Spark
clusters, and in "local mode" on a single machine.

The Hail Python library sends Hail jobs to a Spark cluster. A Spark cluster is a
collection of machines for processing very large amounts of data. Every Spark
cluster has one leader node and at least two worker nodes. Note, when discussing
Spark clusters we often use the term "node" to refer to a virtual machine.

Start a Hail-ready Spark cluster in GCP with `hailctl`:

```
hailctl dataproc start your-name-here-test
```

This creates a Google-managed Spark cluster named
`your-name-here-test`. Google's Spark cluster management service is called
Dataproc. Don't worry about cost yet! This cluster costs less than a dollar per
hour. Very cheap! It only has one leader node and two worker nodes.

Any cluster started by `hailctl` has a Jupyter notebook server running
on the leader node. Connect to this Jupyter notebook server:

```
hailctl dataproc connect your-name-here-test nb
```

As usual, you can add `--help` to any command to learn more about it. Your web
browser should now have opened a tab connected to the Jupyter notebook
server. Create a notebook and try running a simple Hail command like:

```
import hail as hl
mt = hl.balding_nichols_model(3, 1000000, 100)
mt = mt.annotate_rows(gt_stats = hl.agg.stats(mt.GT.n_alt_alleles()))
mt.gt_stats.show()
```

You should see some progress bars indicating the work underway on the
cluster. Your cluster should have twelve compute cores. Each worker has eight
cores, but Dataproc uses four cores of one worker node for its own purposes.

When you're finished with the cluster, shut it down:

```
hailctl dataproc stop your-name-here-test
```

### Labelling (Tagging)

Similarly to Google Cloud Storage buckets, Google Dataproc clusters may have "labels" applied to them. A label is a key and a value, both of which are strings. Google [constrains the kinds of strings](https://cloud.google.com/dataproc/docs/guides/creating-managing-labels#requirements) that can be used for keys and values. `hailctl` automatically adds a label with the key `creator` and the value set to your Google Cloud user account with banned characters replaced by underscores. For example, if your Google Cloud account is `sam@broadinstitute.org`, then your dataproc clusters will have the label: `creator=sam_broadinstitute_org`. Google Dataproc [automatically adds a few labels](https://cloud.google.com/dataproc/docs/guides/creating-managing-labels#automatically_applied_labels), at time of writing this included `goog-dataproc-cluster-name`, `goog-dataproc-cluster-location`, and `goog-dataproc-cluster-uuid`. 

### Long-running Jobs

At some point you'll want a job to run in the background while you are busy
going to meetings, opening and closing your laptop, probably losing and gaining
a WiFi connection. Jupyter Notebooks do not handle this gracefully. If you have
a long-running job, you can `submit` the job to the cluster:

```
hailctl dataproc submit your-name-here-test my-hail-script.py
```

### Reporting Errors

If you encounter an error in your pipeline, make sure to save the logs! The Hail
log file location is printed when Hail first runs a job. The file path is a file
path on the leader node of the cluster, so you must copy it off the leader node:

```
gcloud compute scp your-name-here-test-m:/path/to/hail.log .
```

Do this before shutting down the cluster!

If you need help, post the log and the *full* stack trace to
https://discuss.hail.is and someone from the development team will answer you
question as soon as possible. We often respond within the hour.

## Efficiently Using Hail

We focus on ways to reduce, control, and predict cost.

### Partitioning

When you `write` a Hail MatrixTable or Table, it is, effectively, stored as a folder
of *partitions*. Each partition can, in principle, be processed by a different
core of a cluster. If you have more cores than partitions, then some cores must
have no work to do because Hail cannot split the work on a MatrixTable or Table
into more pieces than there are partitions.

A partition must contain at least one row of a MatrixTable or Table and should
usually contain many more. Hail has some per-partition overhead, so we recommend
that your partitions are at least 128 megabytes in size.

### Preemptible Workers

Many Hail operations will succeed with preemptible workers. Specifically, any
operation which is entirely "row-parallel" (there is no sharing of information
across rows and, in particular, across partitions) will succeed with preemptible
workers. For example, counting the number of missing genotypes at every locus is
row-parallel:

```
mt.annotate_rows(n_missing = hl.agg.count_where(hl.is_missing(mt.GT)))
```

Counting the number of missing genotypes for every sample is *not* row-parallel:

```
mt.annotate_cols(n_missing = hl.agg.count_where(hl.is_missing(mt.GT)))
```

In general, `annotate_rows` is row-parallel and `annotate_cols` is not. A
non-exhaustive list of row-parallel Matrix Table operations:

- `mt.write`
- `hl.export_vcf(..., parallel=True)`
- `hl.linear_regression_rows`
- `mt.filter_rows`
- `mt.annotate_rows`

And a non-exhaustive list of row-parallel Table operations:

- `t.write`
- `t.export(..., parallel="separate_header")` or `t.export(..., parallel="header_per_shard")`
- `t.filter`
- `t.annotate`

An operation that is *not* row-parallel might still succeed with preemptible
workers. The probability of success is related to the [preemption
rate](https://cloud.google.com/compute/docs/instances/preemptible#preemption_selection),
time per partition, and cluster size. If the preemption rate is once every two
hours, a cluster of any size will likely not complete a *non*-row-parallel
operation that takes two or more hours. In general, do not use preemptible
machines for a long running non-row-parallel operation.

### Shuffles

Some non-row-parallel operations, like `annotate_cols` have limited dependencies
between rows and partitions. Other non-row-parallel operations require transfer
of data between each partition and every other partition. For example,
`key_rows_by` requires that the dataset is ordered by the new row keys. In general,
any one input-partition might need to send a different row to every other
output-partition. In analogy to the process of shuffling a deck of cards, these
operations are called "shuffles". Hail is designed to avoid shuffles when
possible.

Unlike other non-row-parallel operations (like `annotate_cols`), a shuffle will
*almost never* succeed on a cluster containing *even one* preemptible
worker. For this reason, we recommend using exclusively non-preemptible workers
when performing a shuffle.

### Cluster Size

Many Hail operations scale nearly linearly in core count. That means if you
double the cores, you nearly halve the wall-clock time (the time you wait for an
answer). Instead of ten cores working for one hour, twenty cores work for two hours,
each core doing half as much work. However, Hail cannot use more cores than
there are partitions of your dataset because Hail cannot split a partition into
pieces and give each piece to a different core.

Under only this constraint, the ideal cluster size is equal to the number of
partitions in our dataset. However, when cluster size is equal to the number of
partitions, we must pay the hourly cost of the entire cluster for the length of
the longest running partition. Each partition ideally takes the same amount of
time. In practice, datasets' partitions are not uniform in size, and
iterative operations (like logistic regression) take unpredictably varying
amounts of time per partition. To mitigate this effect we set cluster size to
some small integer fraction of the number of partitions. This small integer is
often three, four, or five.

### Dynamic Cluster Size

If your cluster is idle (because you are thinking, you switched to a smaller
dataset, etc.), you can set it to a smaller size:

```
hailctl dataproc modify --num-preemptible-workers N --num-workers M
```

If you have a series of row-parallel operations ([see
above](#preemptible-workers)) followed by a shuffle operation
(e.g. `group_rows_by`, `key_rows_by`), followed by more row-parallel operations,
you should consider dynamically changing your choice of preemptibility. You must
first split your pipeline into three steps:

`step1.py`
```
mt = hl.read_matrix_table(...)
# ... row-parallel operations ...
mt = mt.write("gs://bucket/temp1.mt")
```
`step2.py`
```
mt = hl.read_matrix_table("gs://bucket/temp1.mt")
# ... non-row-parallel operations ...
mt = mt.write("gs://bucket/temp2.mt")
```
`step3.py`
```
mt = hl.read_matrix_table("gs://bucket/temp2.mt")
# ... row-parallel operations ...
```

Scale the cluster up to the desired preemptible worker size.

```
hailctl dataproc modify --num-preemptible-workers N --num-workers 0
```

Submit the first job to the cluster.

```
hailctl dataproc submit step1.py
```

When `step1.py` is complete, switch to non-preemptible workers *before*
submitting `step2.py`:

```
hailctl dataproc modify --num-preemptible-workers 0 --num-workers N
hailctl dataproc submit step2.py
```

When `step2.py` is complete, switch back to preemptible workers and finally submit
`step3.py`:

```
hailctl dataproc modify --num-preemptible-workers N --num-workers 0
hailctl dataproc submit step3.py
```

This same strategy can be used for Hail pipelines that contain some operations
that need many workers and some operations that need few workers.

### Estimating Time and Cost

#### Estimating Time

Estimating the time and cost of a Hail operation is simple. Start a small
cluster and use `filter_rows` to read a small fraction of the data:

```
test_mt = mt.filter_rows(mt.locus.contig == '22')
print(mt.count_rows() / test_mt.count_rows())
```

Multiply the time spent computing results on this smaller dataset by the number
printed. This yields a reasonable expectation of the time to compute results on
the full dataset using a cluster of the same size.

#### Estimating Cost

Google charges by the core-hour, so we can convert so-called "wall clock time"
(time elapsed from starting the cluster to stopping the cluster) to
dollars-spent by multiplying it by the number of cores of each type and the
price per core per hour of each type. At time of writing, preemptible cores are
[0.01 dollars per core hour and non-preemptible cores are 0.0475 dollars per
core hour](https://cloud.google.com/compute/vm-instance-pricing). Moreover, each
core has an additional [0.01 dollar "dataproc premium"
fee](https://cloud.google.com/dataproc/pricing?hl=th). The cost of CPU cores for
a cluster with an 8-core leader node; two non-preemptible, 8-core workers; and
10 preemptible, 8-core workers running for 2 hours is:

```
2 * (2  * 8 * 0.0575 +  # non-preemptible workers
     10 * 8 * 0.02 +   # preemptible workers
     1  * 8 * 0.0575)   # master node
```

2.98 USD.

There are additional charges for persistent disk and SSDs. If your leader node
has 100 GB and your worker nodes have 40 GB each you can expect a modest
increase in cost, slightly less than a dollar. The cost per disk is prorated
from a per-month rate; at time of writing it is [0.04 USD per GB per
month](https://cloud.google.com/compute/disks-image-pricing#persistentdisk). SSDs
are more than four times as expensive.

In general, once you know the wall clock time of your job, you can enter your
cluster parameters into the [Google Cloud Pricing
Calculator](https://cloud.google.com/products/calculator/) and get a precise
estimate of cost using the latest prices.

#### Scaling And Cost

Hail aims to provide near-perfect scaling. This means that if you double the
number of partitions and cores the job will finish in half as much time *for the
same cost*. In practice, there is a slight increase in cost when using many more
cores. Generally, the increase in cost is worth the enhanced productivity of
you, the analyst, because your time is very expensive!
