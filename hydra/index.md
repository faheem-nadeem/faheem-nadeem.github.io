# Hydra—Kubernetes based Dataset PubSub and Volume Management System

In the previous articles of this [advent series](https://0x65.dev),  we
described the [architecture of our search engine](https://0x65.dev/blog/2019-12-14/the-architecture-of-a-large-scale-web-search-engine-circa-2019.html)
and highlighted technologies which help us [serve search results](https://cliqz.com/#channel=blog).

After many iterations on our architecture, we are finally on a microservice architecture pattern,
driven through a container orchestration platform: Kubernetes. This brought us
many engineering improvements, but it also has introduced new challenges,
such as, the ability to perform a dataset propogation via Pub-Sub, along with volume management for microservices. Let us illustrate this point further by example.

## Scenario---Working with Services and Datasets

* Let us say, a team of engineers are developing a service, which has a dependency
on dataset X, Y and Z.
* The service requires that periodically, these datasets are updated (*Data Updates*).
* Typically, a dataset will be versioned. Hence, there is the need for a versioning scheme, given the frequency of updates, time-stamps are a good option. e.g.: 2019-12-15T00-00-00 i.e. YYYY-MM-DDThh-mm-ss (*Data Versioning*).
* For the service to present a holistic view of the system the datasets in a given version can have interdependencies (*Service-Data State Management*).
* Newer datasets can be published periodically through an automated pipeline and artifacts stored in an object store like S3.
* One then needs to **keep track** of when these datasets finish building and thereby **notify** the downstream service of their availability
 (keeping in mind that all interdependencies must be satisfied).
* Once all three datasets are available, we **push the data to instances** where
services are running (*Data provisioning on Volumes*).
* The data is **downloaded behind the scenes**, without affecting the downstream service.
* Eventually, the service gets **notified** of successful download for all the
datasets on the instance.
* Service intrinsically **identifies** and **verifies** this change and
**shifts** to new datasets incrementally avoiding downtime..

This is a very common pattern for services, which rely on frequently updating
immutable datasets. If this is a one-off scenario, for instance a prototype or
a service whose data updates once every 3 months, then you are probably better
off doing all steps necessary by hand or by writing a custom automation scripts.
But, if this is a regular scenario like in the real world---for example when
dealing with multiple services, with a multitude of datasets, each having
their own periodicity---it may lead to potentially severe issues pretty quickly.

Moreover, there can be two major failure scenarios:

1. Pods can be evicted (Kubernetes Specific).
2. Instances may be lost (General Scenario).

Hence, *centrally automating dataset propagation, in a high failure system has its challenges but brings in considerable value.*

In this post, we will briefly introduce a solution built in-house, named
*Hydra*, which provides a generic solution to the use-case mentioned above. We
describe the need for such a system in our organization.

On a high level, Hydra is composed of a set of components which,

1. Provide a dataset pub-sub mechanism.
2. Maintain a global state.
3. Manage datasets on volumes for services running within a Kubernetes cluster.

At its core, Hydra is able to tap into the Kubernetes service registry for discovery and observability of running services.

## Managing lifecycle of datasets (models and indexes) on Instances

One of the core requirements of our search engine is fast access to different types
of datasets which can have both static or dynamic composition. The dynamism of
datasets can be described w.r.t their update frequency (some update real-time,
some based on a triggered event and some based on a pre-configured cron job setting).

The dataset representation can be a machine learned model (Tensorflow, Keras,
Scikit-Learn, etc.), a [Granne](https://github.com/granne/granne) Index
(Approximate Nearest Neighbor Index), [qpick](https://github.com/dncc/qpick)
Index, a key-value compiled dataset ([Keyvi](https://github.com/KeyviDev/keyvi),
 RocksDB, etc.), a service configuration, or an artifact from a batch job. The size
may range from a couple of megabytes to several terabytes.

For the purpose of serving search results, the **medium and placement** of these indexes and
models is also important. Datasets may be located on spinning magnetic disks,
SSDs or on RAM (ram-disks). This placement differs from service to service,
with varying degree of Service Level Objectives (SLO) requirements (access
patterns, latency guarantees, replication, etc.) In the context of cloud
provisioning, the storage medium can also differ. For some models which do not
have strict latency requirements, we choose
[Elastic Block Storage](https://aws.amazon.com/ebs/) (EBS) but for others where
read latency needs to be minimized and IOPS can be a determining factor to
achieve SLOs, we employ local volumes based on raided SSDs using the [NVME](https://en.wikipedia.org/wiki/Non-volatile_memory) protocol.
For services requiring even more guarantees we rely on large ram-disks.

Every microservice in the critical request path, as explained in our previous
article[^cliqzarch], can have a persistent layer composed of the aforementioned
datasets. In order to provide fresh results and to support graceful degradation
on failures we need to have a solid mechanism for dataset propagation.

Other use cases may include: data exploration while developing new features,
running A/B tests or modifying configuration of running services on the fly.

Finally, there is another important reason for solid dataset propagation.
If it does not exist, every team working on a feature-related data will
have to reinvent the wheel. We have experienced first hand that multiple teams
come up with different solutions for downloading and keeping state of their
services, this tooling, however, was tightly coupled with the system,
making re-usability impossible. A common system is needed to save people the tedious task of "getting ready".

## Introducing Hydra

These requirements presented an opportunity to centralize said effort and bake
it with infrastructure support. A centralized effort also brings in an
opportunity to build better observability and fault detection tools. Knowing the
actual placement of the data as well as the ability to quickly
change it, is vital for a working system, where the data is served from
and facilitates quick changes. The service which
solves these challenges for us is called *Hydra*.

The core constituents of Hydra are as follows:

* It provides a generic pub-sub approach to handling datasets at scale and
aids a service to publish or subscribe to versioned datasets.
* Introduces a concept of channels, publishers and consumers.
* A **channel** contains an ordered list of releases where the head pointer
points to the latest release subscribers (services) would be interested in.
*Channel states* are managed by a **Global Manager** and metadata for the same is
maintained in a highly available **[Consul](https://www.consul.io/)**.
* **Publishers** can update a channel with a new release (HTTP API) and
consumers will be notified of new changes.
* Data update diffs are propagated from *Global Manager* to downstream
agents named *Docto* running as [deamon-sets](https://kubernetes.io/docs/concepts/workloads/controllers/daemonset/) which can ensure a global state
(downloading of data from S3 and cleanup) in a reconciliation loop.
* On successful availability of a release on an instance, consumers are
incrementally notified of newer datasets, which are then rolled out. This is done via an additional component named *AppMan* which is responsible for this task. This allows
services to be agnostic of Hydra's workings.
* Newer datasets can be rolled out using the [readiness probe](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/) feature from Kubernetes. Whereby given a service is in high availability, rolling data deployments can be done by draining traffic from a pod using readiness probes.
* In scenarios of **replication**, this is done *one by one* for a given service,
thereby reducing downtime.
* The datasets are **immutable** in nature i.e. a single release represents a
complete view of data and incurs no dependency on previous versioned data
releases in the same channel.

## Working with Hydra---End-to-End Use Case

### Publishing State of a Release

A user or a service can post the dataset source paths (composed within a release)
using client-side libraries / CLI and its storage placement as required with the help of a data template, provided as part of Hydra's client CLI tool. This complete information
is posted via the HTTP API to a Global Manager Server.

```python
DISK = 0
RAM = 1

DATA_TEMPLATE = {
    "dataset_1": {
        "SourcePath": f"s3://some-bucket/some-data-set-1/{VERSION_PLACEHOLDER}/file1.xyz",
        "Placement": DISK
    },
    "dataset_2": {
        "SourcePath": f"s3://some-bucket/some-data-set-2/{VERSION_PLACEHOLDER}/file2.xyz",
        "Placement": RAM
    },
}
```

Where, `SourcePath` is the original S3 bucket path where the data is / will be available, and `Placement` describes where the data must be eventually provisioned on the instance.

Some extra information like Environment, Release and Namespace are also
required by the Hydra CLI client. It allows Hydra to formulate channel identity
and scoping and is stored as metadata for respective channel. This can be propagated using special flags:

```bash
--env: Environment describes which cluster to target
--release: Release name of service relevant to dataset template
-n: Namespace of deployed service
```

The hydra client submits this data template with the `VERSION_PLACEHOLDER`
 computed based on two factors:

1. Identifying most recent datasets available on S3.
2. If datasets should have the same timestamps, then it verifies the same before
proceeding ahead.

We assume that once datasets are available on S3 they have been already verified
for integrity and are fit to serve. These checks are done as part of the
data-pipeline's post processing step.

Channels can have retention policies on the number of releases to keep.
Also, you can configure the number of actual releases to keep on instances.
The default value is 2. This means that apart from the newly published release,
the previous dataset is also preserved on instances. All older releases can
be cleaned up based on retention policies to improve disk utilization.

### Hydra's Global Manager & Data Download  (S3 -> Instance)

Once a new data release is published to a channel, the global manager is
able to correlate consumers based on discovery from the Kubernetes
service registry or explicit registration.

All the instances in the cluster run an agent as a daemon-set which only
communicates with the Global manager and is wholly responsible to download
the models from S3 to instance. Thus the global manager communicates with this
daemon application and sends information about which datasets to download.
The daemon verifies the requests, identifies the placement, checks if there
is enough storage available on the instance and starts the download. It
constantly communicates the progress of the download with Global Manager
and notifies a `SUCCESS` upon its completion.

### Application Dataset Reload

Next, the Global Manager notifies the consumer through client-side
library interfaces on availability of newer release datasets. This is done
incrementally to avoid downtimes. This initiates the process, where at
first readiness probes are used to drain traffic, then either by
using a built-in mechanism or a service restart, new data is reloaded
for the service, and readiness probes are turned back ON, to serve live traffic.

If there are several replica pods behind a service, the Global manager
performs the reload one by one, making sure each pod comes back up before
starting the data reloads in other pods of the same service.

**Notes:**

1. In development settings, for non critical data services we employ a
  downscaler strategy which downscales deployments to zero during non working hours.
  This constitutes an uptime during the following times: `Mon-Fri 08:00-19:30 Europe/Berlin`.

    This means that not only the container, but instance with data is downscaled.
    But, since hydra is autonomous, we can quickly prepare the instance with the
    data when the deployment is scaled up again on a newer instance. This works
    great for us, as we save some costs by not spinning high capacity instances
    (a necessary requirement for hosting search indexes) in development setting
    during non-working hours.
2. In a special case where RAM is the preferred placement medium, given the fact
that they are usually a scarce and expensive resource, we make sure that they
are also copied over to disk if available for faster load times to RAMDISK.
We only keep a single version on RAM for that dataset. Due to locking on older
datasets or limited availability of space on-ramdisk services can be drained
and shifted to newer releases.

## Future Work

Hydra shares a lot of its DNA with project [Gutenberg](https://medium.com/netflix-techblog/how-netflix-microservices-tackle-dataset-pub-sub-4a068adcc9a), which was announced
two months ago.

We will keep an eye on Gutenberg's development since both systems attempt
to solve the very same problems. It is just natural that different teams
reach similar conclusions when presented with the same problem.
Nonetheless, it is always nice to get external validation of the decision taken.

We have been developing Hydra for the last 2 years and gradually integrating it into
our production systems. However, it is far from a finished project. A peak on the project's future timeline:

* Better control over Volume discovery, provisioning, budgeting and isolation.
* Multi language support for client side libraries. Including Rust and Go. Currently we only provide Python.
* More scoping options such as region and pinning releases to a specific version.
* Encryption and access control to improve security footprint and avoid disaster scenarios.
* Better incremental rollouts by introducing new strategies like disruption budgets and surge protections.
* Improvements in observability for service owners.
* Better channel retention strategies, such as *time based retention*.

## Conclusion

At Cliqz, Hydra runs in production and handles data for a variety of data-hungry
services deployed in our search stack. It drives part of our search index
which is updated weekly (window-based) as part of our multi-tier lambda architecture[^cliqzarch].

Through this post, we wanted to highlight the common challenges in data
management and how we solve them using Hydra.
We plan to share more details in future posts.

Hydra is still under active development and will eventually be open sourced.
We believe, that the Kubernetes and ML community can greatly benefit from
an end-to-end model management versioning and propagation system.

## Footnotes and Remarks

[^cliqzarch]: [The Architecture of a Large-Scale Web Search Engine, circa 2019](https://0x65.dev/blog/2019-12-14/the-architecture-of-a-large-scale-web-search-engine-circa-2019.html).

