---
title: "The taskcluster-worker Mac OSX engine"
category: mozilla
comments: true
tags: [taskcluster]
---

In this quarter, I worked on implementing the
[taskcluster-worker](http://blog.gregarndt.com/taskcluster/2016/03/24/birth-of-new-worker/)
Mac OSX engine. Before talking about this specific implementation,
let me explain what a worker is and how taskcluster-worker differs from
[docker-worker](https://github.com/taskcluster/docker-worker), the currently
main worker in
[Taskcluster](http://yourdomain.com/mozilla,%20ci/2014/03/04/taskcluster.html).

The role of a Taskcluster worker
================================

When a user submits a task graph to Taskcluster,
contrary to the common sense (at least if you are used on how OSes
schedulers usually work), these tasks are submitted to the scheduler first,
which is responsible to process dependencies and enqueue them. In the
[Taskcluster manual page](https://docs.taskcluster.net/manual) there is a
clear picture ilustrating this concept.

The provisioner is responsible for looking at the queue and determine how
many pending tasks exist and, based on that, it launches worker instances to
run these tasks.

Then comes the figure of the worker. The worker is responsible for actually
executing the task. It claims a task from the queue, runs it, upload the
generated artifacts and submits the status of the finished task, using the
[Taskcluster APIs](https://docs.taskcluster.net/manual/apis).

`docker-worker` is a worker that runs task command inside a docker container.
The task payload specifies a [docker](https://www.docker.com/what-docker)
image as well as a command line to run, among other environment parameters.
docker-worker pulls the specified docker image and runs task commands inside it.

taskcluster-worker and the OSX engine
=====================================

`taskcluster-worker` is a generic and modularized worker under active
development by the Taskcluster team. The worker delegates the task execution
to one of the available
[engines](https://github.com/taskcluster/taskcluster-worker/tree/master/engines).
An engine is a component of taskcluster-worker responsible for running a task
under a specific system environment. Other features, like environment variable
setting, live logging, artifact uploading, etc., are handled by
[worker plugins](https://github.com/taskcluster/taskcluster-worker/tree/master/plugins).

I am implementing the Mac OSX engine, which will mainly be used to run
Firefox automated tests in the Mac OSX environment. There is a
[`macosx` branch](https://github.com/walac/taskcluster-worker/tree/macosx) in
my personal Github taskcluster-worker fork in which I push my commits.

One specific aspect of the engine implementation is the ability to run more
than one task at the same time. For this, we need to implement some kind
of task isolation. For docker-worker, each task ran in its own docker container
so tasks were isolated by definition. But there is no such thing as a container
for OSX engine. Our earlier tries with
[chroot](https://en.wikipedia.org/wiki/Chroot) failed miserably, due to
incompatibilities with OSX graphic system. Our final solution was to create a new user
on the fly and run the task with this user's credentials. This not only provides
some task isolation, but also prevents privilege escalation attacks by running
tasks with different user than the worker.

Instead of dealing with the poorly documented
[Open Directory Framework](https://developer.apple.com/library/mac/documentation/Networking/Conceptual/Open_Directory/Introduction/Introduction.html),
we chose to spawn the
[dscl](https://developer.apple.com/legacy/library/documentation/Darwin/Reference/ManPages/man1/dscl.1.html)
command to create and configure users. Tasks usually takes a long time to
execute, spawning loads of subprocess, so a few spawns of the `dscl` command
won't have any practical performance impact.

One final aspect is how we bootstrap task execution. A tasks boils down to
a script that executes task duties. But where does this script come from?
It doesn't live in the machine that executes the worker. OSX engine provides a
`link` field in task payload that a task can specify an executable to download and
execute.

Running the worker
==================

OSX engine will primarily be used to execute Firefox tests on Mac OSX,
and the environment is expected to have a very specific tools and
configurations set. Because of that, I am testing the code on a
[loaner machine](https://wiki.mozilla.org/ReleaseEngineering/How_To/Loan_a_Slave).
To start the worker, it is just a matter of opening a terminal and typing:

```bash
$ ./taskcluster-worker work macosx --logging-level debug
```

The worker connects to the Taskcluster queue, claims and execute the tasks available.
At the time I am writing, all tests but *Firefox UI functional* tests" were green,
running on optimized Firefox OSX builds. We intend to land Firefox tests in taskcluster-worker as
[Tier-2](https://developer.mozilla.org/en-US/docs/Supported_build_configurations) on next quarter,
running them in parallel with Buildbot.
