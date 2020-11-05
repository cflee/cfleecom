---
title: "Exit status 125 when upgrading Amazon SSM Agent snap"
date: 2020-11-05
---

This week I ran into an issue upgrading Amazon SSM Agent that was a bit of a head-scratcher.

SSM Agent is something that you can run on your EC2 instances, and on-premise systems in some scenarios, that lets [AWS Systems Manager](https://aws.amazon.com/systems-manager/) do [a bunch of handy things](https://docs.aws.amazon.com/systems-manager/latest/userguide/features.html).

While it is supported and packaged as both a .deb and a [snap](https://snapcraft.io/about) on Ubuntu 16.04, for whatever unfortunate reason they switched fully to snaps for Ubuntu 18.04 and later.
In either case, the updating and rollback process is fully handled by the SSM Agent when you run the AWS-UpdateSSMAgent SSM document.

What's especially nice is that even the package distribution is handled by the SSM team's S3 buckets, so EC2 instances without internet access can be managed with just network access to the [various SSM and S3 VPC Endpoints](https://aws.amazon.com/premiumsupport/knowledge-center/ec2-systems-manager-vpc-endpoints/).
I have a lot of gripes about SSM Agent (not rotating its logs, not cleaning up its downloads, just off the top of my head...) but this aspect ain't one of them.
Until now...

## Problem

Upon running the AWS-UpdateSSMAgent SSM document to update to the default/latest agent version, it fails with a mysterious message to stderr:

```
failed to install amazon-ssm-agent 3.0.284.0, ErrorMessage=The execution of command returned Exit Status: 125
 exit status 125
```

And on stdout:

```
Updating amazon-ssm-agent from 2.3.978.0 to latest
Successfully downloaded https://s3.ap-southeast-1.amazonaws.com/amazon-ssm-ap-southeast-1/ssm-agent-manifest.json
Successfully downloaded https://s3.ap-southeast-1.amazonaws.com/amazon-ssm-ap-southeast-1/amazon-ssm-agent-updater/3.0.284.0/amazon-ssm-agent-updater-snap-amd64.tar.gz
Successfully downloaded https://s3.ap-southeast-1.amazonaws.com/amazon-ssm-ap-southeast-1/amazon-ssm-agent/2.3.978.0/amazon-ssm-agent-snap-amd64.tar.gz
Successfully downloaded https://s3.ap-southeast-1.amazonaws.com/amazon-ssm-ap-southeast-1/amazon-ssm-agent/3.0.284.0/amazon-ssm-agent-snap-amd64.tar.gz
Initiating amazon-ssm-agent update to 3.0.284.0
failed to install amazon-ssm-agent 3.0.284.0, ErrorMessage=The execution of command returned Exit Status: 125
 exit status 125
Initiating rollback amazon-ssm-agent to 2.3.978.0
rolledback amazon-ssm-agent to 2.3.978.0
Failed to update amazon-ssm-agent to 3.0.284.0
```

## Analysis

Reviewing the full logs on the hosts at `/var/log/amazon/ssm/AmazonSSMAgent-update.txt` did not yield any insights.

While waiting for AWS Support to get back on this, I searched the [GitHub repo](https://github.com/aws/amazon-ssm-agent) and started poking around the install script [`snap-install.sh`](https://github.com/aws/amazon-ssm-agent/blob/3.0.284.0/Tools/src/update/ubuntu/snap-install.sh).
It does an `exit 125` if the `snap install` commands exits with a nonzero code:

```
# acknowledge the signature pulled from the s3 distro
snap ack amazon-ssm-agent.assert
# install snap in classic mode
echo 'installing snap'
snap install --classic amazon-ssm-agent.snap
pmExit=$?

# [...]

if [ "$pmExit" -ne 0 ]; then
  echo "Package manager failed with exit code '$pmExit'"
  exit 125
fi
```

We can get the `amazon-ssm-agent.assert` and `amazon-ssm-agent.snap` files from the `amazon-ssm-agent-snap-amd64.tar.gz` file mentioned in the logs.
If we try to manually do these steps on the files, the `snap ack` works fine, but the `snap install` spins for quite a long timeout period and then exits with this text:

```
error: cannot perform the following tasks:
- Ensure prerequisites for "amazon-ssm-agent" are available (cannot install snap base "core18": Post https://api.snapcraft.io/v2/snaps/refresh: net/http: request canceled while waiting for connection (Client.Timeout exceeded while awaiting headers))
```

Aha!
`snap list` confirms that the host only has the `core` and `amazon-ssm-agent` snaps installed.
This host has no (unproxied) outbound internet access, so that would explain the timeout.

(It is incredibly annoying that the [snapcraft store page](https://snapcraft.io/amazon-ssm-agent) for the snap does not show dependencies at all.
Neither does running `snap info amazon-ssm-agent` on the snap's name, nor `snap info amazon-ssm-agent.snap` on the snap package.)

Searching through the GitHub repo again yields exactly one hit, a note in [RELEASENOTES.md](https://github.com/aws/amazon-ssm-agent/blob/3.0.284.0/RELEASENOTES.md) for 2.3.1205.0:

```
- Updated the SSM Agent Snap to core18
```

🤯 😱 🤦

There you have it:

- when SSM Agent is installed as a snap package
- on a host where snapd has no internet access to download new snaps,
- and the core18 snap is not already installed,

upgrading to SSM Agent 2.3.1205.0 or newer (including 3.x) will fail.

This happens regardless of whether the newer snap is being installed automatically by the AWS-UpdateSSMAgent SSM document, or manually by retrieving the snap package and attempting to install it, it just fails differently.

## Workarounds

The goal is to get the `core18` snap installed, so that the newer `amazon-ssm-agent` snap versions can be installed.

You could arrange for all the instances to get direct internet access so that snapd can connect to the Snap Store.
If you don't have rules against this and it's just a matter of briefly adding an outbound security group rule to the internet, why not?

You could configure all the instances to have snapd use a http/https proxy for connecting to the Snap Store.
This can be done using the [snapd system options](https://snapcraft.io/docs/system-options), or as environment variables in `/etc/environment` or a systemd unit override on `snapd.service`.

You could set up a [Snap Store Proxy](https://docs.ubuntu.com/snap-store-proxy/en/) that has internet access (possibly through a http/https proxy), and connect the other hosts to it.
However, it seems to be an enterprise product from Canonical, and might be limited to 5 devices for evaluation purposes.

You could `snap download` on a host where snapd has internet access, transfer the downloaded `.snap` and `.assert` files to the other hosts, and then install on each host using `snap ack core18_1234.assert` and `snap install core18_1234.snap`.

You could attempt to locate the actual snap file using the Snapstore Devices API's [`snap_info`](https://api.snapcraft.io/docs/info.html) endpoint, which you can filter for the correct architecture and version, and extract the download url.
However, this only gives you the `.snap` file, and I couldn't figure out how to invoke the assertions endpoint to retrieve the chain of 4 (!) assertions required.
The snap package is still usable, but you'd need to use `snap install --dangerous` to override the assertion requirement.

(I did find [this AskUbuntu question](https://askubuntu.com/questions/1075694/how-to-get-snap-download-url) that had a pointer to the [assert-fetcher](https://bazaar.launchpad.net/~pedronis/snappy-hub/assert-fetcher/view/head:/main.go?start_revid=5) tool, which actually digs directly into snapd to invoke the [github.com/snapcore/snapd/asserts](https://godoc.org/github.com/snapcore/snapd/asserts) go package...
I would usually be pleased to see a small Go program?
But not when it's distributed as a snap package.
Gah!)

You could also make enough noise to the Amazon Systems Manager teams until they add handling to detect this case and distribute the core18 snap/assertion, so that the SSM Agent upgrades go back to working like magic.
