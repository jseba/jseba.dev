---
title: Systems Software Engineer
organization: Cloudflare
organizationUrl: https://www.cloudflare.com
location: Remote, US
start: 2021-02-21
end: 2026-05-15
---

At Cloudflare, I was responsible for the Edge Container  (C4). This platform
started as a way for internal teams to deploy containers to Cloudflare's Edge
locations, and grew into the Workers Containers product that allowed customers
to attach containers to their Workers and run arbitrary code on Cloudflare.

I was the lead maintainer of the original v1 runtime, which was based on
HashiCorp Nomad for job scheduling, gVisor for a secure container sandbox and
a custom userspace network replacement for `libslirp4netns` using gVisor's
network stack (`slirpnetstack`). This grew from being a Python script gluing the
pieces together into a purpose built Nomad driver to provide better
maintainability and reliability. My first major feature work was to begin
migrating more of the container lifecycle management to containerd, starting
with image management and then moving on to the container configuration.

Before I could get started on moving the actual process lifecycle to
a containerd shim process, we were approached by Apple to provide them an API
that would allow them to run part of their Apple Private Relay system in some
key locations that they were short on capacity from other providers. As part of
this work, I completely re-architected the runtime to account for several of the
pain points we had encountered with the v1 approach.

Instead of all the binaries needed to run a container shipping in a single
package, which brought fun operational challenges when we needed to revert
a release that had a surprise backwards incompatible change in it, the v2
runtime used a system container image to provide the needed binaries. This
allowed a running container to keep the same versions until it stopped and then
clean up was just "delete the container and image".

Another key architectural shift was moving from gVisor as the default sandbox
platform to Firecracker. As were were also planning to use this opportunity to
build the new platform into a product for public customers, we wanted to
leverage full VM isolation, along with a separate untrusted kernel build
stripped of various subsystems, drivers and modules.

Apple also had the unique requirement that their applications could only run on
dedicated servers and were unwilling to budge on this point. While I wasn't the
person to do this operational work, I took over when they switched teams. There
were 7 locations that had these dedicated servers and I was the point of contact
for our SRE and datacenter operations teams to check in before any work was
done. I was also the point of contact for Apple's SRE team if they were seeing
any issues or wanted to make feature requests. One example that I shipped for
them was `cloud-init` support to allow them to dynamically configure their VMs
in a more standard way.

After Apple, we began working with other internal teams to drive adoption of the
new v2 platform and API. Workers AI was the team I helped bring onboard. I dug
into gVisor's new `nvproxy` subsystem, which allowed the sandbox to intercept
IOCTLs to the GPU and provide some basic filtering. With some prototyping in the
lab, we got their inferencing server to run and migrated their production
deployments over. While gVisor and `nvproxy` came with some stricter handling,
it gave them a more secure sandbox and our API gave them better control of the
containers.

In 2025, we launched Workers Containers as a public product for customers to
attach container images to their Workers, allowing them to run arbitrary code as
part of their Workers application. This required aligning our API with
Cloudflare's v4 API shape, which I took ownership of as part of the launch
requirements. I also did resource analysis to determine what we could offer
users in the public beta to use. In the end, not only did we have several
Workers teams building on top of our team's work (Workers AI, Workers Builds,
Workers Pipelines, Workers Sandboxes), we had several large customers ready to
scale rapidly on the product: Figma, Discord, Modal, even collaborations with
Anthropic.

With all this new feature work, it was becoming apparent that we hadn't had the
time allocated to managing our own developer experience. As the team grew, each
new engineer encountered the pain points that were our slow builds and the slow
*and* flaky tests, both locally and in CI. I had taken a couple stabs at seeing
if Bazel could help on the build front, especially with its caching and remote
building capabilities. Our team had a polyglot monorepo, with Go, Go+eBPF, Rust,
Rust+eBPF, Rust+Wasm, C+eBPF, Typescript/Workers, Python, Debian packages and
container images as artifacts. The first phase got to the point of being able to
build everything needed to produce the Debian packages. When I checked our build
times in CI, just this much had dropped build times by 30% on AMD64 and 50% on
ARM64, due to Bazel's much more in-depth build graph and parallelism than our
Makefile was able to manage.

