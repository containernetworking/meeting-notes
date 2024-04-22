# CNI meeting notes

_note_: the notes are checked in after every meeting to https://github.com/containernetworking/meeting-notes

An editable copy is hosted at https://hackmd.io/jU7dQ49dQ86ugrXBx1De9w. Feel free
to add agenda items there.

## 2024-04-22

- We chat about k8s and the DRA
    - Antonio working on adding netdevs to oci spec
    - [ben] thinking aloud, if netdev management was done here,
      would that mean that CNI plugins might become like FINALIZE variants (more or less), basically?
- GC doubts: should we add a DisableGC option?
    - use-case: one network, multiple runtimes

- dropin (updated, PR review) https://github.com/containernetworking/cni/pull/1052
- ocicni STATUS and GC PRs: https://github.com/cri-o/ocicni/pull/196, https://github.com/cri-o/ocicni/pull/197

## 2024-04-15

- Tagging v1.1: minor cleanup needed
- v1.2 milestone:
    - dropins [https://github.com/containernetworking/cni/pull/1052]
    - finalize
    - init?
    - gRPC
- FINALIZE discussion
    - use-cases:
        - ensuring routes
        - inserting iptables rules (e.g. service-mesh proxy)
        - ECMP (eaugh)
    - lifecycle:
        1. ADD (network 1)
        2. ADD (network 2)
        3. FINALIZE (network 1)
        4. FINALIZE (network 2)
        5. later... CHECK
    - Configuration source:
        - in-line from config
        - specific FINALIZE configuration?
            - maybe not needed.
            - cri-o / containerd could have a magic dropin directory?
        - What if a configuration only has FINALIZE plugins?
            - then we don't ADD, just FINALIZE
    - What is passed to the plugin(s)?
        - We could pass *all* results of all networks
            - Tomo: this is complicated (and plugin could get that by netlink), let's not
            - fair enough
        - CNI_IFNAME?
        - Standard prevResult?
    - What is returned?
        - Not allowed to produce result?
    - Philosophical question: is FINALIZE "network-level" or "container-level"
        - does it get IFNAME? PrevResult?
        - Homework: come up with more use-cases.
    - 
- [minor] readme PR: https://github.com/containernetworking/cni/pull/1081
- [minor] licensing question https://github.com/containernetworking/plugins/pull/1021
- 

## 2024-04-08

PR
- https://github.com/containernetworking/cni/pull/1054 [approved]
- Merge today! (or reject today), it is so big and hard to keep in PR list...: https://github.com/containernetworking/plugins/pull/921 [merged]
- https://github.com/containernetworking/cni/pull/1052 [for 1.2]
  - Q: should this be a config or not? https://github.com/containernetworking/cni/pull/1052#discussion_r1551653317
  - A: Consensus seems to be leave in the flag, change the name, maybe add docs around how libcni implements the spec with file loading.
  - [dougbtv] working on NetworkPlumbing WG proposal to attach secondary networks more granularly
  - [ben] that sounds good to me, I want to keep this very tightly scoped to avoid main config file contention, more granular behaviors should be handled elsewhere, 100%

Discussion
- Should 'ready' be both CNI network configuration and binaries present? Right now its just the network config. [Zappa]
    - [tomo] agreed but need to care UX (how to told this to user)
        - k8s node object should have error messages ()
    - [ben] since we can't check much about the binary, this is necessarily a simplistic check that a file exists at the binary path - it could still fail to execute, or be partially copied, etc
    - [cdc] I wrote the .Validate() libcni function some time ago for this, use it :-)
- CNI X.Y [Zappa/Casey]
- Finalize/Init Verb [Zappa]
- Loopback fun [Zappa]

## 2024-04-01
No CNI calls today due to Easter holiday, dismiss!

- Headsup:
    - [squeed] Last call for tagging libcni for v1.1.0 and let's conclude next week meeting, 4/8!!

## 2024-03-25

KubeCon update:
- It happened
- Something something DRA?
- Antonio wants to move network establishment in to device plugins
- Casey & Tomo's presentation: https://docs.google.com/presentation/d/1eCOFcro7dc9iq3VS-31045EsVUstfqmF/edit?usp=sharing&ouid=110611166904085433395&rtpof=true&sd=true
CNI v1.1
- https://github.com/containernetworking/plugins/pull/1021
    - please review
