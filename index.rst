:tocdepth: 1

.. Please do not modify tocdepth; will be fixed when a new Sphinx theme is shipped.

.. sectnum::

.. TODO: Delete the note below before merging new content to the main branch.

.. note::

   **This technote is not yet published.**

The Prompt Processing framework has long been a gray area in the overall Rubin Observatory architecture.
This document attempts to fill that hole by proposing a concrete design with an accompanying prototype implementation that should meet minimum requirements.
Additional features and optimizations that could be incorporated later are also described.

Requirements
============

The framework must execute Alert Production pipelines but also Commissioning pipelines and Calibration Products pipelines.

As the framework runs at the US Data Facility (USDF), it does not speak the SAL/DDS messages of the Summit systems.
All information from the Summit, including a ``next_visit`` event and image files, must be conveyed via other means.

The ``next_visit`` event is defined in LSE-72 requirement OCS-DM-COM-ICD-0031.
It specifies that advance notice of telescope pointings will be made available at least 20 seconds in advance of the first exposure of a visit, and that this will include exposure duration, number of exposures, shutter motion start time, and filter selection, as well as an indication of the image purpose.

Latencies and overheads must be minimized.
Since the Alert Production timeline is only 60 seconds from close of shutter on the last exposure of the visit, latencies on the order of seconds and potentially tenths of seconds do matter.
As much I/O and computation as possible should be done after ``next_visit`` arrives before the exposure images do.
Ingest of image metadata and starting pipelines should be as efficient as possible.

The framework should execute in parallel, with a minimum of one process per CCD/detector.
Amplifier-level or Source-level parallelism may be desirable for some functions.

It should handle single and multi-exposure visits, with the number of snaps per visit ideally known only at ``next_visit`` time.

It must be able to execute different pipelines on different images, at a minimum via configuration but ideally on a per-visit basis at ``next_visit`` time.

Since pipeline execution takes longer than the interval between visits, the framework must be able to have more than one visit in flight at a time.
As some visits may take longer than others due to crowding, artifacts, or other influences, it must be elastic to allow compute resources to be used as needed.

It must be resilient to failures of communication, aborted visits, loss of exposures, memory exhaustion, and other problems.
But occasional delays in executing or even outright failures to execute the pipeline for a visit are acceptable, as daytime catch-up processing is planned.


Initial Design
==============

The DDS ``next_visit`` event is translated to an HTTP POST containing the same information.
One POST message is issued per CCD.
That message is used to reserve a worker to process the entire visit.
The HTTP message handler does not return until the visit has been successfully processed or a failure has occurred, been identified, and reported.

Within the message handler, a separate messaging system is configured to receive notifications of exposure image arrival for the visit.
While waiting for those exposures, pipeline-specific calibration data and association data are preloaded into a newly-created handler-local repo.
As exposures arrive, they are ingested into the local repo, including copying of the image data.
When the last exposure has arrived, a pipeline selected by one or more fields in the ``next_visit`` event is executed via the Middleware ``pipetask`` command.

The pipeline is responsible for publication of all outputs including Alerts and quality metrics via non-Butler means (although Butler datasets in the handler-local repo can be used as intermediates).

Rationale
=========

Standard Web API infrastructure provides mechanisms for reliably, elastically handling HTTP verbs.
Google Cloud Run and AWS Fargate are "serverless" container deployment products that are designed for this, but Kubernetes with ReplicaSet deployments and auto-scaling can be used to do this on-premises.
Taking advantage of this infrastructure means that all of the resiliency and elasticity requirements, including coordination, distribution, failover, monitoring, etc., can be left to the infrastructure and not custom-built for the application.

Selecting this method of invoking the worker has limitations, however, as the payload to be executed is not a simple stateless one.
Instead, it needs to receive one or more additional events indicating image arrival.
This is best sent using a reliable, yet low-latency, mechanism other than HTTP, as listening for HTTP events in an HTTP event handler is tricky.
AMQP, MQTT, Redis, or possibly Kafka are potential mechanisms here, in addition to Google Pub/Sub or Amazon SQS.
The trigger for image arrival could be sent by the Camera Control System after it believes it has successfully published the image, or it could be sent by the storage system.

The four LSST Middleware interactions needed by the framework are:

 * On startup, initialize a handler-local Butler repo and copy globally-used calibrations.
 * On receipt of ``next_visit``, copy calibrations, templates, and potentially APDB data required by the pipeline into the handler-local repo.
 * On receipt of an image, ingest its metadata.
 * On receipt of the last image in a visit, trigger execution of the selected pipeline.

