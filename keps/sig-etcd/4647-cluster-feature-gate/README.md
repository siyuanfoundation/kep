# KEP-4647: Cluster Feature Gate in etcd

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
    - [How to Reach Consensus of Feature Gates](#how-to-reach-consensus-of-feature-gates)
  - [Feature Stages](#feature-stages)
  - [Path to Migrate Existing Experimental Features](#path-to-migrate-existing-experimental-features)
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

We are introducing the Kubernetes style feature gate framework into etcd to gate future feature enhancement behind a sequence of feature lifecyles. Users would be able to turn features on or off and query feature enablement in a consistent way. 

## Motivation

Currently any new enhancements to the etcd are typically added as [an experimental feature](https://github.com/etcd-io/etcd/blob/main/Documentation/contributor-guide/features.md#adding-a-new-feature), with a configuration flag prefixed with "experimental", e.g. --experimental-feature-name. 

When it is time to [graduate an experimental feature to stable](https://github.com/etcd-io/etcd/blob/main/Documentation/contributor-guide/features.md#graduating-an-experimental-feature-to-stable), a new stable feature flag identical to the experimental feature flag but without the --experimental prefix is added to replace the old feature flag, which is a breaking change.

We are proposing to add [feature gates](https://kubernetes.io/docs/reference/command-line-tools-reference/feature-gates/) in the similar way as Kubernetes does, so that can turn on or off any feature using the `--feature-gates` flag.

Benefits from this enhancement includes:

* graduating a feature is no longer a breaking change with new flags.

* well-established feature lifecycle convention from Kubernetes.

* a unified place to store all the feature enablement state in the code base for simplicity, and easy to refer to in guarding the implementation code.

* easy to query if a feature is available for an ectd cluster.

### Goals

* Introduce the `--feature-gates` flag in etcd code base.

* Introduce apis/methods to query the enablement state of a feature.

* Establish lifecycle stages of a feature, and clear criteria of lifecycle progression.

### Non-Goals

* Migrate all existing `--experimental` features to feature gate. 
  * While feature gate provides a [path to deprecate existing `--experimental` features]((#path-to-migrate-existing-experimental-features)), and we will migrate several `--experimental` during the feature gate implementation as examples, the completion of the KEP does not require migrating all existing `--experimental` features. It can be done as followup tasks.

* Use feature gate as a mechanism to fix bugs.

## Proposal

### User Stories (Optional)

#### Story 1

It would be easier for developers to add new features without worrying too much about breaking etcd, because the feature would be Alpha first and disabled by default, and new changes would be mostly gated behind the feature gate.

It would be a smoother process to move the feature to the next stage without the need to introduce a potentially breaking change of an `--experimental` flag.

The developer also would not need to worry about feature compatibility when there is a mixed version cluster.

#### Story 2

Users of etcd cluster would be able to find out which features are enabled in the etcd cluster and decide on how to use them downstream. 

Currently Kubernetes uses [`FeatureSupportChecker`](https://github.com/kubernetes/kubernetes/blob/db82fd1604ebf327ab74cde0a7158a8d95d46202/staging/src/k8s.io/apiserver/pkg/storage/etcd3/feature/feature_support_checker.go#L42) to check if a feature is supported by etcd, which basically checks the etcd version and compares the version with a hard-coded map of feature availability for different etcd versions.

#### Story 3

In a HA cluster, users should be able to enable/disable a feature without bring down the whole cluster, by restarting the servers with the new feature gate flag one by one. 

#### Story 4

During cluster upgrade/downgrade, feature changes across versions should have predictable behavior as allowed by the cluster version. 

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

* To support feature gate as soon as in 3.6, we are backporting some proto changes into 3.5. There is the potential risk of changing a stable release. But we do not think this is a real risk, because it does not involve any changes other than adding a new proto field in an existing proto, and 3.5 server does not write or use this new field. Protos are inherently backward compatible. This change should also not affect upgrade/downgrade of 3.5.

## Design Details

### New APIs

A new `--feature-gates` command line argument would be added to start the etcd server, with format like `--feature-gates=featureA=true,featureB=false`. The flag can also be set in the `config-file`.

New Prometheus gauge metrics would be added to monitor and query if a feature is enabled for the server or cluster. 
For easier usage in Kubernetes, we will also add a grpc backed `clientv3` function and a new `etcdctl` command.

The metrics could look like:
* `etcd_feature_enabled{name="featureName",stage="Alpha"} 1` if the feature is enabled.

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
  repeated string features = 2;
}
message FeatureGateStatusResponse {
  ResponseHeader header = 1;
  repeated Feature features = 3;
}
```

The clientv3 function would look like:

```go
package clientv3

type Maintenance interface {
  FeatureGateStatus(ctx context.Context, features ...string) (*FeatureGateStatusResponse, error)
}
```

The `etcdctl` commands could look like:
* `etcdctl endpoint featuregate $featureName` returns true if the feature is enabled

For all the apis, 
* if the feature is a sever level feature, the result would be if the feature is enabled for that server.
* if the feature is a cluster level feature, the result would be if the feature is enabled for the whole cluster.

### Feature Gate

We will use the new `k8s.io/component-base/featuregate.VersionedSpecs`(introduced in [kep-4330](https://github.com/kubernetes/enhancements/tree/master/keps/sig-architecture/4330-compatibility-versions)) to register and track the features along with their lifecycle at different release versions. 

A feature can be registered as server level feature or cluster level feature, but not both.

```go
defaultServerFeatureGates := map[Feature]VersionedSpecs {
		featureA: VersionedSpecs{
			{Version: mustParseVersion("3.6"), Default: false, PreRelease: Beta},
			{Version: mustParseVersion("3.7"), Default: true, PreRelease: GA},
		},
		featureB: VersionedSpecs{
			{Version: mustParseVersion("3.7"), Default: false, PreRelease: Alpha},
		},
}
defaultClusterFeatureGates := map[Feature]VersionedSpecs {
		featureC: VersionedSpecs{
			{Version: mustParseVersion("3.7"), Default: false, PreRelease: Beta},
		},
		featureD: VersionedSpecs{
			{Version: mustParseVersion("3.6"), Default: false, PreRelease: Alpha},
			{Version: mustParseVersion("3.7"), Default: true, PreRelease: Deprecated},
		}
}
```

The feature gates for a server can only be set with the `--feature-gates`flag or in the `config-file` during startup. We do not support dynamically changing the feature gates when the server is running.

The `ServerConfig` struct will have a new `featureGate k8s.io/component-base/featuregate.FeatureGate`(immutable) field. `EtcdServer` would have an interface of `FeatureEnabled(key Feature) bool`, and it can be referenced throughout the server.

```go
type ServerConfig struct {
  ...
  // ServerFeatureGate server level feature gate
  ServerFeatureGate featuregate.FeatureGate
  ...
}

func (s *EtcdServer) FeatureEnabled(key Feature) bool
```

### Cluster Level Feature Enablement

To determine if a feature is enabled in the whole cluster, each member would need to know if the feature is enabled for all cluster members. We plan to store the information in the [`member.Attributes`](https://github.com/etcd-io/etcd/blob/e37a67e40b3f5ff8ef81f9de6e7f475f17fda32b/server/etcdserver/api/membership/member.go#L38), and it would be saved in the `members` bucket in the backend through raft.

```proto
message Attributes {
  option (versionpb.etcd_version_msg) = "3.5";

  string name = 1;
  repeated string client_urls = 2;
  // the values of all cluster level feature gates as seen by the version of the member server.
  string feature_gates = 3;
}
```
When an etcd server starts, its `Attributes.feature_gates` value will be determined by the `--feature-gates` flag and the local server version. The attributes would then be [published through raft](https://github.com/etcd-io/etcd/blob/e37a67e40b3f5ff8ef81f9de6e7f475f17fda32b/server/etcdserver/server.go#L1745) and stored in all members. Whenever a new member joins or an existing member restarts, its feature gate attributes would be automatically updated in the start up process.

We choose to store the `feature_gates` through raft instead of dynamically querying the `\metrics` endpoint because
1. it saves network bandwidth.
1. all members need to reach a linearizable consensus for the feature gates.
1. adding a new field to existing member attribute proto is probably less risky than adding a new loop to monitor the cluster features.

In `membership.RaftCluster`, we will add a new cluster level feature gate to store the cluster level feature enablement. The `EtcdServer.FeatureEnabled` method will call the cluster level feature gate if the feature is registered as a cluster level feature.

```go
type RaftCluster struct {
  ...
  clusterFeatureGate featuregate.FeatureGate
  ...
}

func (c *RaftCluster) FeatureEnabled(key Feature) bool
```

#### How to Reach Consensus of Feature Gates

We will leverage the consensus `ClusterVersion` to negotiate the cluster level feature gate values:
1. if `ClusterVersion` is not set, the cluster feature gate would be `nil`, and all features would be considered disabled. 
1. when the `ClusterVersion` is set or updated, initialize the cluster feature gate with the `ClusterVersion`. At this point,
    * if the `member.Attributes` of any members has not been set, use the default values for feature gate at version `ClusterVersion`.
    * if the `member.Attributes` of all members have been set, take the common set of all the mbmer `--feature-gates` flags and set values of the cluster feature gate: discard any features not recognized at the `ClusterVersion`, and set the feature to false if there is any member setting it to false.
    * learner nodes would not be taken into consideration because they do not serve requests.
1. if any `member.Attributes` is updated, reinitialize the cluster feature gate with the `ClusterVersion`, and repeat step 2.

One drawback of this approach is that there is a period before the `ClusterVersion` is set when no feature would be available. We think that is acceptable because the `ClusterVersion` should be set soon after leader is elected, so that period should very short, and the no-feature-available stage is also consistent throughout the cluster.
* an alternative would be set some default min version similar to `MinClusterVersion = "3.0.0"`. The problem is: if we want to bump `MinClusterVersion` to `4.0.0` at release 4.1.0, we could potentially break the feature gate consistency when upgrading from/downgrading to the previous 3.x release, or we would complicate the upgrade/downgrade process. 

### Feature Stages

Following the convention of Kubernetes, a feature can go through a lifecycle of Alpha → Beta → GA → Deprecated.

| Feature Stage | Properties | Graduation Criteria |
| --- | --- | --- |
| Alpha | <ul><li>Disabled by default. </li><li>Might be buggy. Enabling the feature may expose bugs. </li><li>Support for feature may be dropped at any time without notice. </li><li>The API may change in incompatible ways in a later software release without notice. </li><li>Recommended for use only in short-lived testing clusters, due to increased risk of bugs and lack of long-term support.</li></ul> | Before moving a feature to Beta, it should have <ul><li> Full unit/integration/e2e/robustness test coverage.</li><li>Full performance benchmark/test if applicable.</li><li> No significant changes for at least 1 minor release.</li><li>Other feature specific criteria in the feature KEP. </li></ul> |
| Beta | <ul><li>Enabled by default. </li><li>The feature is well tested. Enabling the feature is considered safe.</li><li>Support for the overall feature will not be dropped, though details may change.</li><li>Recommended for only non-business-critical uses because of potential for discovering new hard-to-spot bugs through wider adoption.</li></ul> | Before moving a feature to GA, it should have <ul><li> Widespread usage.</li><li>No bug reported for at least 1 minor release.</li><li>Other feature specific criteria in the feature KEP. </li></ul> |
| GA | <ul><li>The feature is always enabled; you cannot disable it.</li><li>The corresponding feature gate is no longer needed.</li><li>Stable versions of features will appear in released software for many subsequent versions.</li></ul> | Before deprecating a GA feature, it should have <ul><li> Feature deprecation announcement.</li><li>No user impacting change for at least 1 minor release.</li><li>Other feature specific criteria in the feature KEP. </li></ul> |
| Deprecated | <ul><li>The feature gate is no longer in use. </li><li>The feature has graduated to GA or been removed.</li></ul> | <ul><li>If deprecating Beta feature, should set default to disabled first for at least 1 minor release.</li></ul> |

### Path to Migrate Existing Experimental Features

Feature gate is the replacement of future experimental features. We can also use it to establish a path to deprecate existing experimental features. 

To safely migrate an existing experimental feature `--experimental-feature-a`, we need to go through several stages:
1. the `--experimental-feature-a` and `--feature-gates=FeatureA=true|false` flags coexist for at least 1 minor release. Announce upcoming deprecation of the old flag.
    1. create a new feature `FeatureA` in feature gate. The lifecyle stage of `FeatureA` would be Alpha if the old flag is disabled by default, and Beta if it is enabled by default.
    1. both the `--experimental-feature-a` and `--feature-gates=FeatureA=true|false` can be used to set the enablement of `FeatureA`, and add checks to make sure the two flags are not both set at start-up. 
    1. all references to the `experimental-feature-a` value in the code would be replaced by `featureGate.Enabled(FeatureA)`
    1. print warning messages of flag deprecation in the help, and when the old flag is used.
1. depreate the old `--experimental-feature-a` flag in the next minor release. Keep the lifecycle of `FeatureA` the unchanged for at least 1 minor release.
1. normal lifecycle progression of `FeatureA` from now on.

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
  * `--feature-gates` flag added.
  * feature gate added to the server code, and used by a server level experimental feature.
  * feature metrics added.
  * grpc endpoints added.
  * `clientv3` function added.
  * `etcdctl` command added.
  * e2e tests added for the feature gate equivalent of the selected experimental feature.

#### Milestone 2

* cluster level feature gate implemented.
  * new raft changes added.
  * feature gate added to the server code, and used by a cluster level experimental feature.
  * feature metrics added.
  * e2e tests added for the feature gate equivalent of the selected experimental feature.
  * robustness tests added for current version.

#### Milestone 3

* backport necessary miminal changes to 3.5.
* e2e tests for upgrade & downgrade & mixed version scenarios completed.
* robustness tests with upgrade & downgrade & mixed version scenarios completed. 
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
