Background
==========

As of January 2015, HTC's has launched the cloud service infrastructure, as well as [One
Gallery](https://play.google.com/store/apps/details?id=com.aiqidii.mercury), an gallery
app that aggregates all your cloud photos. The service was gradually launched to worldwide.

While the service was running smoothly in [Google Compute
Engine](https://cloud.google.com/compute/) (GCE), we observed high instance cost. In order to
achieve good user experience, whenever user uploads photos to Facebook, Facebook sends
webhook to our backend which triggers a photo crawl job immediately. Since the number of
instances of each cluster was configured with a fixed value, we can only assume the
maximum load running all the time.

This was waste of resources. Assuming maximum load all the time implies huge amount of
idle computing resource. This however can be resolved by periodically checking current
load to adjust cluster size accordingly.

On the other hand, HTC's cloud infrastructure was equipped with metric pipeline, that
aggregated hundreds of thousands code-level metrics to
[Elasticsearch](https://www.elastic.co/products/elasticsearch) for query. It already
served as primary operational debugging tool, and we believed them to be great indication
to scale clusters.

Possible Solutions 
==================

At that time, GCE is equipped with a simple auto scale service. You set the target CPU and
min/max number of instances, GCE would scale the instance to reach the target. It's
good that:

- The auto scaler's input is very simple. All you need is 3 parameters.
- It's a managed service. No need to worry about operation.

While there are a few things we didn't like:

- It's a black-box with no logs. There's no way to know how it works. 
- You can only scale the cluster by CPU.

The second one is crucial since we need more input factor than just CPU. To explain why,
our service consists of the following types of clusters: API frontend, in-house key value
store, and job queue workers. While the first two types can be scaled by CPU, job queue
workers can't. A job queue worker's capability can be bounded by memory or network IO. For
example notification cluster sending GCM to Android devices is bounded by network IO. In
that case CPU-based scaling method won't work since CPU is always low. Therefore we
decided to design our own solution.

Design
======

Metric Selection
----------------

For most clusters, CPU is a great scaling indicator. This however doesn't hold for job
queue workers, since a worker can be bounded by memory or network IO. The goal for job
queue workers is to complete the jobs as soon as possible. If there are more jobs than
workers can handle, it'll store overflow jobs into a queue until someone is free. An
representative indicator is number of unfinished jobs. If there are much more jobs than
current workers can handle, we scale up the cluster.

Implementation
--------------

The implementation is simply a cron server written in Go. It periodically checks
target metric and current cluster size and decide whether to scale a cluster. [PID
control algorithm](https://en.wikipedia.org/wiki/PID_controller) is used here to decide
proper instance number. If the algorithm decides to adjust cluster size, it sends [gcloud
instance group resize
commands](https://cloud.google.com/sdk/gcloud/reference/compute/instance-groups/managed/resize)
to inform Google Compute Engine.

Auto scale engine itself is a cluster hosted in GCE. To prevent single point of failure, 3
instances are deployed while one served as master and the rest as slaves. Master
periodically runs scaling algorithm while slaves check for master health. If master dies,
one slave will take place of master role.
 
Deployment & Outages
====================

The scaling algorithm is tested in staging server before production. The first target
cluster was job queue workers due to simplicity. Cluster load was manually generated and
can be adjusted dynamically to verify whether the algorithm worked as expected. Once the
performance was proved in staging environment, the same algorithm is ported to control
production server.

There are a few outages at the beginning due to cron server not able to send send scaling
command to GCE. This is because GCE related OAuth was inserted into cron server by
operation engineer manually, and the credential got lost after server upgrade. This was
resolved by automate the OAuth insertion process.

Fine tunes & Improvement
========================
After the service is deployed, we found job queue workers weren't scaled as expected.
They were often scaled up cluster while many workers remained idle. Our in-house job queue
is designed that jobs of the same concurrent key are executed by the same worker, which
makes unbalanced worker load. Therefore, we need a more representative metric to reflect
cluster load.

After some research, we found the load of least busy worker is a better indicator, instead
of using average worker load. When a new job comes, it'll put into the least busy worker's
queue. If the least busy worker's load is high, it's really time to scale up. 

Also to prevent system oscillation, we set a cool-down time before the next scaling
command can take effect. It was originally set to 10 minutes, which turned out not enough
to prevent oscillation. Traditional tuning method suggests reduce the scaling multiplier.
We didn't follow because:

- Our test environment could not always reproduce oscillation, which prolongs tuning process.
- Reducing multiplier may cause undershoot. When a spike comes it can't catch up with
  the load.

Instead we prolong the cool down time before shrinking a cluster. We smooth the overshoot
effect with the price of extra machine cost when a spike comes. 

Result
======
By adopting auto scaling to our instances, we saved 70% of GCE instance cost.
// TODO: illustrate by counter.

Future work
===========
### Reduce minimum cluster size from 3 to 2
Before auto scaling is adopted, the minimum number of instance of a cluster is set as
three. If it was two, someone needs to wake up at midnight restarting instance to prevent
single point of failure. Now auto scale algorithm periodically checks for system health
and restarts instance when necessary, so need to keep the redundant third instance.

### Proactive auto scale
Our algorithm is passive that we scale based on current load.  However, the load of the
same day of a week is similar. Can we predict load by previous week for better scaling the
cluster? Netflix has done [similar
work](http://techblog.netflix.com/search/label/autoscaling).
