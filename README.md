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
Currently, the instances will just go away once they're done processing a job, so you'll have to keep adding more with `./spawn`.
This isn't ideal but can be scripted around easily enough.

In the future, it'd be better to have a more complex script or program
which keeps track of instances, can check how many are ready, how many
are in use and re-plenish the pool so that there always are a couple of
instances ready up to a maximum that the system can handle.
