---
title: "Running phone builds on Taskcluster"
category: mozilla
comments: true
tags: [taskcluster]
---

In this post I am going to talk about my work for phone builds inside the
[Taskcluster](https://docs.taskcluster.net/) infrastructure. Mozilla is
slightly moving from Buildbot to Taskcluster. Here I am going to
give a survivor guide on Firefox OS phone builds.

Submitting tasks
---------------

A task is nothing more than a json file containing the description 
of the job to execute. But you don't need to handle the json directly, all tasks
are written in [YAML](https://en.wikipedia.org/wiki/YAML), and it is then processed
by the [mach](https://mzl.la/1MkZ4gz) command. The in tree tasks are located
at [testing/taskcluster/tasks](https://mzl.la/1MkYOhw) and the build tasks are
inside the `builds/` directory.

My favorite command to try out a task is the `mach taskcluster-build` command.
It allows you to process a single task and output the json formatted task ready
for Taskcluster submission.

```bash
$ ./mach taskcluster-build \
    --head-repository=https://hg.mozilla.org/mozilla-central 
    --head-rev=tip \
    --owner=foobar@mozilla.com \
    tasks/builds/b2g_desktop_opt.yml
```

Although we specify a Mercurial repository, Taskcluster also accepts git
repositories interchangeably.

This command will print out the task to the console output. To
run the task, you can copy the generated task and paste it in the
[task creator](https://tools.taskcluster.net/task-creator/) tool. Then just
click on `Create Task` to schedule it to run. Remember that you need
[Taskcluster Credentials](https://auth.taskcluster.net/) to run Taskcluster
tasks. If you have
[taskcluster-cli](https://www.npmjs.com/package/taskcluster-cli/)
installed, you can the pipe the mach output to `taskcluster run-task`.

The tasks are effectively executed inside a [docker](https://www.docker.com/)
[image](https://dxr.mozilla.org/mozilla-central/source/testing/docker).

Mozharness
----------

[Mozharness](https://wiki.mozilla.org/ReleaseEngineering/Mozharness)
is what we use for effectively build stuff. Mozharness
architecture, despite its code size, is quite simple. Under the 
`scripts` directory you find the harness scripts. We are specifically
interested in the [b2g\_build.py](https://tinyurl.com/nlm8mjm) script. As the script
name says, it is responsible for B2G builds. The B2G harness configuration
files are located at the [b2g/config](https://tinyurl.com/nzqlkfe) directory. Not
surprisingly, all files starting with "taskcluster" are for Taskcluster
related builds.

Here are the most common configurations:

<dl>
  <dt>default_vcs</dt>
  <dd>This is the default vcs used to clone repositories when no other is given.
  [tc_vcs](https://tc-vcs.readthedocs.org/en/latest/) allows mozharness to
  clone either git or mercurial repositories transparently, with repository
  caching support.</dd>
  <dt>default_actions</dt>
  <dd>The actions to execute. They must be present and in the same order as
  in the build class `all_actions` attribute.</dd>
  <dt>balrog_credentials_file</dt>
  <dd>The credentials to send update data to the OTA server.</dd>
  <dt>nightly_build</dt>
  <dd>`True` if this is a nightly build.</dd>
  <dt>upload</dt>
  <dd>Upload info. Not used for Taskcluster.</dd>
  <dt>repo_remote_mappings</dt>
  <dd>Maps externals repository to [mozilla domain](https://git.mozilla.org).</dd>
  <dt>env</dt>
  <dd>Environment variables for commands executed inside mozharness.</dd>
</dl>

The listed actions map to Python methods inside the build class, with `-` replaced
by `_`. For example, the action `checkout-sources` maps to the method
`checkout_sources`. That's where the mozharness simplicity comes from: everything boils
down to a sequence of method calls, just it, no secret. 

For example, here is how you run mozharness to build a flame image:

```bash
python <gecko-dir>/testing/mozharness/scripts/b2g_build.py \
  --config b2g/taskcluster-phone.py \
  --disable-mock \
  --variant=user \
  --work-dir=B2G \
  --gaia-languages-file locales/languages_all.json \
  --log-level=debug \
  --target=flame-kk \
  --b2g-config-dir=flame-kk \
  --repo=https://hg.mozilla.org/mozilla-central \
```

Remember you need your flame connected to the machine so the build system
can extract the blobs.

In general you don't need to worry about mozharness command line because it is wrapped
by the [build scripts](https://tinyurl.com/py798c3).

Hacking Taskcluster B2G builds
------------------------------

All Taskcluster tasks run inside a docker container. Desktop and emulator B2G builds
run inside the `builder` docker image. Phone builds are more complex, because:

1. Mozilla is not allowed to publicly redistribute phone binaries.

2. Phone build tasks need to access the [Balrog](https://wiki.mozilla.org/Balrog)
  server to send OTA update data.

3. Phone build tasks need to upload symbols to the
  [crash reporter](https://mzl.la/1Ta6jfY).

Due to (1), only users authenticated with a @mozilla account are allowed
to download phone binaries (this works the same way as private builds). And
because of (1), (2) and (3), the `phone-builder` docker image is secret,
so only authorized users can submit tasks to it.

If you need to create a build task for a new phone, most of the time you will
starting from an existing task (Flame and Aries tasks are preferred) and then
make your customizations. You might need to add new features to the
[build scripts](https://tinyurl.com/py798c3), which currently are not the most
flexible scripts around.

If you need to customize mozharness, make sure your changes are Python 2.6
compatible, because mozharness is used to run Buildbot builds too, and the
Buildbot machines run Python 2.6. The best way to minimize risk of breaking
stuff is to submit your patches to try with "-p all -b do" flags.

Need help? Ask at the #taskcluster channel.
