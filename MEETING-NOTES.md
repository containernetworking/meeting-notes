# CNI meeting notes



## 2023-03-06

CNI v1.1 roadmap: https://github.com/containernetworking/cni/milestone/9


Agenda:
* CDC: bumping spec version to 1.1 -- https://github.com/containernetworking/cni/pull/967
* 1. GC and INIT for CNI 1.1
    * From last meeting, these are the current priorities
    * TODO: write spec, then implement in libcni
    * mz has opened issues #974 and #975
* 2. Network Plugin for containerd
* 3. Update meeting invite for new 
* Review https://github.com/containernetworking/cni/pull/963
* Review https://github.com/containernetworking/plugins/pull/844


Tomo asks for clarification about GC and INIT

INIT: runtime calls INIT **on a configuration** but without a container. It means "please prepare to receive CNI ADDs". For example, a plugin could create a bridge interface.

GC: two aspects to this discussion. Most of the GC logic would actually be in libcni, which already maintains a cache of configuration and attachments. The runtime would pass a list of still-valid attachments. Libcni could synthesize DEL for any "stale" attachments.

Separately, there could be a spec verb, GC, that would tell runtimes to delete any stale resources


We do some reviews.


Next week: Woah, there are a lot of PRs to review. Oof.