- ContainerD STATUS support is in progress
- Need to do cri-o, multus
- v1.1 is feature frozen? sure!
    - Milestone is completed anyway as of today, call it.

CNI v1.2 ideas:
- drop-in directories
    - Issue: https://github.com/containernetworking/cni/issues/928
    - PR: https://github.com/containernetworking/cni/pull/1052
- Interface metadata
    - TODO: file issue
- FINALIZE verb
    - Problem: we have no inter-container lifecycle guarantees
    - use-cases:
        - Ensuring route table is in a defined state
        - insering iptables rules for proxy sidecar (e.g. Envoy) chaining
    - Biggest problem: how to configure?
        - /etc/cni/net.d/_finalize.d/00_istio.{fconf,conf}
        - Do we use a standardized directory that applies to *all* plugins?
        - Do we have finalizers per-network, or finalizers after all networks?
            - Ben: Do we need both, or could we get away with just global finalizers?
            - Casey, others: We might (for some use cases) actually need per-network.
            - Which one is less footgunny? Would running finalizers per-network "hide" global state that might make finalizers more likely to break things?
        - Multus is also trying to add a finalizer pattern, for multiple CNIs - consider how this works as well.

```
/etc/cni/net.d/00-foobar.conf
{
  // usual CNI
  plugins: [],
  finalize_plugins: [{"type":"force_route"}] // type a: in CNI config
}

// type b: drop-in directory
// should we change '_finalize.d' as configurable?

/etc/cni/net.d/_finalize.d/00-foobar.conf
{
  // one finalizer
}

/etc/cni/net.d/_finalize.d/99-istio.conf
{
  // put istio, that wants to be final!
}

/etc/cni/net.d/_finalize.d/999-barfoo.conf
{
  // oh, sorry, it is the actual final guy!
}
```


## 2024-03-11
Work on outline for Kubecon project update.

CNI: update, what's next
- CNI basic overview
    - CNI is an execution protocol, run by the runtime
- CNI 1.0
    - '.conf' file (i.e. single plugin conf) is removed! 
    - interface has no longer 'version' field (of address)
- new version CNI 1.1 Update!
- new verbs
    - STATUS: replaces "write or delete CNI config file" to signify plugin readiness
    - GC: clean up leftover resources
- new fields
    - route MTU, table, scope
    - interface MTU, PCIID, socket path
- cniVersions array
    - Configuration files can now support multiple supported versions
- Will be released shortly
    - implementing v1.1 in plugins now
    - cri-o, containerd also in-progress
    - Community question: Should the CRI closer reflect the CNI?
        - a.k.a. how can I use these shiny new fields?
        - our opinion: heck yeah! Let's make the CRI more fun
        - Should we expand v1/PodStatus? MultiNetwork WG is proposing this, :thumbsup: 
- device capability (not 1.2, but whatever)
    - Standardized way for devices to be passed down from kubelet -> runtime -> plugin (e.g. SR-IOV)
    - Still no way for plugins to say they *need* a certain device
        - Looking for ways to tie config back in to the scheduler
        - complicated! help wanted!

what's next (for v1.2)
- drop-in directories (definite)
    - no more manipulating CNI configuration files
    - isto community contribution
- Interface metadata, (likely)
    - We prefer to have these as fields
        - We generally have a low threshold for adding a field
    - But some things are just too weird even for us :p
- FINALIZE (maybe??)
    - some kind of post-ADD "finalize" plugin?
    - called after every ADD of every interface
    - possible use case: resolve route conflicts
- INIT (stretch)
    - Called before first ADD
    - Not container-specific
    - Really sorry about this one, Multus
    - Nasty lifecycle leaks
- Dynamic reconfiguration (vague idea)
    - Spec says ADD is **non-idempotent**
    - But there's no reason this has to be the case
    - Do you want this? Get involved!!


So, what about KNI?
- KNI is, and is not a replacement for CNI
    - e.g. KNI proposed to be responsible for isolation domain management
- KNI extensibility is still a work in progress
- If KNI merges, the default impl will be containerd / cri-o wrapping CNI


Calls to action:
- This is a dynamic area of k8s right now, lots of things are being proposed
- CNI fits in a complicated ecosystem (k8s, CNI, CRI, runtimes)
- There is a lot of room for improvement, but it reaches across a lot of concerns
- We are all busy, we can't be in all projects at all times
- Reach out! Let's make features people use!





