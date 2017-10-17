---
title: "In tree tasks configuration"
category: taskcluster
comments: true
tags: [taskcluster]
---

This post is about our plans for representing Taskcluster tasks inside
the gecko tree. [Jonas](https://jonasfj.dk/),
[Dustin](https://code.v.igoro.us/) and I had a discussion in Berlin about this,
here I summarize what we have so far. We currently store tasks in an
[yaml](https://yaml.org/) file and they translate to json format using the
mach command. The syntax we have now is not the most flexible one, it is hard
to parameterize the task and very difficulty to represents tasks relationships.

Let us illustrate the shortcomings with two problems we currently have.
Both apply to B2G.

B2G (as in Android) has three different
[build variants](https://source.android.com/source/building.html#choose-a-target):
user, userdebug and eng. Each one has slightly different task configurations.
As there is no flexible way to parameterize tasks, we end up with one different
task file for each build variant.

When doing nightly builds, we must send update data to the OTA server.
We have plans to run a build task, then run the test tasks on this build,
and if all tests pass, we run a task responsible to update the OTA server.
The point is that today we have no way to represent this relationship
inside the task files.

For the first problem Jonas has a prototype for
[json parameterization](https://github.com/jonasfj/json-parameterization). There
were discussions on Berlin work week either we should stick with yaml files
or use Python files for task configuration. We do want to keep the syntax
declarative, which favors yaml, but storing configurations in Python files
brings much more expressiveness and flexibility, but this can result in
the same configuration hell we have with Buildbot.

The second problem is more complex, and we still haven't reached a final design.
The first question is how we describe task dependencies, top-down, i.e., we
specify which task(s) should run after a completed task, or ground up, a task
specifies which tasks it depends on. In general, we all agreed to go to a
top-down syntax, since most scenarios beg for a top down approach. Other
either should put the description of tasks relationship inside the task
files or in a separated configuration file. We would like to represent task
dependencies inside the task file, the problem is how to check what's the
root task for the task graph. One suggestion is having a task file called
root.yml which only contain root tasks.