Creating a handler-local repo is necessary in order to execute unmodified pipelines, without requiring the writing of special code to work within Prompt Processing.
This repo (which is expected to be located on a RAM disk or fast local SSD) is used to ensure that all information needed by the pipeline is preloaded to the worker, avoiding bottlenecks during the critical time period after images have been received.
The code to preload data could be custom or it could be written as PipelineTasks and executed as an "initial step" pipeline, writing outputs to the handler-local repo.

It is assumed that all information needed to preload data and to identify the images to be processed is present in the ``next_visit`` event.
In particular, this means that the message handler must be able to know how many snap exposures are going to be received and what their URIs will be.
It is not feasible to poll a central repo to determine what images have arrived and been ingested, as this adds substantial latency, both for the central repo triggering and ingest and for the poll and copy to the handler-local repo.
A blocking-with-callback or other asynchronous API to the Butler could reduce the second source of latency, but it would not affect the first.

Copying the image from its publication point to the handler-local repo is assumed to be an irreducible cost.
By placing the image in permanent storage as soon as possible after retrieval from the Camera Data Acquisition system, the possibility of loss is minimized.
Previously, Prompt Processing was to receive a special, low-latency copy of the image (with firmware-based crosstalk correction), while the permanent archive would receive a non-crosstalk-corrected version after some delay (after the network burst caused by Prompt Processing).
Since the Camera is no longer doing firmware crosstalk correction, moving the copy to the destination end of the international link avoids congesting that link, places it in a high-bandwidth local network environment, and enables all potential USDF users of the images to receive the same low-latency service.

The Butler interactions are handled via the Python API for export and import, and, for simplicity of invocation, the command line tooling for ingest and pipeline execution.

Using a batch queue mechanism for firing off the pipeline execution is attractive in that the HTTP handler would not need to persist for the lifetime of the pipeline, and resilience and elasticity could be built into the batch system.
On the other hand, it poses serious issues with regard to latency and/or communications.
If the batch system were to be triggered where ``pipetask`` is invoked in the current design, the latency of batch submission, queueing, dequeueing, and execution startup would be in the critical path.
In addition, it would be difficult to preload data for the batch job, as it would presumably be on a different batch worker machine.
If the batch system were instead to be triggered immediately by the POST handler, latency would not be an issue.
Instead, it would be difficult for the batch job to be notified of image arrival.
Either a Butler poll (undesirable for reasons given above) or special messaging subscription code in the batch job would be necessary.
Neither of these seems to offer much of an advantage over the simpler design of including the preload and pipeline execution in the same handler process, as long as that handler is resilient and elastically scalable, which it is.


Prototype Implementation
========================

The prototype implementation runs in Google Cloud Platform as a convenient location to start services and containers.
One thing that is not so convenient in this environment, however, is access to useful calibration and test data.

The prototype uses Google Cloud Run as its container execution engine.
Since this is "serverless," there is no need to configure a Kubernetes cluster or allocate virtual machines in Google Compute Engine.
Cloud Run can autoscale, given parameters for how busy workers are, and it can reserve a minimum number of nodes to ensure that a new visit can be triggered at any time.
It can cache state between visits using an in-memory ``/tmp``, but containers need to be able to cold start.
Liveness probes can ensure that messages are not sent to a container before it has finished setting up.
Each instrument is run as a separate Cloud Run service so that they do not interfere with each other.
All of these features can be replicated in Kubernetes with some extra management overhead.

The HTTPS POST message is provided by Google Pub/Sub.
It is triggered by a small Python script that also uploads image files to Google Cloud Storage (GCS) object store on an appropriate cadence.
The prototype uses a simple Flask app to accept the POST message.
Each worker can have a different detector from visit to visit.

The object store is organized as ``instrument/detector/group/snap/filename``.
Instruments could be stored in separate buckets, but for now only one is used.
Placing the detector earlier in the object identifier provides a wider distribution of prefixes, enabling higher bandwidth to storage.
Placing the detector, group, and snap in the identifier allows them to be retrieved for matching against the worker's expectations.

