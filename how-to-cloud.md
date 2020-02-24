# How To Cloud

## Prerequisites

You'll need a basic understanding of GNU/Linux and feel comfortable using a
terminal / shell / command-line. If you're unfamiliar with these things, you
might find [Digital Ocean's Linux tutorials
helpful](https://www.digitalocean.com/community/tutorial_series/getting-started-with-linux),
particularly the [introduction to the linux
terminal](https://www.digitalocean.com/community/tutorials/an-introduction-to-the-linux-terminal).

## What is the Cloud?

"Cloud" is a vague term used in many contexts with many meanings. It roughly
refers to the computer services provided by Google (Google Compute Platform
(GCP)), Amazon (Amazon Web Services), Microsoft (Microsoft Azure), Alibaba
(Alibaba Cloud), and others.

The services provided by different cloud providers varies, but they almost
always include: virtual machines (e.g. Amazon EC2, Google Compute Engine) and
bucket storage (e.g. Amazon S3, Google Cloud Storage). A virtual machine appears
to you as a normal computer, like your laptop. In fact, you are sharing a
computer in a secure, isolated manner. In the cloud, you usually pay a flat
per-second, per-core rate.

Bucket storage stores files independently of any virtual machine. In general, a
file is identified by a bucket name and a unique key. Google Cloud Storage (GCS,
also known as GS or by its URI scheme `gs://`), is more flexible: a file is
identified by a bucket name and a hierarchical path:

```
gs://bucket/folder1/folder2/folder3/file
```

However, beware: most bucket storage systems, including GCS, pervasively *do
not* support traditional hierarchical features. For example, "setting
permissions on a folder" is simulated by recursively setting permissions on
every object in the folder, this takes a lot longer than running a single
operation on a folder on a normal file system! Every object in the folder will be treated as an individual link/object independent of the directories they are stored in. 

The rest of this document assumes you're using Google Cloud Platform, the
more common cloud provider at the Broad Institute.

### Interacting with GCP

GCP provides a [web user interface](https://console.cloud.google.com) and a
command line utility. This document will give examples using the two command line
utilities: `gcloud` and `gsutil`. Install both of them by following [Google's
instructions](https://cloud.google.com/sdk/docs/downloads-interactive). After it
is installed run

```
gcloud auth login
```

and follow the web browser prompts.

### GCP Projects

GCP resources (such as virtual machines or buckets) are organized into
projects. You can check which project is active at any time by running

```
gcloud config get-value project
```

By the way, if you ever want to learn more about a command, use `--help`, for
example:

```
gcloud config get-value --help
```

If the documentation is too large to fit in your terminal window (it almost
always is), `gcloud` will display the documentation through a tool called
[`less`](https://en.wikipedia.org/wiki/Less_(Unix)).

### Virtual Machines in GCP

GCP provides many virtual machine configurations. A full list, including prices,
is available [in the GCP
docs](https://cloud.google.com/compute/vm-instance-pricing). If you don't know
what to start with, use the `n1-standard` series of machine configurations. An
`n1-standard-8` is a virtual machine with 8 "vCores", 30 GB of RAM, and a
configurable amount of hard drive space.

In addition to machine configuration, every GCP virtual machine is either
["preemptible" or
"non-preemptible"](https://cloud.google.com/compute/docs/instances/preemptible). A
preemptible VM is roughly one-fifth the per-hour cost of a non-preemptible
VM. In return for this cheap price, Google reserves the right to shut the VM
down with seconds of notice. This would happen in the event of other users who have requested a non-preemptible machine and Google assigns the preemptible machine that you created to said user.  Generally, if you're starting a VM yourself for
interactive analysis, you need a non-preemptible VM. Some applications,
including [Hail](https://hail.is), can use preemptible VMs.

A small non-preemptible machine is really cheap, at time of writing it is 5
cents per hour! Practice starting one named with your name suffixed by `-test`:

```
gcloud compute instances create your-name-here-test --machine-type n1-standard-1
```

This will create an `n1-standard-1` instance. Connect to it by running:

```
gcloud compute ssh your-name-here-test
```

That starts a shell session on the virtual machine. You can end that connection
by pressing control-d on your keyboard or typing `exit` and hitting enter.

From your laptop, you can run

```
gcloud compute scp your-name-here-test:/path/on/vm/to/file .
```

To copy a file located at `/path/on/vm/to/file` to the current directory on your
laptop (run `pwd` to figure out your current directory). To send a file to the
VM swap the argument order:

```
gcloud compute scp /path/to/file your-name-here-test:/path/to/destination/file
```

When you're done you have two choices: *stop* and *delete*. If you *stop* a machine,
you can *start* it again later. Any running processes will have been terminated,
but the hard drive, all installed programs, and any copied files are still
present. In return for this, you continue to pay for the hard drive!

I lied earlier. As mentioned, running `n1-standard-1` VM is actually costing 5 cents per
hour (for the core). With the cost of the disk, it would add 4 cents *per month* (for the disk). That's roughly
six-thousandths of a cent per hour.

Nonetheless, it is good practice to *delete* a virtual machine if you are
certain you will never need to use the same programs and files again:

```
gcloud compute instances delete your-name-here-test
```

### Cloud Storage in GCP

A file in Google Cloud Storage is identified by a bucket and a path within the
bucket. Bucket names are *globally unique*. That means that you must choose a
bucket name that no other person has ever used on GCS in the history. In
practice, this is not very hard. You can create a bucket with:

```
gsutil mb gs://bucket-name
```

By default, buckets are private to the project, so you can safely store
sensitive data there. Some buckets are public, like `gs://hail-1kg/`. You can
list the files in a bucket:

```
gsutil ls gs://hail-1kg
```

You can ask how much data is stored in a folder:

```
gsutil du -sh gs://hail-1kg/1kg-all.mt
```

Note you must omit the trailing slash if you want the sum total size of the
folder. If you include the slash, it will always print `0B`. The cost to store
one gigabyte of data for one month in Google Cloud Storage at the time of
writing this document is 0.026 dollars. [Check the latest
price](https://cloud.google.com/storage/pricing) for "standard storage": that is
the kind of storage you'll primarily use.


### Does a vCore correspond to a physical CPU?

At time of writing, a vCore in GCP corresponded to a [hardware
hyper-thread](https://en.wikipedia.org/wiki/Hyper-threading). Generally, this
should not matter to you; however, if you're using extremely performance
sensitive libraries, such as the Intel MKL (Matrix Kernel Library), you will
notice that 2 vCores combined have performance comparable to 1 physical CPU core
(e.g. a core in the Broad cluster).

# Resources

- Julia Evans' [comics about SSH, cURL and others](https://jvns.ca/blog/2019/02/10/a-few-networking-tool-comics/)
