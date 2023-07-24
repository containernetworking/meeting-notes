# CNI meeting notes

_note_: the notes are checked in after every meeting to https://github.com/containernetworking/meeting-notes

An editable copy is hosted at https://hackmd.io/jU7dQ49dQ86ugrXBx1De9w. Feel free
to add agenda items there

## 2023-07-24

- Multinetwork report from Pete White
    - MikeZ is meditating on how it fits in with the CRI
- STATUS verb (PR [1003](https://github.com/containernetworking/cni/pull/1003)) (Issue [859](https://github.com/containernetworking/cni/issues/859))
- The problem: plugins don't know whether they should use legacy (write file when ready) behavior versus rely on STATUS
- Potential solutions:
    - CRI signals whether or not it supports STATUS via config file or something (discussed in issue [927](https://github.com/containernetworking/cni/issues/927))
        - Biggest blocker for a feature file is downgrades
    - Add an additional directory, "cniv1.1", that is only read by cni clients
    - Plugins write a file that is invalid for v1.0, but valid for v1.1, when status is failing
    - Switch to a new directory entirely
    - New filename suffix (.conflistv1.1)
        - Not a terrible idea
- reviewers wanted: https://github.com/containernetworking/plugins/pull/921
- Review of PRs, looking in pretty good shape

## 2023-07-17

- Regrets: Tomo

## 2023-07-03

- CNI Route type and MTU https://github.com/containernetworking/cni/issues/1004
    - previous effort stalled out at https://github.com/containernetworking/cni/pull/831
    - no opposition, let's try and get this in for v1.1
- Easy-to-review PR list (by Tomo)
    * https://github.com/containernetworking/plugins/pull/902
        * includes https://github.com/containernetworking/plugins/pull/920
    * https://github.com/containernetworking/plugins/pull/918
    * https://github.com/containernetworking/plugins/pull/912
    * https://github.com/containernetworking/plugins/pull/911
    * https://github.com/containernetworking/plugins/pull/907
    * https://github.com/containernetworking/plugins/pull/897 
    * https://github.com/containernetworking/cni.dev/pull/122
- AI: Tomo review: https://github.com/containernetworking/plugins/pull/903
- Finalize CNI STATUS verb
    - https://github.com/containernetworking/cni/pull/1003


## 2023-06-26
- Let's review some PRs
- multi-network chit chat

## 2023-06-19
- Continuing STATUS editing
- Tomo asks about version divergence between a plugin and its delegate. We talk about version negotiation.
- Circle back for CNI+CRI
- Update: we file https://github.com/containernetworking/cni/pull/1003

## 2023-06-12
- Brief discussion about CNI and CRI for old time's sake
- We will work on initial wording of the [STATUS](https://github.com/containernetworking/cni/issues/859) verb
    - presented in [sig-network meeting last Thursday](https://docs.google.com/document/d/1_w77-zG_Xj0zYvEMfQZTQ-wPP4kXkpGD8smVtW_qqWM)


### Status

#### Questions:

1. What do we return? Just non-zero exit code? OR JSON type?
    - We should return a list of conditions
    - Conditions: (please better names please)
        - AddReady
        - RoutingReady (do we need this)?
        - ContainerReady
        - NetworkReady
2. Should we return 0 or non-zero?
    - after a lot of discussion, we come back to returning nothing on success and just error


#### Draft spec:

#### `STATUS`: Check plugin status
`STATUS` is a way for a runtime to determine the readiness of a network plugin.

A plugin must exit with a zero (success) return code if the plugin is ready to service ADD requests. If the plugin is not able to service ADD requests, it must exit with a non-zero return code and output an error on standard out (see below).

The following error codes are defined in the context of `STATUS`:

- 50: The plugin is not available (i.e. cannot service `ADD` requests)
- 51: The plugin is not available, and existing containers in the network may have limited connectivity.

Plugin considerations:
- Status is purely informational. A plugin MUST NOT rely on `STATUS` being called.
- Plugins should always expect other CNI operations (like `ADD`, `DEL`, etc) even if `STATUS` returns an error.`STATUS` does not prevent other runtime requests.
- If a plugin relies on a delegated plugin (e.g. IPAM) to service `ADD` requests, it must also execute a `STATUS` request to that plugin. If the delegated plugin return an error result, the executing plugin should return an error result.

**Input:**

The runtime will provide a json-serialized plugin configuration object (defined below) on standard in.

Optional environment parameters:
- `CNI_PATH`



#### References:

CRI API 
https://github.com/kubernetes/cri-api/blob/e5515a56d18bcd51b266ad9e3a7c40c7371d3a6f/pkg/apis/runtime/v1/api.proto#L1480C1-L1502

```
message RuntimeCondition {
    // Type of runtime condition.
    string type = 1;
    // Status of the condition, one of true/false. Default: false.
    bool status = 2;
    // Brief CamelCase string containing reason for the condition's last transition.
    string reason = 3;
    // Human-readable message indicating details about last transition.
    string message = 4;
}
```

## 2023-06-05
- PTAL: https://github.com/containernetworking/cni.dev/pull/119
    - cdc observes we're due for some website maintainance
- [aojea] - more on CNI status checks
            - dryRun option?
- We would really like the STATUS verb
    - It would solve an annoying user situation
    - Let's do it.
    - Next week we'll sit down and hammer out the spec.
    - https://github.com/containernetworking/cni/issues/859
        - strawman approach: kubelet (networkReady) -- CRI --> container_runtime -- (exec) --> CNI STATUS
        - runtimes should use the version to use the new VERB  
- See if GC spec needs any changes: https://github.com/containernetworking/cni/pull/981
    - We need wording for paralleization:
    - The container runtime must not invoke parallel operations for the same container, but is allowed to invoke parallel operations for different containers. This includes across multiple attachments.
    - **Exception**: The runtime must exclusively execute either _gc_ or _add_ and _delete_. The runtime must ensure that no _add_ or _delete_ operations are in progress before executing _gc_, and must wait for _gc_ to complete before issuing new _add_ or _delete_ commands.

## 2023-05-22


- [aojea] follow CNI conversation from 2023-05-08
    - hard to land important changes on CRI API
    - For configuration understand existing kubelet subsystems like "kubelet devices plugins" and "DirectResourceAllocation" to align and being able to plug CNI
        - New KEP to use QoS https://github.com/kubernetes/enhancements/pull/3004#discussion_r1179519017
        - We look at the DRA design to see how it fits CNI: https://kubernetes.io/docs/concepts/scheduling-eviction/dynamic-resource-allocation/
        - KEP: https://github.com/kubernetes/enhancements/blob/master/keps/sig-node/3063-dynamic-resource-allocation/README.md
            - (No mention of sandbox creation)
        - Could we make a CNI DRA driver?
            - Only if the lifecycle matches exactly what we need
                - We can't create network interfaces until the network namespace exists
                - the network namespace is created with the PodSandbox (a.k.a pause container)
                - Current order: 
                    1. Pod created.
                    2. kubelet calls CreatePodSandbox CRI method...
                    3. containerd creates netns
                    4. containerd calls CNI ADD
                    5. CreatePodSandbox done, Containers now created and started
        - The DRA object model looks really good
            - it has the ability to have arbitrary parameters a.k.a. ResourceClaimTemplate, which would be nice
            - what if we have a CNI plugin that needs devices created from additional DRA providers?
    - For improving supportability improve kubelet check https://github.com/containernetworking/cni/issues/859
    - MZappa great diagram https://drive.google.com/file/d/1TTTM2YP67J4mjG4BchNyEmEi1nXKHtfN/view
- [cdc] GC is stalled, but should have time in 2-3 weeks to work on it
- [cdc] anyone have time to work on status? https://github.com/containernetworking/cni/issues/859
- let's turn off the github auto-staler, it's being mean

## 2023-05-15

- [cdc] we cut a release! yay!
- [cdc] Chatting with Multus implementers about [version negotiation](https://github.com/containernetworking/cni/issues/941)
    - Should we just do this automatically? We already have the VERSION command...
    - Everyone is uncomfortable with "magic" happening without someone asking for it
    - Original proposal was for cniVersions array.
    - What if we added "auto" as a possible cni version?
    - Concern: how do we expose what version we decided to use?
    - Or what if we just autonegotiate all the time
        - YOU GET A NEW VERSION! YOU GET A NEW VERSION!
    - 


## 2023-05-08

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