## 2024-03-04

- PR Review:
    - https://github.com/containernetworking/cni/pull/1069
    - https://github.com/containernetworking/cni/pull/1052
        - getting close to the end :-)
    - 

- Discussions:
    - CNI cached directory questions 
        - Why is the cached directory not on volatile storage?
            - because we try and pass the same values to DEL as to ADD, even after a reboot
        - But we sometimes fail to delete because of invalid cache :-)
            - We should handle this case gracefully, same as a missing cache file
        - Casey wonders: How do we handle this for GC?????
    - [Zappa] go-cni PR for Status [draft]
    - [Zappa] go-cni PR for additional fields [draft]
    - need same work for CRI-O, where are you, Jaime?
    - [cdc] Device conventions: https://github.com/containernetworking/cni/issues/1070
    - [cdc] working on CNI v1.1 for plugins, slowly
    - Kubecon Talk: What did we do????????
        - CNI v1.0
            - no .conf files
            - CHECK
        - CNI v1.1 -- lots of new features
            - GC, SATUS
            - - more types
        - KNI?
        - What do people actually want? What verbs should come next?
            - FINALIZE?
        - Hey multi-networking, please figure out how to configure CNI via k8s API plz thanks


## 2024-02-26

- PR Review:
    - [Draft] https://github.com/containernetworking/cni/pull/1069
        - Need to provide definitions in the spec for both fields
    - [LFR] https://github.com/containernetworking/cni/pull/1052
        - Updated to invert flag logic (enabled by default)

- Discussions:
    - [Zappa] Noticed uptick of issues around /var/lib/cni/results 
        - Unable to delete pods (+ip address)
    - [cdc] Help wanted: plugins v1.1 impl.
        - Open issues: https://github.com/containernetworking/plugins/milestone/6
    - [cdc] Any ideas on device convention? https://github.com/containernetworking/cni/issues/1070
    - [aojea] STATUS implementation
        - I have some volunteers but I need to provide some guidance
            - Casey/Zappa have branches for this 

## 2024-02-19

- US Holiday (ish)
- PR review:
  - https://github.com/containernetworking/cni/pull/1062
  - https://github.com/containernetworking/cni/pull/1068
      - both look good, need integer pointers and the implementation in plugins
  - https://github.com/containernetworking/cni/pull/1052
  - https://github.com/containernetworking/cni/pull/1060 (mtu on interface).
  - https://github.com/containernetworking/plugins/pull/1003
- CNI v1.1 progress
    - 

## 2024-02-12

- PR's
  - Support loading plugins from subdirectories: https://github.com/containernetworking/cni/pull/1052
    - Comments addressed, this now adds a new opt-in config flag rather than forcing more drastic changes to the config spec. PTAL, need Casey/Dan to do a final pass
- New issues
- Metadata proposal (fields vs map)
    - PCIID (to Interface structure)
    - SocketPath (to Interface structure)
    - MTU (to Interface structure)
    - Route table ID (to Route structure)
        - https://github.com/containernetworking/cni/issues/1061
        - https://github.com/containernetworking/cni/pull/1062
    - Route Scope/Flag (to Route structure)
        - https://github.com/containernetworking/cni/issues/598
    - Or map[string] string or go with both approaches? 
- PR review:
    - https://github.com/containernetworking/plugins/pull/1002 (tomo will review)
    - https://github.com/containernetworking/plugins/pull/1003 (tomo will review)
    - 

## 2024-02-05
- PR's
    - Support loading plugins from subdirectories: https://github.com/containernetworking/cni/pull/1052
        - Comments addressed, this now adds a new opt-in config flag rather than forcing more drastic changes to the config spec.
        - PTAL, need Casey/Dan to do a final pass

