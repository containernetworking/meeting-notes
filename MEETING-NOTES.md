# CNI meeting notes

_note_: the notes are checked in after every meeting to https://github.com/containernetworking/meeting-notes

## 2023-05-8

Agenda:
- [tomo] What should GC(de-INIT) do if INIT fails
- What parallel operations should be allowed?
    - Can you GC and ADD / DEL at the same time?
    - No way; the runtime has to "stop-the-world"
    - This makes sense, "network-wide" operations can be thought of as touching "every" attachment, and we don't allow parallel operations on an attachment.
    - 
- [cdc] Sorry, I owe a release
- [aojea] Evolve CNI to support new demands https://github.com/containernetworking/cni/issues/785#issuecomment-1532106958
    - We talk about the difference between the "configuration" API vs. the plugin API
    - Everyone seems to settle on CNI via the CRI
    - Casey drafted a version of this: https://hackmd.io/@squeed/cri-cni
    - 

## 2023-05-1

- Labor day
    

## 2023-04-24
- plugins will be released next week
- We review some PRs:
    - https://github.com/containernetworking/plugins/pull/829
    - https://github.com/containernetworking/plugins/pull/880
    - https://github.com/containernetworking/plugins/pull/888
    - https://github.com/containernetworking/plugins/pull/887
    - https://github.com/containernetworking/plugins/pull/885
    - https://github.com/containernetworking/plugins/pull/837 (needs rebase)
    - https://github.com/containernetworking/plugins/pull/883
    - https://github.com/containernetworking/cni/pull/967

## 2023-04-17
- KubeCon week, kind of quiet.
- PR 873 is merged.
    - Let's try and do a release in the next few weeks.
- https://github.com/containernetworking/cni/pull/981 has feedback, let's address it
- We button up some of the wording for GC
- Review some PRs. Merge some of them.


## 2023-04-10
Agenda:
- https://github.com/containernetworking/plugins/pull/873

## 2023-04-03
Agenda:
- https://github.com/containernetworking/plugins/pull/871

## 2023-03-27
Agenda:
- INIT/DE-INIT discussion
    - Should INIT/DEINIT be per-network, or per-plugin?
        - Probably per-network... but resources shared across networks? (see DEINIT discussion)
    - Serialization
        - Should the runtime be required to serialize calls to a plugin?
        - Eg can the plugin call INIT for two networks for the same plugin simultaneously, or not
    - Tomo asked about ordering guarantees; we shouldn't have double-INIT or double-DEINIT
    - If the config changes, do we DEINIT and then INIT with the new config? That could be very problematic.
        - Do we need an UPDATE for config change case?
        - What if the chain itself changes, plugins added/removed?
    - What if a plugin in the chain fails INIT?
        - What is the failure behavior if INIT fails?
        - When does the runtime retry INIT?
    - What if DEINIT fails?
        - Should GC be called?
    - Timing is pretty vague; when should DEINIT be called?
        - when the network config disappears (deleted from disk, removed from Kube API, etc)
        - when config disappears *and* all containers using the network are gone?
        - How should plugins handle deleting resources common to all networks? (eg plugin iptable chain)
            - Should we require that networks use unique resources to prevent this issue?
            - And/or punt to plugins that they just have to track/handle this kind of thing
    - "How do I uninstall a CNI plugin?"
        - CNI spec doesn't talk about any of this
        - (partly because we let the runtime decide where config is actually stored, even though libcni implements one method for doing this -- on-disk)
    - When config gets deleted, how do we invoke DEINIT with the now-deleted config?
        - Use cached config?
        - libcni would need to keep cached config after DEL; currently it doesn't
        - Keep a new kind of "network"-level config for this?
- PR review
    - https://github.com/containernetworking/plugins/pull/867 (MERGED)
    - https://github.com/containernetworking/plugins/pull/855 (MERGED)


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