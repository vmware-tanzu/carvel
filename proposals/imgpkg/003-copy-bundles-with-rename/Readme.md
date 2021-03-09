---
title: "Rename images when copying bundles"
authors: [ "Joao Pereira <joaod@vmware.com>"]
status: "draft"
approvers: [ "Dmitriy Kalinin <dkalinin@vmware.com>", "Eli Wrenn <ewrenn@vmware.com>", "John Ryan <ryanjo@vmware.com>" ]
---

# Rename images when copying bundles

## Problem Statement

When users use `imgpkg` to copy a bundle between repositories, `imgpkg` concentrates all the copied OCI Images into the
destination repository. This behavior is unexpected by some users and is cause the following problems

@braunsonm
> For instance if I transfer a bundle to a private registry, all of my images are stored as sha
> tags on the bundle image. This makes it impossible to run these images in any context other
> than Kubernetes such as docker run without first attaining the lock file and searching
> for the image you would like to run.

@jorgemoralespou
> Definitely the original tag is the value, as the goal is for trackability. I don't really think
> retagging would provide true value.
> Trackability is being able to know where it came from without much thinking, hence keeping the
> original tag is important.

@jorgemoralespou
> When authoring a bundle, the author can't take decisions on how the user will need/want to have
> the images organized, so he can only provide with a "meaningful" default but allow the user to
> override that default

@jimconner
> Working in heavily locked-down, audited, air-gapped environments with onerous security requirements
> means having to keep the security team happy, and that means working with an off-the-shelf container
> image vulnerability scanning solution which generates reports that the security team can understand.
> The same air-gapped environment gives us a private docker registry where we have full control over
> our namespaces and can avoid conflicts pretty easily.
> A mechnaism for preserving names when relocating from public to private registry is a highly
> desirable feature for me.

The above are some relevant quotes from https://github.com/vmware-tanzu/carvel-imgpkg/issues/60
and https://github.com/vmware-tanzu/carvel-kbld/issues/79 that bootstrapped the conversation that led to this proposal.

In this proposal, we will address some of these concerns by creating a mechanism that will allow the users that are
executing the copy between registries to have more control over the location where the images are copied to.

## Terminology / Concepts

- **Repositories**: Place inside a Registry where an OCI Image is located
- **Registry**: Server that will be used to store and retrieve OCI Images (ex: goharbor.io(on prem), gcr.io(cloud),
  hub.docker.io(cloud))