Notifications of GCS object arrival are also emitted through Pub/Sub, but they cannot be gatewayed to HTTP.
It's not practical to have a channel per detector + visit; channel setup overhead is too great.
It might be practical to have a channel per detector, but it's simpler to have a single channel per instrument that receives all detector image notifications.
In Pub/Sub, a subscriber to a subscription removes messages from the queue, so no other subscriber will see the same message unless its processing fails.
In this case, all workers for a given visit need to see all messages to the same per-instrument channel so that time isn't wasted sending a message to a worker that cannot handle it.
Since there are multiple visits in flight at any given time, each worker handles only some of the visits.
To minimize the old notifications that a worker might see, the subscription is created dynamically upon ``next_visit``, immediately after receipt of the POST.
The worker also checks to see if any snap images arrived before the subscription could be created (if ``next_visit`` was not sent early enough and if the exposure time is short, such as with bias images).

The POST message contains all information needed to start preparing the handler-local repo for the visit.
The repo preparation will depend on selected pipeline; it could be chosen based on next_visit information (but is currently fixed).

After the repo is prepared, the prototype begins waiting for snaps.
It blocks waiting for one or more Pub/Sub messages.
There should be one snap image notification per detector, so the API call allows for a maximum of 189 + 8 + 8 messages corresponding to the science, guide, and (half) wavefront sensors of LSSTCam, expecting that bulk message notification will be more efficient than one-at-a-time.
The list of notifications is searched for the expected instrument/detector/group for this worker.
If present, the image is ingested.
All received messages are acknowledged to ensure that the subscription queue is cleared out.

When all snaps have arrived, the pipeline, as chosen by next_visit, is executed.
Upon successful completion, the handler returns a 200 status from Flask.
Any exceptions or errors, including timeouts from failing to receive image notifications, are handled by a separate error handler that logs the problem and returns a 500 status.
At the end of the visit, the dynamic Pub/Sub subscription is deleted.

If the Cloud Run worker takes too long to respond to the initial ``next_visit`` POST, Cloud Run itself will timeout and restart the worker, ensuring that the system is resilient to algorithmic lockups or failures to receive images.


Future work
===========

Middleware interface
--------------------

Using the Python API would be slightly more efficient than using ``subprocess`` to start a command-line tool.
The lower-level Python APIs for ingesting raw data and executing pipelines are harder to use, however.

Looking for already-present calibration images in the handler-local repo would save re-copying them.
Conversely, cleaning up outdated images and calibrations from the handler-local repo could be desirable, although workers can also be cold-started from scratch at any time.

Selecting which calibration datasets are needed based on the ``kind`` attribute of the visit would be desirable.

MinIO/Ceph
----------

Both of these object stores are candidates for deployment on-premises.
Both have custom APIs for notifying on successful object ``put`` that would replace the Google Pub/Sub message used in the prototype.
They can publish to several kinds of messaging infrastructures; one would have to be chosen.

Message handling
----------------

For Google Pub/Sub, it might be better to have a single subscription per worker, ignoring all messages that are unexpected (e.g. for older visits not processed by this worker).
The messaging infrastructure will have to be replaced for on-premises usage, but all systems should have similar subscription mechanisms.
There may be better channel filtering available to simplify this aspect of the prototype.

Affinity
--------

It would be much more efficient to send images from a single detector to same the same worker each visit.
This would allow caching and reuse of calibration information.
Cloud Run has some affinity controls, but it's not clear that they would be sufficient for this.
A custom on-premises ingress could likely do better.

Fanout
------

The current upload script sends a separate ``next_visit`` message for each detector.
In actual usage, a single ``next_visit`` message would likely be sent from the Summit to the USDF.
A USDF-based server (potentially the ingress mentioned above) would then translate this into multiple POSTs to the back-end worker infrastructure.

Output handling
---------------

It could be possible to build some standard output handling methods into the worker.
These could include retrieving certain products from the output collection in the handler-local repo and transmitting them elsewhere.
In particular, telemetry back to the Summit is specified as going over Kafka.
But it is not clear if this is beneficial over having this publication in Tasks within the pipeline.

Autoscaling
-----------

Configuring the auto-scaling properly to expand when a visit's processing runs long may take some tuning.
Ideally, a hot spare set of nodes large enough for a visit should be on standby at all times.

.. .. rubric:: References

.. Make in-text citations with: :cite:`bibkey`.

.. .. bibliography:: local.bib lsstbib/books.bib lsstbib/lsst.bib lsstbib/lsst-dm.bib lsstbib/refs.bib lsstbib/refs_ads.bib
..    :style: lsst_aa