I continued migrating more of the build system over to Bazel, moving from the
Workspace layout to `bzlmod` and writing custom modules to support our eBPF
builds better. The DevTools team had also recently hired an engineer that had
extensive Bazel experience and I was able to work with him to understand more
idiomatic build rules and tooling, such as getting a custom sysroot, not only
for more hermetic builds, but also being able to easily cross-compile. Being
able to drop the wasted time for each person on the team not only allowed
everyone to not have to babysit the CI jobs for their PRs, it was also just
a motivational boost to the team to have dependable and fast builds.

Focusing on the team's velocity allowed me time to jump into any issues and pair
up with anyone that needed a second pair of eyes. Being the longest tenured
person on the team (albeit not at Cloudflare), I had a large working knowledge
of the system and was able to point others to potential things to look into,
explain how a component works, how it was built, which teams or persons would
know more about another service, or even just jump on a screen share to dive
into code and rubber duck how some bug could have happened.

I also become the go-to person when there was a thorny issue that didn't have
much of anything to go on. Some of the more involved ones that I enjoyed solving
(click to see a synopsis of the root cause):

<ul class="bulletlist">
<li><details><summary>
A memory leak caused by an incorrect network rule class for slirpnetstack
causing the container to be OOM killed
</summary>
<p>
The container was performing thousands of DNS requests. By default,
slirpnetstack holds a UDP socket open for 10 seconds, since it can't really
know if the connection will be used repeatedly or once. This was causing
thousands of sockets, and socket buffers, to be held allocated, despite never
being used again by the application. By switching the network rule to a "UDP
RPC" class, slirpnetstack would close the socket once it got a response or the
10 second timeout.
</p></details></li>

<li><details><summary>
A memory leak due to continuous collection of gVisor debug logs after startup
</summary>
<p>
We captured gVisor's debug logs during startup into a buffer by passing gVisor
a pipe to use. Since we only needed these logs in case of a failure in gVisor,
we assumed that the flag "--debug-commands=boot" would only set the log level to
debug during its boot phase. That was an incorrect assumption and instead Python
was continuously growing that buffer the longer the application ran in the
sandbox, leading to an eventual OOM kill. I fixed this by swapping the handler
that read from the pipe for one that just discards the data, as well as flushing
the contents of the debug buffer, once we had determined that the sandbox was
past startup successfully.
</p></details></li>
<li><details><summary>
An unexpected ENOMEM fatal error during Firecracker's vCPU startup
</summary>
<p>
We were seeing sporadic failures of Firecracker not being able to launch the KVM
vCPU when it started a VM, with the error oddly being ENOMEM. Since we didn't
have any ideas on getting a reproducible test case for this, it took until it
was hitting consistently in CI for a few days to start digging in to solving it.
I set up retsnoop, a wrapper around bpftrace, to watch a few kernel
functions we were suspicious of and triggered some CI runs on a build agent that
was triggering the issue pretty consistently. It took several rounds to
eliminate some paths and drill into others, but it wasn't long to prove that
ENOMEM didn't mean something like "allocation failure". After jumping around
lots of inner functions in the VM start handlers in KVM, we found a thread
creation failure in the post-init phase. There was only one post-init thread
that KVM would create, a background worker to unshatter 4KB pages back into huge
pages. However, Firecracker did not support huge pages as a VMM, so it would
trigger a race condition where the background thread would fork from Firecracker
while there was an interrupt pending. A process cannot be forked while it has
unhandled signals, so the kernel would fail to create the thread. Why was there
an interrupt pending? Because the way to knock a thread running a KVM vCPU out
of the guest VM is to send it a signal so that the host thread blocked on the
IOCTL to run the vCPU gets an EINTR and can process the exit reason. In this
case, Firecracker would see the ENOMEM as the IOCTL result. Disabling huge
page support immediately fixed the issue, as it would bypass that background
worker thread being created in the first place.
</p></details></li></ul>
