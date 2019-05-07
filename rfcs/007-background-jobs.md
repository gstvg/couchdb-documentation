---
name: Formal RFC
about: Submit a formal Request For Comments for consideration by the team.
title: 'Background jobs with FoundationDB'
labels: rfc, discussion
assignees: ''

---

[NOTE]: # ( ^^ Provide a general summary of the RFC in the title above. ^^ )

# Introduction

This document describes a data model, implementation, and an API for running
CouchDB background jobs with FoundationDB.

## Abstract

CouchDB background jobs are used for things like index building, replication
and couch-peruser processing. We present a generalized model which allows
creation, running, and monitoring of these jobs.

The document starts with a description of the framework API in Erlang
pseudo-code, then we show the data model, followed by the implementation
details.

## Requirements Language

[NOTE]: # ( Do not alter the section below. Follow its instructions. )

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT",
"SHOULD", "SHOULD NOT", "RECOMMENDED",  "MAY", and "OPTIONAL" in this
document are to be interpreted as described in
[RFC 2119](https://www.rfc-editor.org/rfc/rfc2119.txt).

## Terminology

---

`Job`: A unit of work, identified by a `JobId` and also having a `Type`.

`Worker` : A language-specific execution unit that runs the job. Could be an
Erlang process, a thread, or just a function.

`Job table`: An FDB subspace holding the list of jobs.

`Pending job`: A job that is waiting to run.

`Pending queue` : A queue of pending jobs ordered by priority.

`Running job`: A job which is currently executing. To be considered "running"
the worker must periodically update the job's state in the global job table.

`Priority`: A job's priority specifies its order in the pending queue. Priority
can by any term that can be encoded as a key in the FoundationDB's tuple layer. The
exact value of `Priority` is job type specific. It MAY be a rough timestamp, a
`Sequence`, a list of tags, etc.

`Job re-submission` : Re-submitting a job means putting a previously running
job back into the pending queue.

`Activity monitor` : Functionality implemented by the framework which checks
job liveness (activity). If workers don't update their status often enough,
activity monitor will re-enqueue their jobs as pending. This ensures jobs make
progress even if some workers terminate unexpectedly.

`JobState`: Describes the current state of the job. The possible values are
`"running"`, `"pending"`, and `"finished"`. These are the minimal number of
states needed to describe a job's behavior in respect to this framework. Each
job type MAY have additional, type specific states, such as `"failed`",
`"error"`, `"retrying"`, etc.

`Sequence`: a 13 byte value formed by combining the current `Incarnation` of
the database and the `Versionstamp` of the transaction. Sequences are
monotonically increasing even when a database is relocated across FoundationDB
clusters. See (RFC002) for a full explanation.

---

# Framework API

This section describes the job creation and worker implementation APIs. It doesn't
describe how the framework is implemented. The intended audience is CouchDB
developers using this framework to implement background jobs for indexing,
replication, and couch-peruser.

Both the job creation and the worker implementation APIs use a `JobOpts` map to
represent a job. It MAY also contain these top level fields:

  * `"priority"` : The value of this field will contain the `Priority` value of
    the job. `Priority` is job-type specific.
  * `"data"`: An opaque object (map), from the framework's point of view,
    containing job-type specific data. It MAY contain an update sequence, or an
    error message, for example.
  * `"cancel"` : Boolean field defaulting to `false`. If `true` indicates the
    user intends to stop a job's execution.
  * `"resubmit"` : Boolean field defaulting to `false`. If `true` indicates
    the job should be re-submitted.

### Job Creation API ###

```
add(Type, JobId, JobOpts) -> ok | {error, Error}
```
 - Add a job to be executed by a background worker.

```
remove(Type, JobId) -> ok | not_found
```
 - Remove a job. If it is running, it will be stopped, then it will be removed
   from the job table.

```
resubmit(Type, JobId) -> ok | not_found
```
 - Indicates that the job should be re-submitted for execution.

```
get_job(Type, JobId) -> {ok, JobOpts, JobState}
```
 - Return `JobOpts` and the `JobState`. `JobState` value MAY be:
  * `"pending"` : This job is pending.
  * `"running"` : This job is currently running.
  * `"finished"` : This job has finished running and is not pending.

### Worker Implementation API

This API is to be used when implementing workers for various job types. The general pattern
is to call `accept()` from something like a job manager, then for each accepted
job spawn a worker to execute it, and then resume calling `accept()` to get
other jobs. When a job is running, the worker MUST periodically call `update()`
to prevent the activity monitor from re-enqueueing it. When the worker decides to stop
running a job, they MUST call `finish()` to indicate that the job has finished running.

```
accept(Type[, MaxPriority]) -> {ok, JobId, WorkerLockId} | not_found
```

 - Dequeue a job from the pending queue. `WorkerLockId` is a UUID indicating
   that the job is owned exclusively by the worker. `WorkerLockId` will be
   passed as an argument to other API functions below and will be used to
   verify that the current worker is still the only worker executing that job.
   `MaxPriority` is an optional value which can limit the maximum job priority
   that will be accepted. Jobs with priorities higher than that will not be
   accepted. The intended usage is to allow `Priority` to be used as a way to
   schedule job for executing at a future date. For example, `Priority` MAY
   indicate that a replication job which has been repeatedly failing should
   not execute any sooner than one hour from now.

```
finish(Tx, Type, JobId, JobOpts, WorkerLockId) -> ok | worker_conflict | canceled
```

 - Called by the worker when the job has finished running. The `"data"` field
   in `JobOpts` MAY contain a final result field or information about a fatal
   error.

```
resubmit(Tx, Type, JobId, WorkerLockId) -> ok | worker_conflict | canceled

```
 - The worker MAY call this function in order to mark the job for
   re-submission. This MAY be used to penalize jobs which have been running for
   too long, or jobs which have been repeatedly failing. Note that both the
   user and a worker can request a job to be re-submitted.

```
update(Tx, Type, JobId, JobOpts, WorkerLockId) -> ok | worker_conflict | canceled
```
 - This MAY be called to update a job's progress (update sequence, number of
   changes left, etc.) This function MUST be called at least as often as the
   `ActivityTimeout` in order for the activity monitor to not re-submit the
   job into the pending queue due to inactivity.