- **OCI**: Open Container Initiative, [the official website](https://opencontainers.org/)
- **API**: Application Programming Interface, in this proposal case we will refer to YAML configuration as API
- **Bundle**: OCI Image that contains configuration and OCI Images, for more information
  the [original proposal](https://github.com/vmware-tanzu/carvel/blob/develop/proposals/imgpkg/001-bundles/README.md)
- **kebab-case**: Naming Case Style that consists on replacing spaces with `-`. (ex: `some name` -> `some-name`)
  . [More information](https://betterprogramming.pub/string-case-styles-camel-pascal-snake-and-kebab-case-981407998841)
- **snake_case**: Naming Case Style that consists on replacing spaces with `_`. (ex: `some name` -> `some_name`)
  . [More information](https://betterprogramming.pub/string-case-styles-camel-pascal-snake-and-kebab-case-981407998841)

## Proposal

When a user copies a bundle between registries `imgpkg` copies all the OCI Images that are part of the bundle to the
same repository. This solution was adopted to protect copy functionality from the following problems:

- Multiple images that have the same but are not the same. Assuming we have 2 bundles one that contains the image
  `my.repo.io/controller@sha256:aaaa` and another that contains the image `other.repo.io/controller@sha256:bbbb`, when
  we copy both images to the registry `third.repo.io` they would be copied to `third.repo.io/controller@sha256:aaaa`
  and `third.repo.io/controller@sha256:bbbb` respectively. This can confuse because even though now they share the same
  Repository they are completely different Images from 2 completely different Source Codes. This might cause problems
  for Registry Administrators when they try to understand what each Repository contains.
- Finding if an Image is already present in the destination repository is complicated.
- Registries like `gcr.io` support paths in their Repositories while `hub.docker.io` does not. Copying an OCI Image
  from `gcr.io/my/specific/path/controller@sha256:aaaa` to `index.docker.io` is a challenge because we would lose
  information about the original OCI Image that can be significant.

The adopted solution can solve the above problems, but it introduces some other problems, like the discoverability of
the OCI Images when they are being used in Kubernetes context or other contexts, as it is expressed in the quotes
provided in the [Problem Statement section](#Problem-Statement).

With this proposal, we will introduce a mechanism to enable the users of `imgpkg` to decide the Repository where each
OCI Image will be copied to. This will be archived by providing the `copy` command with a Strategy that `imgpkg` will
follow to place OCI Images in the desired locations.

### Goals and Non-goals

#### Goals

- Propose a solution for the problems in the [Problem Statement section](#problem-statement)
- Maintain compatibility with the current implementation
- Provide the minimum amount of strategies to solve current problems
- Define API to provide the strategy information to `imgpkg`
- Define provisioning information for this strategy

#### Non-goals

- Provide all possible combinations of strategies for copying OCI Images
- Ensure that destination Repository contains path information from Origin Repository
- Provide strict implementation guide

### Specification / Use Cases

#### API

The following example contains an example of the configuration YAML Document that would be provided to `imgpkg`

```yaml
apiVersion: imgpkg.carvel.dev/v1alpha1
kind: CopyWithRename
copyStrategy: same-repository
overrides:
  - source:
      matchRegistryRepo:
        registry: public-reg.io
        repository: simple-app
    destination: my.private-registry.io/myname/simple-app
  - source:
      matchExact: public-reg.io/exact-app@sha256:aaaaaaa
    destination: my.private-registry.io/myname/exact-app
```

Fields:

- `kind` Type of configuration, this value should always be `CopyWithRename`. Used to allow `imgpkg` to understand what
  type of configuration this document is defining
- `copyStrategy` This field will contain the Strategy that will be used by `imgpkg` the default value
  is `same-repository`. Check the [Copy Strategies section](#copy-strategies) for more information.
- `overrides` This is a list of overrides for the `copyStrategy`. In this list, the user can define specific behavior
  for a particular set of OCI Images. Check the [Overrides section](#overrides) for more information on overrides.

#### Copy Strategies

In this proposal, only 2 strategies will be defined, but in the future, more options can be made available

##### Copy all OCI Images to the same Repository as the Bundle

This strategy is the current strategy that `imgpkg` uses. Given this is the default strategy if this is the intended
behavior the users will not need to provide any configuration to the copy command.

When defining the YAML configuration the user can provide the following configuration

```yaml
apiVersion: imgpkg.carvel.dev/v1alpha1
kind: CopyWithRename
copyStrategy: same-repository
```

If the Bundle contains a particular OCI Image that the user wants to place in a different Repository the following
configuration can be used

```yaml
apiVersion: imgpkg.carvel.dev/v1alpha1
kind: CopyWithRename
copyStrategy: same-repository
overrides:
  - source:
      matchExact: public-reg.io/exact-app@sha256:aaaaaaa
    destination: my.private-registry.io/myname/exact-app
```

The above configuration will ensure that the OCI Image `public-reg.io/exact-app@sha256:aaaaaaa` will be copied
to `my.private-registry.io/myname/exact-app@sha256:aaaaaaa`. For other options of matching check
the [Overrides section](#overrides)

##### Copy all OCI Images to the new Registry but maintain the Repository

This strategy will copy each OCI Image to the target Registry and will store it in the same Repository as the OCI Image
originated.

**Examples:**

|Original Registry and Repository|Final Registry and Repository|
|---|---|
|origin.io/my-image@sha256:aaaa|destination.io/my-image@sha256:aaaa|
|origin.io/some/path/my-image@sha256:aaaa|destination.io/my-image@sha256:aaaa|
|origin.io/my-image@sha256:aaaa|destination.io/destination/path/my-image@sha256:aaaa|
|origin.io/user/initialmy-image@sha256:aaaa|destination.io/destination/path/my-image@sha256:aaaa|

The configuration to accomplish this is

```yaml
apiVersion: imgpkg.carvel.dev/v1alpha1
kind: CopyWithRename
copyStrategy: maintain-repository
```

This strategy also allow overrides

```yaml
apiVersion: imgpkg.carvel.dev/v1alpha1
kind: CopyWithRename
copyStrategy: maintain-repository
overrides:
  - source:
      matchExact: public-reg.io/exact-app@sha256:aaaaaaa
    destination: my.private-registry.io/myname/exact-app
```

#### Overrides

### Other Approaches Considered

- Keep current behavior but provide more insight into the OCI Images inside a Bundle

  The proposal was to create a new command that would read a bundle and would provide the user with the Original
  Registry and Repository of a particular OCI Image as well as the new location. Something similar
  to `imgpkg images -b my.private-registry.io/bundle-with-simple-app`.

  Pros:
    - Command could provide all the needed information about the new OCI Image location
    - From the output of the command would be easy for the user to execute `docker run` or `podman run` with the OCI
      Image they were looking for

  Cons:
    - Kubernetes Manifest would still be hard to understand where a particular OCI Image is from
    - Require `imgpkg` to decode this information from the Bundle

- Tagging with origin Registry and Repository each OCI Image and copy OCI Images to the Bundle Repository

  The idea behind this approach would be to continue with the current behavior but whenever an OCI Image is copied it
  would be tagged with a kebab-case or snake_case Tag that would represent the original Registry and Repository(ex:
  origin: `gcr.io/some/location/repo@sha256:aaa` would be tagged with `gcr-io-some-location-repo`).

  Pros:
    - No major change required to `imgpkg`
    - Easier to find if the user can read all tags associated with a particular Repository

  Cons:
    - If the Digest is not included in Tag, the Tag might be overwritten if we copy a newer version of the bundle
    - Is easier to find the original location but still would be hard if the user wanted to run the OCI image
      using `docker run` or `podman run`

## Open Questions

- Who will `imgpkg` know where to find the OCI Images after they are copied?

  The problem can happen if a user copies a Bundles from Registry A -> Registry B and afterwards a different user wants
  to copy the Bundle from Registry B to Registry C, how might `imgpkg` know where to get the Images from?

- What are the implication for Nested Bundles?

  The same problem as the above questions but we need to understand if there is any drawback on the Nested Bundles case

- Will we need to keep some records about the location of the OCI Images after the first copy?

    - If we need how might we save that in a place that `imgpkg` can easily retrieve in the future?
    - Maybe a new OCI Image in the Bundle Repository with a TAG that we know about?

- Can we wait for the Artifact OCI spec change be complete?

    - If we waited what would be the drawbacks?
    - Do we need to wait for it to be able to implement this feature?

- What is the good initial chunk of work that would make more sense?
- Would it be helpful to implement some intermediate steps while we build this feature?

## Answered Questions

_A list of questions that have been answered._
