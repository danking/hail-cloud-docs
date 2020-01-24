# How To Cloud With Hail

## Prerequisites

Read the How to Cloud document first.

You should already have Hail installed on your laptop, see the [Hail
installation documentation](https://hail.is/docs/0.2/getting_started.html). You
should already have completed the [GWAS
Tutorial](https://hail.is/docs/0.2/tutorials-landing.html) on your laptop. You
do not need to have read the
[Overview](https://hail.is/docs/0.2/overview/index.html), but if you find
yourself confused by Hail expressions or unsure how to use Hail to solve your
problem, read through that.

## Hail on the Cloud

This document will focus entirely on running Hail in GCP, though it is possible
to run Hail on any Spark cluster whether Cloud-based or on-premises.

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
hour, very cheap! It only has one leader node and two worker nodes.

Any cluster started by `hailctl` has a Jupiter notebook server running
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
cluster. Your cluster should have 12 compute cores. Why not 16? Dataproc uses 4
cores of one worker node for its own purposes.

When you're finished with the cluster, shut it down:

```
hailctl dataproc stop your-name-here-test
```

### Long-running Jobs

At some point you'll want a job to run in the background while you are busy
going to meetings, opening and closing your laptop, probably losing and gaining
a WiFi connection. Jupyter Notebooks do not handle this gracefully. If you have
a long-running job, you can `submit` the job to the cluster:

```
hailctl dataproc submit your-name-here-test my-hail-script.py
```

### Reporting Errors

If you encounter an error in your pipeline, make sure to save the logs! The hail
log file location is printed when Hail first runs a job. The file path is a file
path on the leader node of the cluster, so you must copy it off the leader node:

```
gcloud compute scp your-name-here-test-m:/path/to/hail.log .
```

Do this before shutting down the cluster!

If you need help, post the log and the *full* stack trace to
https://discuss.hail.is and someone from the development team will answer you
question as soon as possible. We often respond within the hour.

# Partitioning

????

# Efficiently Using Hail

We focus primarily on monetary efficiency.

## Preemptible Workers

Many hail operations will succeed with preemptible workers. Specifically, any
operation which is entirely "row-parallel" (there is no sharing of information
across rows) will succeed with preemptible workers. For example, counting the
number of missing genotypes at every locus is row-parallel:

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

And non-exhaustive list of row-parallel Table operations:

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

## Cluster Size

Many Hail operations scale nearly linearly in core count. That means if you
double the cores, you nearly halve the wall-clock time (the time you wait for an
answer). Instead of ten cores working for one hour, twenty cores work for two hours,
each core doing half as much work. However, Hail cannot use more cores than
there are partitions of our dataset because Hail cannot split a partition into
pieces and give each piece to a different core.

Under only this constraint, the ideal cluster size is equal to the number of
partitions in our dataset. However, when cluster size is equal to the number of
partitions, we must pay the hourly cost of the entire cluster for the length of
the longest running partition. If every partition took the same amount of time,
this would be OK. In practice, datasets are partitions are not uniform in size
and iterative operations (like logistic regression) take unpredictably varying
amounts of time per partition. To mitigate this effect we set cluster size to
some small integer fraction of partition size. This small integer is often
three, four, or five.

## Dynamic Cluster Size

If your cluster is idle (because you are thinking, you switched to a smaller
dataset, etc.), you can set it to a smaller size:

```
hailctl dataproc modify --num-preemptible-workers N --num-workers M
```

If you have a series of row-parallel operations (see above) followed by a
non-row-parallel operation (e.g. `group_rows_by`), followed by more row-parallel
operations, you should consider dynamically changing your choice of
preemptibility. You must first split your pipeline into three steps:

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

# Resources
