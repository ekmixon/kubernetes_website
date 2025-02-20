---
layout: blog
title: "Updated: Dockershim Removal FAQ"
linkTitle: "Dockershim Removal FAQ"
date: 2022-02-17
slug: dockershim-faq
aliases: [ '/dockershim' ]
---

**This is an update to the original [Dockershim Deprecation FAQ](/blog/2020/12/02/dockershim-faq/) article,
published in late 2020.**

This document goes over some frequently asked questions regarding the
deprecation and removal of _dockershim_, that was
[announced](/blog/2020/12/08/kubernetes-1-20-release-announcement/)
as a part of the Kubernetes v1.20 release. For more detail
on what that means, check out the blog post
[Don't Panic: Kubernetes and Docker](/blog/2020/12/02/dont-panic-kubernetes-and-docker/).

Also, you can read [check whether dockershim removal affects you](/docs/tasks/administer-cluster/migrating-from-dockershim/check-if-dockershim-deprecation-affects-you/)
to determine how much impact the removal of dockershim would have for you
or for your organization.

As the Kubernetes 1.24 release has become imminent, we've been working hard to try to make this a smooth transition.

- We've written a blog post detailing our [commitment and next steps](/blog/2022/01/07/kubernetes-is-moving-on-from-dockershim/).
- We believe there are no major blockers to migration to [other container runtimes](/docs/setup/production-environment/container-runtimes/#container-runtimes).
- There is also a [Migrating from dockershim](/docs/tasks/administer-cluster/migrating-from-dockershim/) guide available.
- We've also created a page to list
  [articles on dockershim removal and on using CRI-compatible runtimes](/docs/reference/node/topics-on-dockershim-and-cri-compatible-runtimes/).
  That list includes some of the already mentioned docs, and also covers selected external sources
  (including vendor guides).

### Why is the dockershim being removed from Kubernetes?

Early versions of Kubernetes only worked with a specific container runtime:
Docker Engine. Later, Kubernetes added support for working with other container runtimes.
The CRI standard was [created](/blog/2016/12/container-runtime-interface-cri-in-kubernetes/) to
enable interoperability between orchestrators (like Kubernetes) and many different container
runtimes.
Docker Engine doesn't implement that interface (CRI), so the Kubernetes project created
special code to help with the transition, and made that _dockershim_ code part of Kubernetes
itself.

The dockershim code was always intended to be a temporary solution (hence the name: shim).
You can read more about the community discussion and planning in the
[Dockershim Removal Kubernetes Enhancement Proposal][drkep].
In fact, maintaining dockershim had become a heavy burden on the Kubernetes maintainers.

Additionally, features that were largely incompatible with the dockershim, such
as cgroups v2 and user namespaces are being implemented in these newer CRI
runtimes. Removing support for the dockershim will allow further development in
those areas.

[drkep]: https://github.com/kubernetes/enhancements/tree/master/keps/sig-node/2221-remove-dockershim

### Can I still use Docker Engine in Kubernetes 1.23?

Yes, the only thing changed in 1.20 is a single warning log printed at [kubelet]
startup if using Docker Engine as the runtime. You'll see this warning in all versions up to 1.23. The dockershim removal occurs in Kubernetes 1.24.

[kubelet]: /docs/reference/command-line-tools-reference/kubelet/

### When will dockershim be removed?

Given the impact of this change, we are using an extended deprecation timeline.
Removal of dockershim is scheduled for Kubernetes v1.24, see [Dockershim Removal Kubernetes Enhancement Proposal][drkep].
The Kubernetes project will be working closely with vendors and other ecosystem groups to ensure
a smooth transition and will evaluate things as the situation evolves.

### Can I still use Docker Engine as my container runtime?

First off, if you use Docker on your own PC to develop or test containers: nothing changes.
You can still use Docker locally no matter what container runtime(s) you use for your
Kubernetes clusters. Containers make this kind of interoperability possible.

Mirantis and Docker have [committed][mirantis] to maintaining a replacement adapter for
Docker Engine, and to maintain that adapter even after the in-tree dockershim is removed
from Kubernetes. The replacement adapter is named [`cri-dockerd`](https://github.com/Mirantis/cri-dockerd).

[mirantis]: https://www.mirantis.com/blog/mirantis-to-take-over-support-of-kubernetes-dockershim-2/

### Will my existing container images still work?

Yes, the images produced from `docker build` will work with all CRI implementations.
All your existing images will still work exactly the same.

#### What about private images?

Yes. All CRI runtimes support the same pull secrets configuration used in
Kubernetes, either via the PodSpec or ServiceAccount.

### Are Docker and containers the same thing?

Docker popularized the Linux containers pattern and has been instrumental in
developing the underlying technology, however containers in Linux have existed
for a long time. The container ecosystem has grown to be much broader than just
Docker. Standards like OCI and CRI have helped many tools grow and thrive in our
ecosystem, some replacing aspects of Docker while others enhance existing
functionality.

### Are there examples of folks using other runtimes in production today?

All Kubernetes project produced artifacts (Kubernetes binaries) are validated
with each release.

Additionally, the [kind] project has been using containerd for some time and has
seen an improvement in stability for its use case. Kind and containerd are leveraged
multiple times every day to validate any changes to the Kubernetes codebase. Other
related projects follow a similar pattern as well, demonstrating the stability and
usability of other container runtimes. As an example, OpenShift 4.x has been
using the [CRI-O] runtime in production since June 2019.

For other examples and references you can look at the adopters of containerd and
CRI-O, two container runtimes under the Cloud Native Computing Foundation ([CNCF]).

- [containerd](https://github.com/containerd/containerd/blob/master/ADOPTERS.md)
- [CRI-O](https://github.com/cri-o/cri-o/blob/master/ADOPTERS.md)

[CRI-O]: https://cri-o.io/
[kind]: https://kind.sigs.k8s.io/
[CNCF]: https://cncf.io

### People keep referencing OCI, what is that?

OCI stands for the [Open Container Initiative], which standardized many of the
interfaces between container tools and technologies. They maintain a standard
specification for packaging container images (OCI image-spec) and running containers
(OCI runtime-spec). They also maintain an actual implementation of the runtime-spec
in the form of [runc], which is the underlying default runtime for both
[containerd] and [CRI-O]. The CRI builds on these low-level specifications to
provide an end-to-end standard for managing containers.

[Open Container Initiative]: https://opencontainers.org/about/overview/
[runc]: https://github.com/opencontainers/runc
[containerd]: https://containerd.io/

### Which CRI implementation should I use?

That’s a complex question and it depends on a lot of factors. If Docker is
working for you, moving to containerd should be a relatively easy swap and
will have strictly better performance and less overhead. However, we encourage you
to explore all the options from the [CNCF landscape] in case another would be an
even better fit for your environment.

[CNCF landscape]: https://landscape.cncf.io/card-mode?category=container-runtime&grouping=category

### What should I look out for when changing CRI implementations?

While the underlying containerization code is the same between Docker and most
CRIs (including containerd), there are a few differences around the edges. Some
common things to consider when migrating are:

- Logging configuration
- Runtime resource limitations
- Node provisioning scripts that call docker or use docker via it's control socket
- Kubectl plugins that require docker CLI or the control socket
- Tools from the Kubernetes project that require direct access to Docker Engine
  (for example: the deprecated `kube-imagepuller` tool)
- Configuration of functionality like `registry-mirrors` and insecure registries 
- Other support scripts or daemons that expect Docker Engine to be available and are run
  outside of Kubernetes (for example, monitoring or security agents)
- GPUs or special hardware and how they integrate with your runtime and Kubernetes

If you use Kubernetes resource requests/limits or file-based log collection
DaemonSets then they will continue to work the same, but if you’ve customized
your `dockerd` configuration, you’ll need to adapt that for your new container
runtime where possible.

Another thing to look out for is anything expecting to run for system maintenance
or nested inside a container when building images will no longer work. For the
former, you can use the [`crictl`][cr] tool as a drop-in replacement (see [mapping from docker cli to crictl](https://kubernetes.io/docs/tasks/debug-application-cluster/crictl/#mapping-from-docker-cli-to-crictl)) and for the
latter you can use newer container build options like [img], [buildah],
[kaniko], or [buildkit-cli-for-kubectl] that don’t require Docker.

[cr]: https://github.com/kubernetes-sigs/cri-tools
[img]: https://github.com/genuinetools/img
[buildah]: https://github.com/containers/buildah
[kaniko]: https://github.com/GoogleContainerTools/kaniko
[buildkit-cli-for-kubectl]: https://github.com/vmware-tanzu/buildkit-cli-for-kubectl

For containerd, you can start with their [documentation] to see what configuration
options are available as you migrate things over.

[documentation]: https://github.com/containerd/cri/blob/master/docs/registry.md

For instructions on how to use containerd and CRI-O with Kubernetes, see the
Kubernetes documentation on [Container Runtimes].

[Container Runtimes]: /docs/setup/production-environment/container-runtimes/

### What if I have more questions?

If you use a vendor-supported Kubernetes distribution, you can ask them about
upgrade plans for their products. For end-user questions, please post them
to our end user community forum: https://discuss.kubernetes.io/.

You can discuss the decision to remove dockershim via a dedicated
[GitHub issue](https://github.com/kubernetes/kubernetes/issues/106917).

You can also check out the excellent blog post
[Wait, Docker is deprecated in Kubernetes now?][dep] a more in-depth technical
discussion of the changes.

[dep]: https://dev.to/inductor/wait-docker-is-deprecated-in-kubernetes-now-what-do-i-do-e4m

### Can I have a hug?

Yes, we're still giving hugs as requested. 🤗🤗🤗
