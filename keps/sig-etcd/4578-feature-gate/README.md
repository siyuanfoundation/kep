# KEP-4578: Feature Gate in etcd

<!-- toc -->
- [Release Signoff Checklist](#release-signoff-checklist)
- [Summary](#summary)
- [Motivation](#motivation)
  - [Goals](#goals)
  - [Non-Goals](#non-goals)
- [Proposal](#proposal)
  - [User Stories (Optional)](#user-stories-optional)
    - [Story 1](#story-1)
    - [Story 2](#story-2)
    - [Story 3](#story-3)
    - [Story 4](#story-4)
  - [Notes/Constraints/Caveats (Optional)](#notesconstraintscaveats-optional)
    - [Is feature enablement state a server level or cluster level property?](#is-feature-enablement-state-a-server-level-or-cluster-level-property)
    - [Should we use feature gate for bug fixes?](#should-we-use-feature-gate-for-bug-fixes)
    - [Could the lifecycle of a feature change in patch versions?](#could-the-lifecycle-of-a-feature-change-in-patch-versions)
  - [Risks and Mitigations](#risks-and-mitigations)
- [Design Details](#design-details)
  - [New APIs](#new-apis)
  - [Feature Gate](#feature-gate)
  - [Cluster Level Feature Enablement](#cluster-level-feature-enablement)
  - [Feature Stages](#feature-stages)
  - [Test Plan](#test-plan)
      - [Prerequisite testing updates](#prerequisite-testing-updates)
      - [Unit tests](#unit-tests)
      - [Integration tests](#integration-tests)
      - [e2e tests](#e2e-tests)
  - [Graduation Criteria](#graduation-criteria)
    - [Milestone 1](#milestone-1)
    - [Milestone 2](#milestone-2)
    - [Milestone 3](#milestone-3)
  - [Upgrade / Downgrade Strategy](#upgrade--downgrade-strategy)
  - [Version Skew Strategy](#version-skew-strategy)
- [Implementation History](#implementation-history)
- [Drawbacks](#drawbacks)
- [Alternatives](#alternatives)
<!-- /toc -->

## Release Signoff Checklist

<!--
**ACTION REQUIRED:** In order to merge code into a release, there must be an
issue in [kubernetes/enhancements] referencing this KEP and targeting a release
milestone **before the [Enhancement Freeze](https://git.k8s.io/sig-release/releases)
of the targeted release**.

For enhancements that make changes to code or processes/procedures in core
Kubernetes—i.e., [kubernetes/kubernetes], we require the following Release
Signoff checklist to be completed.

Check these off as they are completed for the Release Team to track. These
checklist items _must_ be updated for the enhancement to be released.
-->

Items marked with (R) are required *prior to targeting to a milestone / release*.

- [ ] (R) Enhancement issue in release milestone, which links to KEP dir in [kubernetes/enhancements] (not the initial KEP PR)
- [ ] (R) KEP approvers have approved the KEP status as `implementable`
- [ ] (R) Design details are appropriately documented
- [ ] (R) Test plan is in place
- [ ] (R) Graduation criteria is in place
- [ ] "Implementation History" section is up-to-date for milestone
- [ ] User-facing documentation has been created in [etcd-io/website], for publication to [etcd.io]
- [ ] Supporting documentation—e.g., additional design documents, links to mailing list discussions/SIG meetings, relevant PRs/issues, release notes

<!--
**Note:** This checklist is iterative and should be reviewed and updated every time this enhancement is being considered for a milestone.
-->

[etcd.io]: https://etcd.io/
[kubernetes/enhancements]: https://git.k8s.io/enhancements
[kubernetes/kubernetes]: https://git.k8s.io/kubernetes
[etcd-io/website]: https://github.com/etcd-io/website

## Summary

We are introducing a new `--feature-gates` flag in etcd, and the underlying framework to gate future feature enhancement to etcd. 

## Motivation

Currently any new enhancements to the etcd are typically added as [an experimental feature](https://github.com/etcd-io/etcd/blob/main/Documentation/contributor-guide/features.md#adding-a-new-feature), with a configuration flag prefixed with "experimental", e.g. --experimental-feature-name. 

When it is time to [graduate an experimental feature to stable](https://github.com/etcd-io/etcd/blob/main/Documentation/contributor-guide/features.md#graduating-an-experimental-feature-to-stable), a new stable feature flag identical to the experimental feature flag but without the --experimental prefix is added to replace the old feature flag, which is a breaking change.

We are proposing to add [feature gates](https://kubernetes.io/docs/reference/command-line-tools-reference/feature-gates/) in the similar way as Kubernetes does, so that can turn on or off any feature using the `--feature-gates` flag.

Benefits from this enhancement includes:

* graduating a feature is no longer a breaking change with new flags.

* well-established feature lifecycle convention from Kubernetes.

* a unified place to store all the feature enablement state in the code base for simplicity, and easy to refer to in guarding the implementation code.

### Goals

* Introduce the `--feature-gates` flag in etcd code base.

* Introduce apis/methods to query the enablement state of a feature.

* Establish lifecycle stages of a feature, and clear criteria of lifecycle progression.

### Non-Goals

* Migrate all existing `--experimental` features to feature gate.

* Use feature gate as a mechanism to fix bugs.

## Proposal

### User Stories (Optional)

#### Story 1

It would be easier for developers to add new features without worrying too much about breaking etcd, because the feature would be Alpha first and disabled by default, and new changes would be mostly gated behind the feature gate.

It would be a smoother process to move the feature to the next stage without the need to introduce a potentially breaking change of an `--experimental` flag.

The developer also would not need to worry about feature compatibility when there is a mixed version cluster.

#### Story 2

Users of etcd cluster would be able to find out which features are enabled in the etcd cluster and decide on how to use them downstream.

#### Story 3

In a HA cluster, users should be able to enable/disable a feature without bring down the whole cluster, by restarting the servers with the new feature gate flag one by one. 

#### Story 4

During cluster upgrade/downgrade, feature changes across versions should have predictable behavior in this mix version scenario. 

### Notes/Constraints/Caveats (Optional)

#### Is feature enablement state a server level or cluster level property?

There are features like `ExperimentalEnableLeaseCheckpoint` (enables leader to send regular checkpoints to other members to prevent reset of remaining TTL on leader change), if different server nodes have different enablement values for `ExperimentalEnableLeaseCheckpoint`, the results would be inconsistent depending on which one is the leader and confusing.

There are also features like `ExperimentalEnableDistributedTracing` (enables distributed tracing using OpenTelemetry protocol), which could work fine at the local server level.

We are proposing to have separate APIs to query if a feature is enabled for the local server, and for the whole cluster.

#### Should we use feature gate for bug fixes?

There are some use cases like `ProgressNotify`, for which some bugs are found and later fixed in a patch version. The client would need to know if the etcd version in use contains that fix to decide whether or not to use that feature. 

The question is: should a new feature gate be added to signal the bug fix? 

We think the answer is NO:
* the new feature would need to be enabled by default to always apply the bug fix for new releases.
* it changes the API which is not desirable in patch version releases.

The proper way of handling cases like `ProgressNotify` is:
1. the feature should be gated by the feature gate from the beginning.
1. the feature should be disabled by default until it is widely tested in practice. 
1. when the bug is found, the feature should ideally be at a lifecycle in which it is disabled by default. If not, the admin should disable it by the `--feature-gates` flag.
1. when the client upgrades etcd to the patch version with the fix, the admin could enable it by the `--feature-gates` flag.

#### Could the lifecycle of a feature change in patch versions?

Kubernetes have a minor release every 3 months, while the cadence of etcd minor releases is much less frequent. The question is: do we have to wait for years before graduating a new feature?

We think we should still stick to the common practice of not changing the lifecycle of a feature in patch versions. Because:
* changing the lifecycle of a feature is an API change. According to the [etcd Operations Guide](https://etcd.io/docs/v3.5/op-guide/versioning/), only new minor versions may add additional features to the API.
* bugs in etcd could be hard to detect, and the reliability and robustness of etcd is more important than speed. A long history of testing through practical adoption a new feature is beneficial.

With the feature gate in place, we could consider increasing the etcd release cadence because it would be easier to add new features and less risky to release new features.

### Risks and Mitigations

<!--
What are the risks of this proposal, and how do we mitigate? Think broadly.
For example, consider both security and how this will impact the larger
Kubernetes ecosystem.

How will security be reviewed, and by whom?

How will UX be reviewed, and by whom?

Consider including folks who also work outside the SIG or subproject.
-->

## Design Details

### New APIs

A new `--feature-gates` command line argument would be added to start the etcd server, with format like `--feature-gates=featureA=true,featureB=false`. The flag can also be set in the `config-file`.

New grpc and http APIs would be added to query if a feature is enabled for the server or cluster. New `etcdctl` command will also be added.

For grpc, new rpc type would be added to the `Maintenance` service.

```proto
service Maintenance {
  ...
  rpc FeatureGateStatus(FeatureGateStatusRequest) returns (FeatureGateStatusResponse) {
    option (google.api.http) = {
      post: "/v3/maintenance/featuregate"
      body: "*"
    };
  }
  ...
}
message Feature {
  string name = 1;
  bool enabled = 2;
}

message FeatureGateStatusRequest {
  // if true, query if the features are enabled for the single server.
  // otherwise, query if the features are enabled for the cluster.
  bool isServerFeature = 1;
  repeated string features = 2; // return all the enabled features if empty.
}

message FeatureGateStatusResponse {
  ResponseHeader header = 1;
  bool isServerFeature = 2;
  repeated Feature features = 3;
}
```

The http endpoints could look like:
* `{server_endpoint}/featuregate?feature={featureName}` returns true if the feature is enabled for the cluster
* `{server_endpoint}/featuregate?feature={featureName}&isServerFeature=true` returns true if the feature is enabled for the server

The `etcdctl` commands could look like:
* `etcdctl endpoint featuregate $featureName` returns true if the feature is enabled for the cluster
* `etcdctl endpoint featuregate $featureName --is-server-feature` returns true if the feature is enabled for the server

### Feature Gate

We will use the new `k8s.io/component-base/featuregate.VersionedSpecs`(introduced in [kep-4330](https://github.com/kubernetes/enhancements/tree/master/keps/sig-architecture/4330-compatibility-versions)) to register and track the features along with their lifecycle at different release versions. 

```go
map[Feature]VersionedSpecs {
		featureA: VersionedSpecs{
			{Version: mustParseVersion("1.27"), Default: false, PreRelease: Beta},
			{Version: mustParseVersion("1.28"), Default: true, PreRelease: GA},
		},
		featureB: VersionedSpecs{
			{Version: mustParseVersion("1.28"), Default: false, PreRelease: Alpha},
		},
		featureC: VersionedSpecs{
			{Version: mustParseVersion("1.28"), Default: false, PreRelease: Beta},
		},
		featureD: VersionedSpecs{
			{Version: mustParseVersion("1.26"), Default: false, PreRelease: Alpha},
			{Version: mustParseVersion("1.28"), Default: true, PreRelease: Deprecated},
		}
}
```

The feature gates for a server can only be set with the `--feature-gates`flag or in the `config-file` during startup. We do not support dynamically changing the feature gates when the server is running.

The `ServerConfig` struct will have a new `featureGate k8s.io/component-base/featuregate.FeatureGate`(immutable) field, which contains the interface of `Enabled(key Feature) bool`, and it would be piped through to where the feature gate is needed.

```go
type ServerConfig struct {
  ...
  FeatureGate featuregate.FeatureGate
  ...
}
```

(global singleton is convenient but it would be hard to track where it is mutated by different component flags (etcd server or proxy server), or in different tests.)

### Cluster Level Feature Enablement

To determine if a feature is enabled in the whole cluster, each member would need to know if the feature is enabled for all cluster members. We plan to store the information in the [`member.Attributes`](https://github.com/etcd-io/etcd/blob/e37a67e40b3f5ff8ef81f9de6e7f475f17fda32b/server/etcdserver/api/membership/member.go#L38), and it would be saved in the `members` bucket in the backend through raft.

```proto
message Attributes {
  option (versionpb.etcd_version_msg) = "3.5";

  string name = 1;
  repeated string client_urls = 2;
  repeated string enabled_features = 3;
}
```
When an etcd server starts, the attributes would be [published through raft](https://github.com/etcd-io/etcd/blob/e37a67e40b3f5ff8ef81f9de6e7f475f17fda32b/server/etcdserver/server.go#L1745) and stored in all members. Whenever a new member joins or an existing member restarts, its feature gate attributes would be automatically updated in the start up process.

We choose to store the `enabled_features` through raft instead of dynamically querying the `\featuregate` endpoint because
1. it saves network bandwidth.
1. we want the cluster feature enablement status to be as stable as possible, only subject to change when the admin intends it as in the case of adding/removing/upgrading/downgrading/restarting servers. In the case of unpredictable loss of a member server, the feature gate of the cluster should not change.

In `membership.RaftCluster`, we will add a new an aggregate method `FeatureEnabled(key Feature) bool` to query cluster level enablement of a feature. 

```go
// A feature is enabled for the whole cluster only if it is enabled for all the members. 
// In case of a conflict, the feature would be disabled, and would not bring down the whole cluster because conflict is expected during upgrade/downgrade.
func (*RaftCluster) FeatureEnabled(key Feature) bool {}
```

### Feature Stages

Following the convention of Kubernetes, a feature can go through a lifecycle of Alpha → Beta → GA → Deprecated.

| Feature Stage | Properties | Graduation Criteria |
| --- | --- | --- |
| Alpha | <ul><li>Disabled by default. </li><li>Might be buggy. Enabling the feature may expose bugs. </li><li>Support for feature may be dropped at any time without notice. </li><li>The API may change in incompatible ways in a later software release without notice. </li><li>Recommended for use only in short-lived testing clusters, due to increased risk of bugs and lack of long-term support.</li></ul> | Before moving a feature to Beta, it should have <ul><li> Full unit/integration/e2e/robustness test coverage.</li><li>Full performance benchmark/test if applicable.</li><li> No significant changes for at least 1 minor release.</li><li>Other feature specific criteria in the feature KEP. </li></ul> |
| Beta | <ul><li>Enabled by default. </li><li>The feature is well tested. Enabling the feature is considered safe.</li><li>Support for the overall feature will not be dropped, though details may change.</li><li>Recommended for only non-business-critical uses because of potential for discovering new hard-to-spot bugs through wider adoption.</li></ul> | Before moving a feature to GA, it should have <ul><li> Widespread usage.</li><li>No bug reported for at least 1 minor release.</li><li>Other feature specific criteria in the feature KEP. </li></ul> |
| GA | <ul><li>The feature is always enabled; you cannot disable it.</li><li>The corresponding feature gate is no longer needed.</li><li>Stable versions of features will appear in released software for many subsequent versions.</li></ul> | Before deprecating a GA feature, it should have <ul><li> Feature deprecation announcement.</li><li>No user impacting change for at least 1 minor release.</li><li>Other feature specific criteria in the feature KEP. </li></ul> |
| Deprecated | <ul><li>The feature gate is no longer in use. </li><li>The feature has graduated to GA or been removed.</li></ul> | <ul><li>If deprecating Beta feature, should set default to disabled first for at least 1 minor release.</li></ul> |
### Test Plan

[x] I/we understand the owners of the involved components may require updates to
existing tests to make this code solid enough prior to committing the changes necessary
to implement this enhancement.

##### Prerequisite testing updates

##### Unit tests

New feature gate unit tests will be added.

##### Integration tests

New feature gate integration tests will be added.

##### e2e tests

We will add one cluster level and one server level experimental features to the feature gate (without removing the original experimental flag), and add e2e tests to make sure the feature gate functions equivalently to their `--experimental-xxx` flags.

We will also add downgrade/upgrade e2e tests to make sure the feature gate does not break the cluster in mixed version cluster.

### Graduation Criteria

#### Milestone 1

* server level feature gate implemented.
* cluster level feature gate implemented.

#### Milestone 2

* backport necessary proto changes to 3.5.
* http endpoints added.
* grpc endpoints added.
* `etcdctl` command added.
* equivalent of one cluster level and one server level experimental features added to the feature gate.
* initial e2e tests completed.

#### Milestone 3

* e2e tests for upgrade & downgrade & mixed version scenarios completed.
* robustness tests completed. 
* documentation.

### Upgrade / Downgrade Strategy

The feature gate feature would available in 3.6+. 
In 3.5, the necessary proto changes would be backported, but feature gate would not be available. 
When upgrading/downgrading between 3.5 and 3.6, no feature would be enabled if there is at least one member with version 3.5.

If a user uses a 3.5 version that does not recognize the new proto field, it is recommended they should upgrade to the latest 3.5 release first before upgrading to 3.6. Since unknown fields is ignored by proto, etcd should still work even if they do not upgrade to latest 3.5 version.

### Version Skew Strategy

Since we are introducing new fields in `member.Attributes` which is sent through raft, we will backport the new fields into 3.5, but would not use them in any way affecting 3.5 etcd server.

## Implementation History

<!--
Major milestones in the lifecycle of a KEP should be tracked in this section.
Major milestones might include:
- the `Summary` and `Motivation` sections being merged, signaling SIG acceptance
- the `Proposal` section being merged, signaling agreement on a proposed design
- the date implementation started
- the first Kubernetes release where an initial version of the KEP was available
- the version of Kubernetes where the KEP graduated to general availability
- when the KEP was retired or superseded
-->

## Drawbacks

<!--
Why should this KEP _not_ be implemented?
-->

## Alternatives

<!--
What other approaches did you consider, and why did you rule them out? These do
not need to be as detailed as the proposal, but should include enough
information to express the idea and why it was not acceptable.
-->
