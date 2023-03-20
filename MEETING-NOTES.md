# CNI meeting notes

_note_: the notes are checked in after every meeting to https://github.com/containernetworking/meeting-notes

## 2023-03-20
Agenda:
- Brief discussion about some sort of SYNC
    - usual chained plugin issue
- Let's try and write the spec for GC.
- We do! https://github.com/containernetworking/cni/pull/981 


## 2023-03-13
Agenda:

round of introductions

- STATUS for v1.1?
- Multi-network? Doug not present
- Network Plugin for containerd(Henry)
    - initial problem: trying to solve leaking resources (sometimes cleanup fails)
    - led to GC proposal, as well as GC() method on libcni
    - https://github.com/containerd/containerd/pull/7947
- Does it make sense for some kind of idempotent Sync()
    - Challenges:
        - hard to make fast / high overhead
        - chained plugins make this difficult, might have flapping interfaces
        - pushes a lot of overhead on the plugins
    - Does INIT solve this? Not really; runtime might not call INIT when it's needed
- What do we do on failed CHECK?
    - Should we allow for ADD after failed CHECK
    - Chained plugins make this difficult, but we could change the spec
- (tomo) Consider a bridge - when should we delete it?
    - even though no container interface in bridge, user may add some physical interface to the pod
    - bridge plugin does not have lock mechanism for multiple container
    - we considered a DEINIT verb, but it didn't seem useful
- Let's do some reviews. Oops, we run out of time
- Should we formalize "how to interact with libcni"?
    - What are the expectations for how configuration files are dropped in? (e.g. permission error)
- 


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