- discuss: 
    - SocketPath/DeviceID aka metadata (need Casey/Dan)
    - KNI? (need Casey/Dan)
    - CNI 2.0 / Multi-Network / KNI / DRA / [NRI](https://github.com/containerd/nri) / ? meetup at KubeCon?
        - Plan/strategy? KNI will probably need a plugin strategy, will probably support CNI plugins, but going forward could support a better/smarter plugin interface.
    - https://github.com/containernetworking/cni/issues/1061
    - https://github.com/containernetworking/cni/issues/598

## 2024-01-29
- PR's
    - Add Github action to build binary at tag release https://github.com/containernetworking/plugins/pull/1000
    - bridge: Enable disabling bridge interface: https://github.com/containernetworking/plugins/pull/997
    - Initial attempt to support loading plugins from subdirectories: https://github.com/containernetworking/cni/pull/1052
        - Hypothetical impl there now, PTAL & offer opinions
- (if time, otherwise defer) Conf vs Conflist in libcni [bl]
    - only conflist is supported by current spec, and that has been true for some time.
    - looking for historical context on current state 
    - should we mark conf as deprecated, and remove on major bump? Given the above, that seems reasonable.
    - Decision: Even though the pre-1.0.0 format is deprecated, we cannot remove it yet.
- Metadata 
    - Socket Path
    - pciid
- Convention for results from device plugins [cdc, others]
    - Should we make it easy for containerd / crio to pass devices ti plugins?
    - Prefer to phased approach
        - phase1: Just formalize in cni repo and change multus/sr-iov
        - phase2: Integrate cni runtime as well as container runtime
            - CDI
- Add MTU to the interface results in the CNI
    - https://github.com/containernetworking/cni/blob/main/pkg/types/100/types.go#L272
- annoying that spec v1.0 is library v1.1
    - Do we split spec repo and library?
    - What if we skip v1.1 and move to v1.2?
- KNI
    - 

## 2024-01-22
- PR's
    - Add Github action to build binary at tag release https://github.com/containernetworking/plugins/pull/1000
    - Initial attempt to support loading plugins from subdirectories: https://github.com/containernetworking/cni/pull/1052
        - Hypothetical impl there now, PTAL & offer opinions
- (if time, otherwise defer) Conf vs Conflist in libcni [bl]
    - only conflist is supported by current spec, and that has been true for some time.
    - looking for historical context on current state 
    - should we mark conf as deprecated, and remove on major bump? Given the above, that seems reasonable.
- Metadata 
    - Socket Path
    - pciid

## 2024-01-15
- US Holiday, cdc out too

## 2024-01-08
- Welcome to the New Year!
- PR:
    - https://github.com/containernetworking/plugins/pull/844
        - from the last comment: It's unfortunate that this as been pending so long, it seems like the maintainers are ignoring this or don't find value in this PR 😭
        - May need to decision (to include/not include?)
    - https://github.com/containernetworking/plugins/pull/921 (local bandwidth)
        - Tomo and Mike approved
        - We will ship this in the next release.
- CNI 1.1
    - tag -rc1?
    - implement in plugins
    - implement in runtimes (go-cni cri-o)
- What belongs in CNI v1.2:
- Metadata [proposal](https://github.com/containernetworking/cni/issues/1050)
    - conclusion: this seems worthy, let's expore it
    - come up with some use cases, draft a SPEC change
- multi-ADD / idempotent ADD / reconfigure
    - wellllll, k8s doesn't have network configuration, so how can we *reconfigure* what we don't have?
    - Pete to write up proposal?
- [bl] QQ: Config versioning 
    - Do we distinguish between config file and config (in-mem) schema versions, or are they always 1:1 in the spec?



## 2023-12-18
- PR:
    - https://github.com/containernetworking/cni/pull/1039 (remind)
    - https://github.com/containernetworking/plugins/pull/921 (follow-up to MikeZ)
- Metadata Proposal Discussion [Zappa]
    - https://github.com/containernetworking/cni/issues/1050
    - PR: https://github.com/containernetworking/cni/pull/1051
- New definitions specification update [Zappa]
    - https://github.com/containernetworking/cni/pull/1053
- ready to cut 1.1?
    - https://github.com/containernetworking/cni/milestone/9 was clear
- Subdir-based plugin conf [Ben L.] [Post 1.1/Jan]
    - Issue: https://github.com/containernetworking/cni/issues/928
        - Spec rewrite PR to start convo around details (mostly spec rev. versus none): 
            - https://github.com/containernetworking/cni/pull/1052

## 2023-12-11
- PR:
    - https://github.com/containernetworking/cni/pull/1039 (remind)
    - https://github.com/containernetworking/plugins/pull/921 (follow-up to MikeZ)
- v1.1 blockers https://github.com/containernetworking/cni/milestone/9
    - only big question is cni versions (https://github.com/containernetworking/cni/pull/1010) MERGED
    - and route MTU: https://github.com/containernetworking/cni/pull/1041
- Metadata Proposal Discussion [Zappa]
- Review https://github.com/jasonliny/tag-security/blob/main/assessments/projects/cni/self-assessment.md
    - Idea: we should not allow delegated plugins outside of CNI_BINDIR
    - We talk about how delegated plugins are totally dangerous and we should be more careful


## 2023-12-04
- [cdc] need to cut a plugins release
    -  (grump about CVE scanners)
    -  last PR sweep
    -  cdc to cut release shortly
- CNCF TAG Security: working on a self-assessment for CNI, which will be submitted to TAG Security (details here:  https://github.com/cncf/tag-security/issues/1102)
    - We would appreciate some feedback, as we suspect there may be misunderstandings or inaccurate information in our current draft (https://github.com/jasonliny/tag-security/blob/main/assessments/projects/cni/self-assessment.md)

## 2023-11-27
- CNI 2.0 Note Comparison [Zappa]
    - [KNI Design proposal](https://docs.google.com/document/d/1Gz7iNtJNMI-zKJhaOcI3aflPCx3etJ01JMxzbtvruKk/edit/)
    - Mike Z is going to do further work on gRPC in the container runtime to see if it fits conceptually once its in there.
    - Key points:
        - Something like this exists in some private forks
        - 
- Extra metadata on CNI result [Zappa]
- KubeCon EU Presentation? (skipped / next time) (From Tomo)
    - Tomo is out but Doug said he'd bring this up.
    - usual (not community one) CFP deadline is Nov 26.
    - CNI1.1 talk?
    - No one has the dates for the maintainer track CFP due date, but Casey's going to ask around about it.
- CNI 2.0 requirements discussion (We should still discuss 1.1)
- [mz] Extra metadata on CNI result

## 2023-11-20
- KubeCon EU Presentation? (skipped / next time)
    - usual (not community one) CFP deadline is Nov 26.
    - CNI1.1 talk?
    - CNI 2.0 requirements discussion (We should still discuss 1.1) [Zappa]
    - Extra metadata on CNI result
- [Tomo] PR / Issue (follow-up)
    - https://github.com/containernetworking/cni/pull/1038 (simple fix)
    - https://github.com/containernetworking/cni/pull/1039 (supersedes PR#1035)
    - https://github.com/containernetworking/plugins/pull/921 (support exclude subnets in bandwidth plugin)
    - https://github.com/containernetworking/cni.dev/pull/130 (bandwidth plugin doc change)
- [MikeZ] CNI 2.0 discussion

## 2023-11-13
- [Tomo] PR / Issue
    - https://github.com/containernetworking/plugins/issues/973 (to close)
    - https://github.com/containernetworking/cni/pull/1038 (simple fix)
    - https://github.com/containernetworking/cni/pull/1039 (supersedes PR#1035)
    - https://github.com/containernetworking/plugins/pull/974 (Add v6 disc_notify in macvlan)
    - https://github.com/containernetworking/plugins/pull/979 (Add v6 disc_notify in ipvlan)
    - https://github.com/containernetworking/plugins/pull/969 (required for future CNI vendor update)
    - https://github.com/containernetworking/plugins/pull/962 (remove unused code)
    - https://github.com/containernetworking/plugins/pull/921 (support exclude subnets in bandwidth plugin)
    - https://github.com/containernetworking/cni.dev/pull/130 (bandwidth plugin doc change)
- [cdc] STATUS implementation: https://github.com/containernetworking/cni/pull/1030
    - and SPEC: https://github.com/containernetworking/cni/pull/1003


## 2023-11-06
- Do some PR reviews (it's kubecon week)

## 2023-10-30
- [Tomo] PR review: CNI repo change to omit DNS in CNI Conf and Result (1.0.0 only)
    - Currently DNS field is 'omitempty' but not pointer, hence empty structure is returned
    - https://github.com/containernetworking/cni/pull/1035
    - Supersedes https://github.com/containernetworking/cni/pull/1007 (that changes to DNS Conf side)
- [PeterW] Reference multi-network design doc: https://docs.google.com/document/d/1oVOzlX4nDMyQM6VWJzqMO02FJDYaPb-FgD_88LkXjkU/edit#heading=h.m758iblg0in4

## 2023-10-23
- [Tomo] (TODO)Writing DNS Doc...
    - Discovery: DNS type is not currently called out as optional, it should be
    - 
- [Ed] Request from kubevirt community
    1. https://github.com/containernetworking/plugins/issues/951 (activateInterface option for bridge CNI plugin)
    2. support non-interface specific sysctl params in `tuning` (with as stand-alone, not meta plugin)
    - note: tuning currently allows anything in /proc/sys/net
- [Casey] Status PRs are ready for review
    - https://github.com/containernetworking/cni/pull/1003
    - https://github.com/containernetworking/cni/pull/1030


## 2023-10-16
NOTE: jitsi died, https://meet.google.com/hpm-ifun-ujm
- [PeterW] Multi-network update
    - KEP-EP heading towards Alpha
    - 
- GC is merged!
- 

## 2023-10-09
- We get distracted talking about multi-network and DNS responses
    - AI: Tomo will create doc (problem statement)
- [Tomo] https://github.com/containernetworking/plugins/issues/951
    - should we support that? 
    - config name / semantic both are weird...
    - Todo: Ask them about that in next 
- [Tomo] Request from kubevirt community
    - support non-interface specific sysctl params in `tuning` (with as stand-alone, not meta plugin)
    - Todo: Ask them about that in next 
    - note: tuning currently allows anything in /proc/sys/net
- [cdc] re-review GC (https://github.com/containernetworking/cni/pull/1022)

## 2023-10-02
- Aojea has questions about ContainerD sandbox API
    - Shouldn't affect CNI; just uses Sandbox api instead of runc / OCI
- https://github.com/containernetworking/cni/pull/1022 : GC PR
- Review outstanding TODOs for v1.1
    - https://github.com/containernetworking/cni/milestone/9
    - Looking good. We push INIT to v1.2
- PR: https://github.com/containernetworking/cni/pull/1024

## 2023-09-25
- Attendance: Doug, Michael Cambria, Antonio, Dan Williams , Dan Winship 
- Tomo: maintainer
    - If not discussed this meeting Tomo will open a PR for github discussion
    - resolved: files PR to add to maintainer list
    - https://github.com/containernetworking/cni/pull/1024
- Doug: High level question, what do you all think about K8s native multi-networking?
    - Giving a talk with Maciej Skrocki (Google) at Kubecon NA on K8s native multi-networking
    - I want to address "what's the position from a CNI viewpoint?"
    - My point is: We kinda of "ignore CNI as an implementation detail"
        - But! ...It's an important ecosystem.
    - Multus is kind of a "kubernetes enabled CNI runtime" -- or at least, users treat it that way
        - Should it continue to function in that role?
        - Should CNI evolve to have the "kubernetes enabled" functionality?
        - What do you all think?
    - CNI has always supported multinetworking (especially: rkt)
        - And K8s has taken almost 10 years!
        - Mike brings up, that it's really that the runtime insisted on doing only one interface.
        - Doug asks is it the runtime should be enabled with the functionality
            - What about dynamic reconfiguration
        - Mike Z mentions he's working on the CRI side, executing multiple cni
            - Pod sandbox status, to relay multiple IPs back, and the network names
            - Node network interface(NRI) doesn't have a network domain
            - Network domain hooks pre-and-post 
            - This is happening outside of the k8s space.
            - Use cases outside of Kubernetes, as well.
            - Custom schedulers for BGP, OSPF, etc.
        - Mike C brings up consideration of scheduling a pod with knowledge of which networks will be available.
    - Re: STATUS
        - Progrmatically distributing network configuration, and 
            - that problem appears in single network as well, and has relation to STATUS
    - Antonio brings up, what percent of community benefits from multiple interfaces?
    - CNI has been surprisingly static in the face of other changes (e.g. kubenet -> [...])
- Doug: Also any updates in Kubecon NA maintainer's summit?
    - no maintainers going :-(
- Back to STATUS?
    - Sticking point: how do you know whether or not to rely on STATUS -- as a plugin that automatically writes a configuration file
    - Idea: version negotiation when cniVersion is empty
        - This works if CRI-O / containerd ignore conflist files w/o a cniVersion
        - Casey to experiment
    - Still doesn't solve the problem of plugins knowing which value to use
- How do we know that a node supports v1.1 (and thus uses STATUS)?
    - sweet, the ContainerD / CRI-O version is exported in the Node object
    - Ugly, but heuristics will work
- draft GC PR: https://github.com/containernetworking/cni/pull/1022
    - of interest: deprecate PluginMain(add, del, check) b/c signature changes stink
- 

## 2023-09-18
- Attendance: Tomo, Antonio, Henry Wang, Dan Williams, Dan Winship
- Network Ready
    - Now: have a CNI config file on-disk
    - Container Runtimes: containerd and crio reply NetworkReady through CRI based on the existing of that file
    - When CNI plugin can't add interfaces to new Pods, we want the node to be no-schedule TODO(aojea) if condition Network notReady = tainted
        - Only way to currently indicate this is the CNI config file on-disk
        - Can't really remove the config file to indicate readiness (though libcni does cache config for DEL)
    - One option: enforce STATUS in plugin by always writing out CNI config with CNIVersion that includes status
        - Runtimes that don't know STATUS won't parse your config and will ignore your plugin
        - Downside: you have to know the runtime supports STATUS
        - Downside: in OpenShift upgrades, old CRIO runs with new plugin until node reboot, this would break that. You'd have to have a window where the runtime supported STATUS but your plugin didn't use it yet. Then 2 OpenShift releases later you can flip to requiring STATUS.

## 2023-09-11
- KubeCon NA: maintainer's summit?
    - Usually we can book these (might be a bit too late though)
- PR Update:
    - https://github.com/containernetworking/plugins/pull/921
        - Tomo approved

## 2023-09-04 (Labour day in US/Canada)
- Attendance: Casey, Peter, Tomo
- Question for the US people: KubeCon maintainer's summit?
    - Definite topic for next week
- Tomo: maintainer
    - will discuss next week
- 

## 2023-08-28
- Attendance: Antonio, MikeZ, Tomo
- Discussion about NRI/multi-network design

## 2023-08-21
- Attendance: Antonio, Peter, Dan Winship, Tomo
- Mike Zappa is likely to write down KEP for kubernetes CRI/CNI/NRI to handle cases like multi network
- Current multinetwork approach for the KEP is focusing on API phases
- Follow STATUS PR: https://github.com/containernetworking/cni/pull/1003

## 2023-08-14
- CDC on vacation next two weeks
- Milestone review -- https://github.com/containernetworking/cni/milestone/9
    - we close out a few proposals that have been rejected
- Further disussion for CNI over CRI
    - NRI: node resource interface: a series of hooks
        - https://github.com/containerd/nri
    - It could make sense for networking to be integrated in NRI
    - NRI has no networking support / domain right now; could be conceivably expanded
    - Network Service for Containerd

## 2023-08-07

- Reviving version negotiation
    - Casey's proposal: a configlist without a version uses VERSION to pick the highest one
    - This means that administrators don't have to pick a version, which requires understanding too many disparate components
- Can we rely on IPs never being used?
    - Nope, you have to use the ContainerID
    - No good way around it
- CNCF graduation?
    - MikeZ reached out about security audit, need to add the "best practices" badge
- We merge the GC spec
    - woohoo!
- Casey looks at PR https://github.com/containernetworking/plugins/pull/936/files and is a bit surprised at how many bridge VLAN settings there are
    - Could we get some holistic documentation of these options?
    - 

## 2023-07-31

- Discussion w.r.t.: https://github.com/containernetworking/cni/issues/927
    - Background: how do we tell what version if config to install?
    - We talk about adding more information to the VERSION command; could do things like discovering capabilities
    - Dream about executing containers instead of binaries on disk
        - (wow, it's like a shitty PodSpec! But still very interesting)
    - Remove as much as possible from CNI configuration, make it easy for administrators
    - Hope that multi-networking will make it easier for admins to push out network changes
    - How does one feel about version autonegotiation
        - let's do it
- CNCF graduated project?
    - requirements: https://github.com/cncf/toc/blob/main/process/graduation_criteria.md#graduation-stage
    - no real opposition, just not high on the list
- nftables! (FYI) https://github.com/containernetworking/plugins/pull/935
    - We talk about whether it is safe to rely on IP addresses being cleaned up between DEL and ADD
    - libcni always deletes chained plugins last-to-first to avoid this very issue... except not quite
    - ~~Thus, it is potentially safe to delete map entries solely by IP address~~
- Is it safe to rely on IP addresses always being cleaned up?

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