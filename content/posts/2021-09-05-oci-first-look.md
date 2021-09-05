+++
title = "Oracle Cloud: first impressions"
+++

Oracle Cloud [announced the availability of OCI Ampere A1 Compute](https://www.oracle.com/news/announcement/oracle-unlocks-power-of-arm-processors-at-one-cent-per-core-hour-2021-05-25/), as well as it being part of OCI's "Always Free" offers about three months ago.
It sounds like a pretty good choice for experimenting with a decently sized Arm virtual machine, given that AWS is 'only' offering a t4g.micro free trial till end-2021.

Here are some first impressions snippets that I've run across so far.
Of the hyperscale public cloud providers, I'm only familiar with AWS right now, so what's notable is definitely going to be considered from that angle.

## Always Free

Highlights of [the Always Free offer](https://docs.oracle.com/en-us/iaas/Content/FreeTier/freetier_topic-Always_Free_Resources.htm), as far as poking around a VM or two is concerned:

- Two instances of the VM.Standard.E2.1.Micro shape (1/8th of an OCPU, 1 GB memory, one VNIC with one public IP address, and up to 50 Mbps bandwidth to the internet)
- The equivalent of 4 OCPUs and 24 GB of memory on the VM.Standard.A1.Flex shape (OCPU and memory can be flexibly sized into up to four VMs, as the shape allows flexible allocation of OCPUs in integer steps and min memory of 1 GB per OCPU, one VNIC per OCPU with a floor of two VNICs, and 1 Gbps network bandwidth per OCPU)
- Six public IPv4 addresses
- 200 GB of Block Volume storage, and five volume backups (there's a 50 GB minimum for boot volumes, so four instances will fully consume this)
- Two Virtual Cloud Networks
- 10 TB per month of outbound data

This looks fairly good.
You can comfortably have two to four Arm VMs without feeling too squeezed.

Note: OCI measures processors in OCPUs, where 1 OCPU represents two vCPUs (one physical core, two threads) for Intel/AMD and one vCPU (one core) for Arm.

## Creating an account (tenancy)

The first surprise is being asked for a "Home Region".
There's a little note that comes with it which notes that Ampere A1 availability is limited in a whole long slew of regions.
When I was opening my account, it was basically (for APAC) all the Japan and South Korea regions, so that left the Australia and India regions.
If you look at the OCI docs, it looks like home region shouldn't matter so much as the tenancy can subscribe to additional regions at any time, but in the [Free Tier docs](https://docs.oracle.com/en-us/iaas/Content/FreeTier/freetier.htm), there's a little note that Always Free compute instances can only be provisioned in the home region.
Pick wisely - you can try your luck in a closer region, or just go with wherever has capacity right now.

The second (more pleasant) surprise was that there's a hard line drawn between the 30-day Free Trial / "Always Free only" mode, and the "paid" mode that you have to explicitly upgrade to.
No bill shocks for the former!
This is really significant compared to the all-too-common AWS bill shocks to people learning and experimenting on their platform.

The downside of this is that you may get some other potential "shocks" or rough edges in the transition from the Free Trial mode to the "Always Free only" mode.
The ones I've spotted so far are:

- Ampere A1 Compute instances will be disabled and deleted (terminated) after the 30-day period, if the account is not upgraded to a paid account. New instances need to be created to continue use of A1 Compute instances.
- If the Free Trial ends with Object Storage usage over the free 20 GB limit, all objects are deleted.

It doesn't look like there's any way to accelerate the Free Trial other than waiting for 30 days.
Meanwhile, just remember to treat things as potentially ephemeral!

## Networking

The tutorials suggest that you can just use the "Create a VCN with Internet Connectivity" wizard.
Even though it involves creating a NAT Gateway for the private subnet to connect out to the internet, and a Service Gateway for accessing Oracle Cloud services.
Yes, it's fine, there's no charge for those items!
We're not in Kansas anymore!

"Security lists" are generally stateful and apply to the entire subnet, or rather, all VNICs in the subnet.
That's the default mechanism that is automatically created and set up by the VCN creation wizard.
However, there's a newer construct of "network security groups" that you can attach to specific Compute instances, or rather its primary VNIC, and use it to configure rules to some specific VNICs.
You'll want to create a NSG inside the VCN, and then hop back to the Compute instance to configure it onto the instance.

Oracle explicitly [recommends](https://docs.oracle.com/en-us/iaas/Content/Network/Concepts/securityrules.htm#comparison) using NSGs because they will prioritise it for future enhancements, which I guess is sorta-but-not-a-deprecation for security lists.

Rules on the subnet's security lists and the VNIC's network security groups are taken as a union, not as two separate layers which must both agree.

There's definitely a lot more to unpack here, especially their load balancers, but this is enough to get a VM up and running and reasonably closed off.

If you use Tailscale, just ssh in to install Tailscale, tailscale up and join the machine to your tailnet.
Once that's done and tested, you can just shut off the tcp/22 ingress entirely at the security list or network security group, and get rid of all the brute force noise entirely.
Together with mosh, it feels like magic.

## Creating compute instances

The public key provided for SSH doesn't have to be RSA, [it can be ECDSA or Ed25519](https://docs.oracle.com/en-us/iaas/Content/Compute/Tasks/managingkeypairs.htm#public-key-format) on the default platform images, as long as it's in OpenSSH format.
You also don't need to specifically import it somewhere and then reference it, you can just paste it into an input box when creating a new instance.

Oracle Linux 8 / 7 and Ubuntu Linux 20.04 / 18.04 images are available for aarch64 right now.
Something interesting is [Oracle Linux Cloud Developer 8](https://docs.oracle.com/en/operating-systems/oracle-linux/oci/developer-image/), which looks like Oracle Linux 8 with a ton of dev dependencies pre-installed - Java and GraalVM of course, but also Python3, Node.js, Ruby, GCC, Go, Ansible, Terraform, and so on.
(Podman too, since it's an RHEL base.)

Since the VMs can be relatively resource-rich, it might be possible to run a desktop environment on it and VNC in, but latency might be an issue given that the nearest region is Tokyo, and the currently nearest region that's not Ampere A1 supply-constrained is Melbourne or Mumbai.

Something to investigate next: whether Docker Engine can run fine on Oracle Linux 8 given that it's using a potentially rather different kernel, or if Podman is a suitable replacement in 2021.
