# Introduction
This repository contains some scripts to create LXD instances suitable
to be used as Github Actions runners as well as tools to spawn a set of
such instances.

Self-hosted Github Actions runners allow for more powerful test runners,
testing with physical hardware attached (USB, serial, GPU, ...), testing
on otherwise unavailable architectures and more.

This will use LXD with a set of ephemeral instances which will then be
added as Github Runners. Each of those will process exactly one job and
then self-destroy. This is to avoid running jobs in unclean
environments, preventing one job from impacting or attacking the next.

https://www.youtube.com/watch?v=c40BPKcSr4E

# Creating a suitable base instance
Either containers or VMs can be use for this.

For example, for an Ubuntu 20.04 container, you'd use:
```
lxc launch images:ubuntu/20.04 base-container
```

For a VM:
```
lxc launch images:ubuntu/20.04 base-vm
```

(From this point on, I'll refer to that instance as `base-XYZ`)

You'll then use `lxc exec base-XYZ bash` to get a shell inside of the
environment and customize it in whatever way you see fit for your use.
This can be automated with the likes of a shell script, Ansible or Packer.

Now you'll need to fetch the action token from:
https://github.com/ORG/REPO/settings/actions/runners/new

Once you're done, you need to get it ready for use with Github, for that, run:

```
./prepare-instance base-XYZ https://github.com/ORG/REPO TOKEN
```

You can also pass additional labels like this:

```
./prepare-instance base-XYZ https://github.com/ORG/REPO TOKEN vm,gpu
```

Then stop the instance with `lxc stop base-XYZ`.

And before it's ready for use, let's set some limits and security options on it:
 - lxc config set base-XYZ security.idmap.isolated=true (only useful for containers)
 - lxc config set base-XYZ limits.cpu=2 limits.memory=4GiB
 - lxc config device override base-XYZ root size=20GiB

This restricts resources to 2 vCPU, 4GB of RAM and 20GB of disk. Adapt this to whatever makes sense for you.

# Starting some runners
Lastly, you need a set of instances to start and register with Github:

```
./spawn base-XYZ 10
```

This will start an initial set of 10 instances.

# What's next
Currently, the instances will just go away once they're done processing
a job, so you'll have to keep adding more with `./spawn`.
This isn't ideal but can be scripted around easily enough.

There's also the issue of the TOKEN being somewhat short lived, so
eventually, you'll have to run `./prepare` again with a new token.

To address that, it'd be great to:
 - Write a daemon process that manages all this
 - Have the daemon ensure a pool of available instances up to a maximum
 - Move the registration logic to the daemon, have it use the Github API
   to generate one-time tokens and pass those into the instance
 - Have the daemon enforce maximum job run time and cleanup any instance
   that exceeds it or that's otherwise broken in some way

Other future improvements could include:
 - Automatic update/re-generation of the base instances
 - Ability to pull in standard Github Actions images

This initial work was really all done in a few hours to see how easy it
is to run some self-hosted Github Actions runners now that the Actions
API properly support ephemeral test runners (was blocked on
https://github.com/actions/runner/issues/510 for a long time).

Anyone interested is very much encouraged to get in touch and contribute!