When the functions above return a `worker_conflict` value it means the activity
monitor has already re-enqueued the job. It is now either pending or is executed
by another worker. In that case, the caller MUST stop running the job, and they
MUST NOT update the job's entry any longer.

If `canceled` is returned, it means the user has requested the job to stop
executing. In that case the worker MUST top running the job.

#### Mutual Exclusion

If each worker updates their job status on time, and the activity monitor is
running correctly, the same job SHOULD be executed by no more than one worker
at a time. But, if the worker process is blocked for too long (for instance, in
an overload scenario), it may fail to update its status often enough, and the
activity monitor MAY re-enqueue the job. However, even in such cases, the
mutual exclusion constraint can be maintained as long as `update(Tx,...)`
function is called in the same transaction as the type-specific DB writes, and
its result is checked for the `worker_conflict` return.

# Implementation Details

## Data Model

`("couch_jobs", "data", Type, JobId) = (Sequence, WorkerLockId, JobOpts)`
`("couch_jobs", "pending", Type, Priority, JobId) = ""`
`("couch_jobs", "watches", Type) = Sequence`
`("couch_jobs", "activity_timeout", Type) = ActivityTimeout`
`("couch_jobs", "activity", Type, Sequence) = JobId`

The `("couch_jobs", "data", Type, JobId)` entry is the `job table` referenced
in the term definition and throughout the document.

`Type`, `Priority`, and `JobId` MUST be any type supported by the FoundationDB
tuple layer.

`JobOpts` MUST be a JSON object encoded as a string. It MAY have `"priority"`,
`"cancel"`, `"resubmit"`, or `"data"` fields.

## Job Lifecyle Implementation

This section describes how the framework implements some of the API functions.

`add()` :
  * Add the new job to the main jobs table
  * If a job with the same `JobId` exists, return an error
  * `WorkerLockId` is set to `null`.

`accept()` :
 * Generate a unique `WorkerLockId` UUID.
 * Attempt to dequeue the item from the pending queue, then assign it the
   `WorkerLockId` in the jobs table.
 * Create entry in the `"activity"` subspace.
 * Update sequence in the `"watches"` subspace for that job's type.

