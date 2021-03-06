---
layout: "intro"
page_title: "Jobs"
sidebar_current: "getting-started-jobs"
description: |-
  Learn how to submit, modify and stop jobs in Nomad.
---

# Jobs

Jobs are the primary configuration that users interact with when using
Nomad. A job is a declarative specification of tasks that Nomad should run.
Jobs have a globally unique name, one or many task groups, which are themselves
collections of one or many tasks.

The format of the jobs is [documented here](/docs/jobspec/index.html). They
can either be specified in [HCL](https://github.com/hashicorp/hcl) or JSON,
however we recommend only using JSON when the configuration is generated by a machine.

## Running a Job

To get started, we will use the [`init` command](/docs/commands/init.html) which
generates a skeleton job file:

```
$ nomad init
Example job file written to example.nomad

$ cat example.nomad

# There can only be a single job definition per file.
# Create a job with ID and Name 'example'
job "example" {
	# Run the job in the global region, which is the default.
	# region = "global"
...
```

In this example job file, we have declared a single task 'redis' which is using
the Docker driver to run the task. The primary way you interact with Nomad
is with the [`run` command](/docs/commands/run.html). The `run` command takes
a job file and registers it with Nomad. This is used both to register new
jobs and to update existing jobs.

We can register our example job now:

```
$ nomad run example.nomad
==> Monitoring evaluation "26cfc69e"
    Evaluation triggered by job "example"
    Allocation "8ba85cef" created: node "171a583b", group "cache"
    Evaluation status changed: "pending" -> "complete"
==> Evaluation "26cfc69e" finished with status "complete"
```

Anytime a job is updated, Nomad creates an evaluation to determine what
actions need to take place. In this case, because this is a new job, Nomad has
determined that an allocation should be created and has scheduled it on our
local agent.

To inspect the status of our job we use the [`status` command](/docs/commands/status.html):

```
$ nomad status example
ID          = example
Name        = example
Type        = service
Priority    = 50
Datacenters = dc1
Status      = running
Periodic    = false

Allocations
ID        Eval ID   Node ID   Task Group  Desired  Status
dadcdb81  61b0b423  72687b1a  cache       run      running
```

Here we can see that the result of our evaluation was the creation of an
allocation that is now running on the local node.

An allocation represents an instance of Task Group placed on a node. To inspect
an Allocation we use the [`alloc-status` command](/docs/commands/alloc-status.html):

```
$ nomad alloc-status dadcdb81
ID            = dadcdb81
Eval ID       = 61b0b423
Name          = example.cache[0]
Node ID       = 72687b1a
Job ID        = example
Client Status = running

Task "redis" is "running"
Task Resources
CPU    Memory           Disk     IOPS  Addresses
2/500  6.3 MiB/256 MiB  300 MiB  0     db: 127.0.0.1:30329

Recent Events:
Time                   Type      Description
06/23/16 01:41:16 UTC  Started   Task started by client
06/23/16 01:41:13 UTC  Received  Task received by client
```

We can see that Nomad reports the state of the allocation as well as its
current resource usage. By supplying the `-stats` flag, more detailed resource
usage statistics will be reported.

To inspect the file system of a running allocation, we can use the [`fs`
command](/docs/commands/fs.html):

```
$ nomad fs 8ba85cef alloc/logs
Mode        Size    Modified Time          Name
-rw-rw-r--  0 B     15/03/16 15:40:56 PDT  redis.stderr.0
-rw-rw-r--  2.3 kB  15/03/16 15:40:57 PDT  redis.stdout.0

$ nomad fs 8ba85cef alloc/logs/redis.stdout.0
                 _._
            _.-``__ ''-._
       _.-``    `.  `_.  ''-._           Redis 3.2.1 (00000000/0) 64 bit
   .-`` .-```.  ```\/    _.,_ ''-._
  (    '      ,       .-`  | `,    )     Running in standalone mode
  |`-._`-...-` __...-.``-._|'` _.-'|     Port: 6379
  |    `-._   `._    /     _.-'    |     PID: 1
   `-._    `-._  `-./  _.-'    _.-'
  |`-._`-._    `-.__.-'    _.-'_.-'|
  |    `-._`-._        _.-'_.-'    |           http://redis.io
   `-._    `-._`-.__.-'_.-'    _.-'
  |`-._`-._    `-.__.-'    _.-'_.-'|
  |    `-._`-._        _.-'_.-'    |
   `-._    `-._`-.__.-'_.-'    _.-'
       `-._    `-.__.-'    _.-'
           `-._        _.-'
               `-.__.-'
...
```

## Modifying a Job

The definition of a job is not static, and is meant to be updated over time.
You may update a job to change the docker container, to update the application version,
or to change the count of a task group to scale with load.

For now, edit the `example.nomad` file to uncomment the count and set it to 3:

```
# Control the number of instances of this group.
# Defaults to 1
count = 3
```

Once you have finished modifying the job specification, use [`plan`
command](/docs/commands/plan.html) to invoke a dry-run of the scheduler to see
what would happen if your ran the updated job:

```
$ nomad plan example.nomad
+/- Job: "example"
+/- Task Group: "cache" (2 create, 1 in-place update)
  +/- Count: "1" => "3" (forces create)
      Task: "redis"

Scheduler dry-run:
- All tasks successfully allocated.

Job Modify Index: 6
To submit the job with version verification run:

nomad run -check-index 6 example.nomad

When running the job with the check-index flag, the job will only be run if the
server side version matches the the job modify index returned. If the index has
changed, another user has modified the job and the plan's results are
potentially invalid.
```

We can see that the scheduler detected the change in count and informs us that
it will cause 2 new instances to be created. The in-place update that will
occur is to push the update job specification to the existing allocation and
will not cause any service interuption. We can then run the job with the
run command the `plan` emitted.

By running with the `-check-index` flag, Nomad checks that the job has not
been modified since the plan was run. This is useful if multiple people are
interacting with the job at the same time to ensure the job hasn't changed
before you apply your modifications.

```
$ nomad run -check-index 6 example.nomad
==> Monitoring evaluation "127a49d0"
    Evaluation triggered by job "example"
    Allocation "8ab24eef" created: node "171a583b", group "cache"
    Allocation "f6c29874" created: node "171a583b", group "cache"
    Allocation "8ba85cef" modified: node "171a583b", group "cache"
    Evaluation status changed: "pending" -> "complete"
==> Evaluation "127a49d0" finished with status "complete"
```

Because we set the count of the task group to three, Nomad created two
additional allocations to get to the desired state. It is idempotent to
run the same job specification again and no new allocations will be created.

Now, let's try to do an application update. In this case, we will simply change
the version of redis we want to run. Edit the `example.nomad` file and change
the Docker image from "redis:latest" to "redis:2.8":

```
# Configure Docker driver with the image
config {
    image = "redis:2.8"
}
```

We can run `plan` again to see what will happen if we submit this change:

```
$ nomad plan example.nomad
+/- Job: "example"
+/- Task Group: "cache" (3 create/destroy update)
  +/- Task: "redis" (forces create/destroy update)
    +/- Config {
      +/- image:           "redis:latest" => "redis:2.8"
          port_map[0][db]: "6379"
    }

Scheduler dry-run:
- All tasks successfully allocated.
- Rolling update, next evaluation will be in 10s.

Job Modify Index: 42
To submit the job with version verification run:

nomad run -check-index 42 example.nomad

When running the job with the check-index flag, the job will only be run if the
server side version matches the the job modify index returned. If the index has
changed, another user has modified the job and the plan's results are
potentially invalid.
```

Here we can see the `plan` reports it will do three create/destroy updates
which stops the old tasks and starts the new tasks because we have changed the
version of redis to run. We also see that the update will happen with a rolling
update. This is because our example job is configured to do a rolling update
via the `stagger` attribute, doing a single update every 10 seconds. 

Once ready, use `run` to push the updated specification now:

```
$ nomad run example.nomad
==> Monitoring evaluation "ebcc3e14"
    Evaluation triggered by job "example"
    Allocation "9a3743f4" created: node "171a583b", group "cache"
    Evaluation status changed: "pending" -> "complete"
==> Evaluation "ebcc3e14" finished with status "complete"
==> Monitoring evaluation "b508d8f0"
    Evaluation triggered by job "example"
    Allocation "926e5876" created: node "171a583b", group "cache"
    Evaluation status changed: "pending" -> "complete"
==> Evaluation "b508d8f0" finished with status "complete"
==> Monitoring next evaluation "ea78c05a" in 10s
==> Monitoring evaluation "ea78c05a"
    Evaluation triggered by job "example"
    Allocation "3c8589d5" created: node "171a583b", group "cache"
    Evaluation status changed: "pending" -> "complete"
==> Evaluation "ea78c05a" finished with status "complete"
```

We can see that Nomad handled the update in three phases, only updating a single task
group in each phase. The update strategy can be configured, but rolling updates makes
it easy to upgrade an application at large scale.

## Stopping a Job

So far we've created, run and modified a job. The final step in a job lifecycle
is stopping the job. This is done with the [`stop` command](/docs/commands/stop.html):

```
$ nomad stop example
==> Monitoring evaluation "fd03c9f8"
    Evaluation triggered by job "example"
    Evaluation status changed: "pending" -> "complete"
==> Evaluation "fd03c9f8" finished with status "complete"
```

When we stop a job, it creates an evaluation which is used to stop all
the existing allocations. This also deletes the job definition out of Nomad.
If we try to query the job status, we can see it is no longer registered:

```
$ nomad status example
No job(s) with prefix or id "example" found
```

If we wanted to start the job again, we could simply `run` it again.

## Next Steps

Users of Nomad primarily interact with jobs, and we've now seen
how to create and scale our job, perform an application update,
and do a job tear down. Next we will add another Nomad
client to [create our first cluster](cluster.html)

