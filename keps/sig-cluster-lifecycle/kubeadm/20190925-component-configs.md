---
title: kubeadm component config management
authors:
  - "@rosti"
owning-sig: sig-cluster-lifecycle
participating-sigs:
  - sig-cluster-lifecycle
reviewers:
  - "@fabriziopandini"
  - "@neolit123"
  - "@ereslibre"
  - "@yastij"
approvers:
  - "@timothysc"
editor: "@rosti"
creation-date: 2019-09-25
last-updated: 2020-04-29
status: implementable
see-also:
  - "/keps/sig-cluster-lifecycle/kubeadm/0023-kubeadm-config.md"
  - "/keps/sig-cluster-lifecycle/kubeadm/20190722-Advanced-configurations-with-kubeadm-(Kustomize).md"
---

# kubeadm component config management

## Table of Contents

<!-- toc -->
- [Release Signoff Checklist](#release-signoff-checklist)
- [Summary](#summary)
- [Motivation](#motivation)
  - [Goals](#goals)
  - [Non-Goals](#non-goals)
- [Proposal](#proposal)
  - [User Stories](#user-stories)
    - [Story 1](#story-1)
    - [Story 2](#story-2)
    - [Story 3](#story-3)
    - [Story 4](#story-4)
  - [Stop Component Config defaulting](#stop-component-config-defaulting)
  - [Delegate config validation to the components](#delegate-config-validation-to-the-components)
  - [User supplied vs kubeadm generated component configs](#user-supplied-vs-kubeadm-generated-component-configs)
  - [Kubernetes Core vs AddOns Component Configs](#kubernetes-core-vs-addons-component-configs)
  - [Stop using component config internal types](#stop-using-component-config-internal-types)
  - [“kubeadm upgrade plan” changes](#kubeadm-upgrade-plan-changes)
  - [&quot;kubeadm upgrade apply&quot; changes](#kubeadm-upgrade-apply-changes)
  - [Introduce the &quot;kubeadm alpha config view upgradeable&quot; command](#introduce-the-kubeadm-alpha-config-view-upgradeable-command)
  - [Introduce the &quot;kubeadm alpha config print component-defaults&quot; command](#introduce-the-kubeadm-alpha-config-print-component-defaults-command)
  - [Risks and Mitigations](#risks-and-mitigations)
    - [Users will have to manually migrate more component configs than before](#users-will-have-to-manually-migrate-more-component-configs-than-before)
    - [Users will loose track of what needs manual migration and what not](#users-will-loose-track-of-what-needs-manual-migration-and-what-not)
    - [Users won't be able to safely upgrade manually component configs](#users-wont-be-able-to-safely-upgrade-manually-component-configs)
- [Design Details](#design-details)
  - [Test Plan](#test-plan)
  - [Graduation Criteria](#graduation-criteria)
  - [Upgrade / Downgrade Strategy](#upgrade--downgrade-strategy)
  - [Version Skew Strategy](#version-skew-strategy)
- [Implementation History](#implementation-history)
  - [09/25/2019](#09252019)
  - [04/29/2020](#04292020)
<!-- /toc -->

## Release Signoff Checklist

- [ ] kubernetes/enhancements issue in release milestone, which links to KEP (this should be a link to the KEP location in kubernetes/enhancements, not the initial KEP PR)
- [X] KEP approvers have set the KEP status to `implementable`
- [X] Design details are appropriately documented
- [ ] Test plan is in place, giving consideration to SIG Architecture and SIG Testing input
- [ ] Graduation criteria is in place
- [ ] "Implementation History" section is up-to-date for milestone
- [ ] User-facing documentation has been created in [kubernetes/website], for publication to [kubernetes.io]
- [ ] Supporting documentation e.g., additional design documents, links to mailing list discussions/SIG meetings, relevant PRs/issues, release notes

## Summary

This document outlines a model by which kubeadm will manage component configs other than its own.

## Motivation

For a long time now, kubeadm has been using component configs for kubelet and kube-proxy. These configs are usually generated by kubeadm by using a default component config as a basis and adding setup specific details (either opinionated by kubeadm itself, or supplied via some of the kubeadm’s config objects).  
There is support for users to supply their own component configs along with the kubeadm specific config objects upon kubeadm init. Kubeadm validates, defaults and stores these config on the cluster in the form of config maps. It also supplies these configs in one way or another to the target component itself.  
There is also a desire and code has been historically structured in a way, to allow kubeadm to convert component config formats upon upgrade, should the need for that arise.

Hence, the following problems have been observed:
- Internal types have been used to allow for defaulting, validating and migrating component configs. This cannot continue if kubeadm is to move out of the kubernetes/kubernetes repository.
- Defaulting the configs, that are generated by kubeadm or supplied by users, populates additional fields with default values. This makes it difficult to determine in retrospect, what fields were set on purpose with their default values, and what fields were simply automatically filled in by the defaulting code.
- Providing support for validation by vendoring component specific code can lead to false-positive and missed errors. This can be caused by the deviation between the imported code by kubeadm and the actual component.
- Migrating between config versions also requires code to be vendored in or forked from the components. This can also lead to difficult to diagnose errors and changed behaviour in unexpected ways.
- Not distinguishing between kubeadm generated and user supplied component configs, necessitates the requirement for config migration. Kubeadm generated configs that are not modified by users directly in the config map, can be fully regenerated upon config format change. In all other cases kubeadm must convert the config to preserve user changes.

These observations along with the emergence of kubeadm phases, kustomize support and work around addon operators allows us to have the following future goals.

### Goals

- Stop vendoring internal types and validation, defaulting and conversion code.
- Stop defaulting component configs.
- Outsource validation to the component binaries themselves.
- Distinguish between user supplied and kubeadm generated and unmodified component configs.
- Don't force users to convert manually component configs that were generated by kubeadm.
- Use more community wide solutions, like Kustomize, kubeadm phases and addon operators.
- Provide for as clean as possible upgrade path for existing clusters.

### Non-Goals

- Change the handling of kubeadm’s own config types in any way.
- Cover in any way the migration of command line flags to component configs.
- Define strategies for config map naming and backup.
- Define ways to use advanced configurations with component configs

## Proposal

### User Stories

#### Story 1

As a cluster administrator, I don't want to manually migrate component configs that need upgrading, but were generated in their entirety by kubeadm.

#### Story 2

As a cluster administrator, I want to easily see what component configs need manual migration and to what version, prior to cluster upgrade.

#### Story 3

As a cluster administrator, I want an easy way of obtaining defaults for all component configs, supported by kubeadm, taking my ClusterConfiguration into consideration.

#### Story 4

As a cluster administrator, I want an easy and reliable way of specifying newer versions for component configs upon cluster upgrades.

### Stop Component Config defaulting

Component config defaulting has a number of issues:
- It’s difficult to tell in retrospect if a setting value was intended or simply defaulted.
- If a default value changes a defaulted setting will stop us from using the new value.
- Defaulting bloats the resulting config, thus making it hard for users to migrate it upon a version change.

Hence, kubeadm should stop defaulting component configs and leave this to components themselves.

### Delegate config validation to the components

Currently, kubeadm vendors in pieces of Go code from the components to be able to perform component config validation. This is a bad practice as it bloats the kubeadm dependencies and deviation between the vendored code and the components can lead to inconsistencies.  
To solve this, validation is going to be performed by executing a container or the binary of the component to be used with an appropriate command to verify the supplied config.  
If a target component does not have any means to verify its config, then kubeadm won’t do any validation for the time being. It will instead display a warning to the user denoting that certain component’s config is unvalidated.

### User supplied vs kubeadm generated component configs

Component configs can be specified by users in their entirety along with the kubeadm config during init or in a config map. Hewever, most users choose not to supply the configs themselves, but to trust kubeadm to generate correct configs. There are also users who would let kubeadm generate the configs and then edit them in place in the config maps that house them.

Because kubeadm looses its ability to automatically convert an older component config version into a newer one, it is best to start distinguishing between user supplied or user modified component configs and kubeadm generated ones. That way kubeadm generated configs can be thrown out and regenerated when an upgrade is required.

Unfortunately, user supplied or user modified configs would still need to be manually migrated if a config version upgrade is necessary.

To distinguish between user supplied and generated configs, kubeadm shall sign each config map, that contains generated by it config, during cluster upload. If the signature is missing or incorrect during download the config is treated as user supplied.

To calculate the signature of a generated config, kubeadm would prepare the standard `core/v1` config map. Then it would take the `data` section, order the keys lexicographically and then feed the values in that order to a SHA256 digest. After that, the same procedure is repeated for the values in the `binaryData` section too.

The resulting SHA256 digest is prepended its type - the string `sha256:` and is stored as a value of the `kubeadm.kubernetes.io/autogen-checksum` annotation in the config map object.

### Kubernetes Core vs AddOns Component Configs

Currently kubeadm maintains the component configs of both kubelet and kube-proxy. These are components that are maintained in very different ways.

Kubelet, along with API Server, Scheduler, Controller Manager and etcd, are deemed essential Kubernetes components. They are deployed and managed locally by kubeadm, using systemd services (kubelet) or static pods on the local kubelet (API Server, Scheduler, etc.). Hence, their configs are usually consumed by the components in the form of local files.

On the other hand, kube-proxy and CoreDNS/kube-dns are addons, that are managed as standard Kubernetes deployments. Hence their configs are usually accessed via config maps.

Therefore, two different config lifecycles are distinguishable here - one for Kubernetes core components and one for cluster addons.

All of the changes, proposed in this document, are going to be applied to both Kubernetes core and AddOn component config types.
However, the ultimate goal for addon component configs, is for them to be eventually managed in their entirety by cluster addon operators, instead of kubeadm.

### Stop using component config internal types

As kubeadm will no longer default, validate or migrate component configs inside its code base, the need to use component config internal types is no longer needed. In that case the usage of internal types can be removed. This would also allow us to reduce the number of imported packages and to drop one of the last dependencies on the kubernetes/kubernetes code base.  
The switch is going to be performed by replacing each internal type with a corresponding versioned type. This, by itself, is not going to have any impact on users.

### “kubeadm upgrade plan” changes

As supported component configs are expected to increase in number in the future and some of them might require manual user migration prior to upgrade, an ahead of time notice about what actions are required is a nice thing to have.  
Hence, the `kubeadm upgrade plan` command is going to provide information on the component configs. In particular, if they are managed by kubeadm or are user supplied, and if they need manual upgrade or not.

Currently, the `--config` flag of `kubeadm upgrade plan` is used to specify a new `ClusterConfiguration` and, optionally, new component configs. If a `ClusterConfiguration` is specified, but a component config is omitted, the missing component config is generated by kubeadm. The new `ClusterConfiguration` and component configs then completely replace the existing ones in the cluster.  
In addition to that, `kubeadm upgrade plan` is now also going to extend the support for the `--config` flag to allow for files that don't contain a `ClusterConfiguration`, but component configs only. If such a file is supplied, the component configs in it are used in place of the ones found in config maps on the cluster. Configs that were not specified in the file, supplied with `--config`, are going to be fetched from the cluster. This includes the currently active `ClusterConfiguration`.  
The loaded configs, that were supplied with `--config` would be taken into account when printing the additional component config information.

### "kubeadm upgrade apply" changes

As supported component configs are expected to increase in number in the future and some of them might require manual user migration prior to upgrade, a handy and secure method for supplying new configs is necessary.

For that matter `kubeadm upgrade apply` also gains the same extended semantics of the `--config` flag.  
In addition to that, the newly used configs, from the file supplied with `--config`, would be stored in the cluster as user specified configs during the upgrade process.

### Introduce the "kubeadm alpha config view upgradeable" command

This new command would fetch the YAML documents of any component configs that are in use in the cluster and require manual migration to a newer version prior to cluster upgrade. Then it would print those YAML documents to the standard output or to a file, specified with the `--output` command line option.  
If there are no configs that need manual upgrade, this command would print nothing.

### Introduce the "kubeadm alpha config print component-defaults" command

This new command would print defaults for all or selected (via command line option) component configurations. It is going to use kubeadm configuration from a cluster. However, if no cluster is available or users want to try something different, a `--config` command line option would allow for users to supply kubeadm config types which are then going to be used to generate the component configs.

The resulting configs would be printed to the standard output or a file specified at the command line via the `--output` option.

### Risks and Mitigations

#### Users will have to manually migrate more component configs than before

In general, most of the end users are not expected to modify component configs. In that case kubeadm would regenerate them if a version change is required. The combination of a user modified component config and a mandatory version change is likely going to be very rare.

#### Users will loose track of what needs manual migration and what not

The proposed changes to the "kubeadm upgrade plan" command will mitigate this risk, as kubeadm users are required to run that command prior to performing an actual upgrade.

#### Users won't be able to safely upgrade manually component configs

Actually, users would be able to do this by supplying the manually upgraded component configs in a file during the execution of `kubeadm upgrade apply`.

## Design Details

### Test Plan

In addition to unit tests, extensive E2E tests should be created to verify various aspects if the implementation.  
Most notably E2E tests should be written to verify different aspects of:
- AddOn and Kubernetes core component configs
- Configs supplied by users or generated by kubeadm
- behavior during kubeadm init and upgrade

### Graduation Criteria

Implementing the accepted proposal in full with sufficient test coverage is the bare minimum for graduation to beta.
However, as this is only an initial version of the proposal and further changes are expected over time. It's likely that the graduation criteria will change in the course of some alpha iterations.

### Upgrade / Downgrade Strategy

Upon upgrade to the new component config strategy, kubeadm users can do one of the following:

- Delete old config maps and use `kubeadm alpha config print component-defaults` to regenerate the configs and apply them upon upgrade.
- Keep old configs untouched, but refuse an upgrade if an issue is discovered (e.g. old config format is no longer accepted by the component).

### Version Skew Strategy

N/A

## Implementation History

### 09/25/2019
- Document created with Summary, Motivation, Proposal and other sections

### 04/29/2020
- Made the difference between addon and core configs more ideological
- Advanced configurations are now a non-goal as they are transferred to the Advanced Configurations KEP. Hence, also deleted the Kustomize stuff.
- Documented kubeadm generated config map signatures
- Proposed new UX improvements