`update()`:
 * Check if `WorkerLockId` matches, otherwise return `worker_conflict`
 * Check if `"canceled"` field in the `JobOpts` is `true`, if so, return `canceled`
 * Delete old `"activity"` sequence entry
 * Update `JobOpts`
 * Create a new `"activity"` sequence entry and in main job table
 * Update `"watches"` sequence for that job type

`finish()`:
 * Check if `WorkerLockId` matches, otherwise returns `worker_conflict`
 * Check if `Canceled` is `true`, if so, return `canceled`
 * Delete old `"activity"` sequence entry
 * If `"resubmit"` field in `JobOpts` is `true`, re-enqueue the job into the pending queue.
 * Set job table's `WorkerLockId` to `null`

## Activity Monitor Implementation

There is an activity monitor running for each job type. It performs periodic
scans of the `"activity"` range with sequences less than `"watches"` sequence for
that type:

  * On first run:
    * Read the `ActivityTimeout` value and remember the `"watches"` sequence
  * On subsequent runs:
    * Call `get_range()` on `"activity"` entries less than the remembered sequence
    * Read and store the new `"watches"` sequence
    * If there are any stale entries returned, re-enqueue them

Every time a job is updating its `"activity"`, it bumps the `"watches"`
sequence. That means `"watches"` sequence points to the newest updated sequence
for that job type at that point in time. That allows anyone, in this case the
activity monitor, to remember the `"watches"` sequence value then return some
time later, and query if anything has not been updated since time it saw that
sequence. If everything is working properly and jobs are updating that should
return an empty list.

## Job State Diagram

The job state transition diagram looks something like this:

```
             +--------- accept() ---------->+
             |                              |
             |                              v
add() -> [PENDING]                      [RUNNING] ---> finish(), resubmit=false --> [FINISHED]
             ^                              |
             |                              |
             +-- finish(), resubmit=true --+
             ^                              |
             |                              |
             +-- activity monitor timeout --+
```

When a job is added it immediately becomes `pending`. When it is `accepted()`
for execution by a worker it becomes `running`. Then, if `finish` is called it
becomes finished if the `"resubmit"` option is `false`, or it goes back to
`pending` if it is `true`. A job may also be moved from `running` to
`pending` by the activity monitor if the job fails to update its status often
enough.

Each job type MAY define additional states. For example, replication jobs MAY
have both `completed` and `failed` as terminal states. In both cases the
framework will consider those as `finished`, but job-type specific data MAY
indicate that `completed` is finished with a successful result, and `failed` is
finished with a terminal failure (say an invalid replication doc format).

# Advantages and Disadvantages

A previous draft of this RFC discussed an implementation centered around
workers instead of jobs. In that implementation a "worker" was something closer
to a "job manager" for each job type. The workers would register in the
database and activity monitoring was performed for each worker instead of for
each job. However, that entailed extra API complexity from a job creation and
worker implementation API. Focusing on jobs as a central concept simplifies the
API considerably.

## Possible Future Extensions

Since all job keys and values are just FDB tuples and json encoded objects, in
the future it might be possible to accept external jobs, not just jobs defined
by the CouchDB internals. Also, since workers could be written in any language
as long as they can talk to the FDB cluster, and follow the behavior describes
in the design, it opens the possibility to have custom (user defined) workers
of different types. But that is out of scope in the current RFC discussion.

# Key Changes

 - New job execution framework
 - A single global job queue for each job type
 - An activity monitor to ensure jobs continue to make progress

## Applications and Modules Affected

Replication, indexing, couch-peruser

## HTTP API Additions

None. However, in the future, it might be useful to have an API to query and
monitor the state of all the queues and workers.

## HTTP API Deprecations

None have been identified.

# Security Considerations

None have been identified.

# References

[Original mailing list discussion](https://lists.apache.org/thread.html/9338bd50f39d7fdec68d7ab2441c055c166041bd84b403644f662735@%3Cdev.couchdb.apache.org%3E)

# Co-authors
  - @davisp

# Acknowledgments
 - @davisp
 - @kocolosk
 - @garrensmith
 - @rnewson
 - @mikerhodes
 - @sansato
