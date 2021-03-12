---
title: "Rename images when copying bundles"
authors: [ "Joao Pereira <joaod@vmware.com>"]
status: "in-review"
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
- **MVP**: Minimum Viable Product

## Proposal

When a user copies a bundle between registries `imgpkg` copies all the OCI Images that are part of the bundle to the
same repository. This solution was adopted to protect copy functionality from the following problems:

- Multiple images that have the same Repository Name but are not the same. Assuming we have 2 bundles one that contains
  the image  `my.repo.io/controller@sha256:aaaa` and another that contains the
  image `other.repo.io/controller@sha256:bbbb`, when we copy both images to the registry `third.repo.io` they would be
  copied to `third.repo.io/controller@sha256:aaaa` and `third.repo.io/controller@sha256:bbbb` respectively. This can
  confuse because even though now they share the same Repository they are completely different Images from 2 completely
  different Source Codes. This might cause problems for Registry Administrators when they try to understand what each
  Repository contains.
- Finding if an Image is already present in the destination repository is complicated.
- Registries like `gcr.io` support paths in their Repositories while `hub.docker.io` does not. Copying an OCI Image
  from `gcr.io/my/specific/path/controller@sha256:aaaa` to `index.docker.io` is a challenge because we would lose
  information about the original OCI Image that can be significant.
- Copying OCI Images to different repositories can require different user or authentication. When copying an OCI Image
  to a particular Repository the user needs to have credentials to do so, in a scenario where the
  repository `my.repo.io/controller` was previously created by a different user, the current user might not have
  permission to copy the new OCI Image to that particular Repository.
- ECR requires users to create a Repository before pushing OCI
  Images. [reference](https://docs.aws.amazon.com/AmazonECR/latest/userguide/docker-push-ecr-image.html)

The adopted solution can solve the above problems, but it introduces some other problems, like the discoverability of
the OCI Images when they are being used in Kubernetes context or other contexts, as it is expressed in the quotes
provided in the [Problem Statement section](#Problem-Statement).

With this proposal, we will introduce a mechanism to enable the users of `imgpkg` to decide the Repository where each
OCI Image will be copied to. This will be achieved by providing the `copy` command with a Strategy that `imgpkg` will
follow to place OCI Images in the desired locations.

### Goals and Non-goals

#### Goals

- Propose a solution for the problems in the [Problem Statement section](#problem-statement)
- Maintain compatibility with the current implementation
- Provide the minimum amount of strategies to solve current problems
- Define API to provide the Strategy information to `imgpkg`
- Define provisioning information for this strategy

#### Non-goals

- Provide all possible combinations of strategies for copying OCI Images
- Provide strict implementation guide

### Specification / Use Cases

#### API

The following example contains an example of the configuration YAML Document that would be provided to `imgpkg`

```yaml
apiVersion: imgpkg.carvel.dev/v1alpha1
kind: CopyConfig
strategy: SingleRepository
overrides:
  - imageRepo: public-reg.io/simple-app
    destination: my.private-registry.io/myname/simple-app
  - image: public-reg.io/exact-app@sha256:aaaaaaa
    destination: my.private-registry.io/myname/exact-app
```

Fields:

- `apiVersion` (required; string) Version of this configuration.
- `kind` (required; `CopyConfig`) Type of configuration, this value should always be `CopyConfig`. Used to
  allow `imgpkg` to understand what type of configuration this document is defining
- `strategy` (required; `SingleRepository|MaintainOriginRepository`) This field will contain the Strategy that will be
  used by `imgpkg` the default value is `SingleRepository`. Check the [Copy Strategies section](#copy-strategies) for
  more information.
- `overrides` (optional; array of overrides) This is a list of overrides for the `strategy`. In this list, the user can
  define specific behavior for a particular set of OCI Images. Check the [Overrides section](#overrides) for more
  information on overrides.
- `mappingFunctionYttStar` (optional; `ytt` starlark function) This field contains an implementation of a `ytt` starlark
  function that will be responsible for creating a map between the OCI Images and the new locations

**Note:** `overrides` and `mappingFunctionYttStar` cannot be present at the same time in a configuration.

#### Copy Strategies

In this proposal, only 2 strategies will be defined, but in the future, more options can be made available. These
Strategies and Overrides will apply to all OCI Images that are contained in the Bundle that is being copied, this
includes all OCI Images from the current Bundle and Nested Bundles.

##### Copy all OCI Images to the same Repository as the Bundle

This strategy is the current strategy that `imgpkg` uses. Given this is the default strategy if this is the intended
behavior the users will not need to provide any configuration to the copy command.

When defining the YAML configuration the user can provide the following configuration

```yaml
apiVersion: imgpkg.carvel.dev/v1alpha1
kind: CopyConfig
strategy: SingleRepository
```

Given the Bundle contains a particular OCI Image that the user wants to place in a different Repository the following
configuration can be used

```yaml
apiVersion: imgpkg.carvel.dev/v1alpha1
kind: CopyConfig
strategy: SingleRepository
overrides:
  - image: public-reg.io/exact-app@sha256:aaaaaaa
    destination: my.private-registry.io/myname/exact-app
```

The above configuration will ensure that the OCI Image `public-reg.io/exact-app@sha256:aaaaaaa` will be copied
to `my.private-registry.io/myname/exact-app@sha256:aaaaaaa`. For other options of matching check
the [Overrides section](#overrides)

##### Copy all OCI Images to the new Registry but maintain the Original Repository name

This strategy will copy each OCI Image to the target Registry and will store it in the same Repository as the OCI Image
originated.

The base configuration to enable this strategy is

```yaml
apiVersion: imgpkg.carvel.dev/v1alpha1
kind: CopyConfig
strategy: MaintainOriginRepository
```

When copying a Bundle to `destination.io/bundle` all OCI Images will be copied
to `destination.io/{Original Repository Name}`

**Example:**

- OCI Image from `origin.io/my-image@sha256:aaaa` will be copied to `destination.io/my-image@sha256:aaaa`
- OCI Image from `origin.io/some/path/my-image@sha256:aaaa` will be copied to `destination.io/my-image@sha256:aaaa`

When copying a Bundle to `destination.io/some/path/bundle` all OCI Images will be copied
to `destination.io/some/path/{Original Repository Name}`

**Example:**

- OCI Image from `origin.io/my-image@sha256:aaaa` will be copied to `destination.io/some/path/my-image@sha256:aaaa`
- OCI Image from `origin.io/image/origin/my-image@sha256:aaaa` will be copied
  to `destination.io/some/path/my-image@sha256:aaaa`

This strategy also allows overrides

```yaml
apiVersion: imgpkg.carvel.dev/v1alpha1
kind: CopyConfig
strategy: MaintainOriginRepository
overrides:
  - image: public-reg.io/exact-app@sha256:aaaaaaa
    destination: my.private-registry.io/myname/exact-app
```

For other options of matching check the [Overrides section](#overrides)

#### Overrides

The `overrides` key in the configuration will contain a list of a Matcher and a Destination. All the OCI Images present
in the Bundle, and Nested Bundles will be matched against this list, but only the OCI Images the user wish to Override
should be present in this section. If an OCI Image do not Match any of these Overrides it will be copied based on the
strategy defined.

In the event that 2 Overrides match a particular OCI Image the first Override in the list takes precedence.

In the following example the `exact-app` OCI Image will be copied to `my.private-registry.io/myname/exact-app`

```yaml
overrides:
  - image: public-reg.io/exact-app@sha256:aaaaaaa
    destination: my.private-registry.io/myname/exact-app
  - imageRepo: public-reg.io/exact-app
    destination: my.private-registry.io/image/will/not/be/copied/here
```

**Note:** `overrides` key cannot be present in the configuration if the key `mappingFunctionYttStar` is also defined

##### Assumptions

- In the beginning, a few possible overrides will be provided, eventually only the exact match one
- The user cannot copy different OCI Images to different registries

##### Exact match

This overrides will do an exact match of the OCI Image present in `.imgpkg/images.yml` and will copy the OCI Image to
the Repository that is provided in the `destination` field

Schema fields:

- `image` (required; string) Registry,Repository and Tag or SHA that will be used to do an exact match. This can be an
  OCI Image with a Tag or a SHA. (ex: `my.repo.io/img@sha256:aaaaaa` or `my.repo.io/img:my-tag`)
- `destination` (required; string) Registry and Repository the OCI Image will be copied to. This field can only contain
  Registry+Repository.

Example of the override using a SHA

```yaml
image: public-reg.io/exact-app@sha256:aaaaaaa
destination: my.private-registry.io/myname/exact-app
```

Example of override using a Tag

```yaml
image: public-reg.io/exact-app:some-tag
destination: my.private-registry.io/myname/exact-app
```

In the case of exact matching with tag `imgpkg` will have to do the matching using both the Registry+Repository and the
annotation `imgpkg.carvel.dev/image-tags` to determine if a particular OCI Image match.

##### Match by Registry and Repository

This Matcher will match the Registry+Repository provided in the field `imageRepo` against all the OCI Images that are
present in the ImagesLock files for the Bundle and Nested Bundles. In order to match the Registry+Repository have to be
equal, no wildcards allowed.

Schema fields:

- `imageRepo` (required; string) OCI Image that will be used to do an exact match on Registry and Repository.
- `destination` (required; string) Registry and Repository the OCI Image will be copied to. This field can only contain
  Registry+Repository.

Example of the override

```yaml
imageRepo: public-reg.io/simple-app
destination: my.private-registry.io/myname/simple-app
```

Result:

The OCI Image present in `public-reg.io/simple-app` will be copied to `my.private-registry.io/myname/simple-app`. The
example assumes that in the ImagesLock file there is 1 entry that will match `public-reg.io/simple-app@sha256:*`.

#### Advanced Mapping Scenarios

To enable more advanced matching integrating with `ytt` should be in the roadmap.

Example of using `ytt` starlark function that maps an OCI Image with a destination

```yaml
apiVersion: imgpkg.carvel.dev/v1alpha1
kind: CopyConfig
strategy: MaintainOriginRepository
mappingFunctionYttStar: |
  def process(image):
    registry = 'my.private-registry.io/'
    if regexMatch(image, '.*image1.*')
      return registry + 'myname/exact-app'
    elif repository(image) == 'other-image'
      return registry + 'some-other-repo'
    end
    return registry + repository(image)
  end
```

**Note:** `mappingFunctionYttStar` key cannot be present in the configuration if the key `overrides` is also defined

The mapping function will have to respect the following definition `process(string) -> string`.

To help users to create simpler mapping the following functions will be available:

##### regexMatch

Function that given a Regular Expression, will check if the provided string matches.

Definition: `def regexMatch(image, expression) -> boolean`

Parameters:

- `image` string with the Registry + Repository of the OCI Image
- `expression` string with the Regular Expression that is being matched

Return:

- `True` if the `image` matches the Regular Expression
- `False` if the `image` does not match the regular expression

##### registry

Function will remove all information from an OCI Image except the registry location.

Definition: `def registry(image) -> string`

Parameters:

- `image` string with the Registry + Repository of the OCI Image

Return: Registry URL

**Examples:**

- `registry('https://index.docker.io/u/ubuntu@sha256:aaaaaaaa')` will return `index.docker.io`
- `registry('gcr.io/some/deep/path/my-image@sha256:aaaaaaaa')` will return `gcr.io`
- `registry('gcr.io/some/deep/path/my-image:some-tag')` will return `gcr.io`

##### repository

Function will remove all information from an OCI Image location and return the Repository name.

Definition: `def repository(image) -> string`

Parameters:

- `image` string with the Registry + Repository of the OCI Image

Return: Repository name

**Examples:**

- `repository('https://index.docker.io/u/ubuntu@sha256:aaaaaaaa')` will return `ubuntu`
- `repository('gcr.io/some/deep/path/my-image@sha256:aaaaaaaa')` will return `my-image`
- `repository('gcr.io/some/deep/path/my-image:some-tag')` will return `my-image`

##### matchTag

Function will check if a particular tag is associated with an OCI Image.

Definition: `def matchTag(image, tag) -> boolean`

Parameters:

- `image` string with the Registry + Repository + SHA of the OCI Image
- `tag` string with the tag to check

Return:

- `True` if the `tag` can be found in the annotation `imgpkg.carvel.dev/image-tags` for the `image`
- `False` if the `tag` cannot be found in the annotation `imgpkg.carvel.dev/image-tags` for the `image`

#### Nested bundles considerations

If we assume the following bundle

```
other.registry.io/main-bundle
  - another.registry.io/img2
  - yet.another.registry.io/nested-bundle-2
    - world.io/img3
```

When a user tries to copy `other.registry.io/main-bundle` to `registry.acme.io/main-bundle` `imgpkg` will have to find
all the OCI Images that are part of this bundle. Before this proposal `imgpkg` could assume that any particular OCI
Image could be found in one of the following locations:

1. Bundle Repository we are copying from
1. Original location of the OCI Image (if the bundle was never copied)

After this feature is implemented `imgpkg` cannot assume the location of the OCI Image in question, since these Bundle
might or might not have been copied and changed the location where the OCI Images are stored. An example of this can be
found next

1. Create the nested-bundle-2 with the following ImagesLock file
   ```yaml
   apiVersion: imgpkg.carvel.dev/v1alpha1
   kind: ImagesLock
   images:
   - image: world.io/img3@sha256:aaaaaaaaaa
   ```

   `imgpkg push -b yet.another.registry.io/nested-bundle-2 -f folder1`

1. Copy `yet.another.registry.io/nested-bundle-2` to the Registry `other.registry.io`
   `imgpkg copy -b yet.another.registry.io/nested-bundle-2@sha256:bbbbbbbbbb --to-repo other.registry.io/main-bundle`

1. Create `other.registry.io/main-bundle`

   Using the following ImagesLock
   ```yaml
   apiVersion: imgpkg.carvel.dev/v1alpha1
   kind: ImagesLock
   images:
   - image: other.registry.io/nested-bundle-2@sha256:bbbbbbbbbb
   ```

   `imgpkg push -b other.registry.io/main-bundle -f folder2`

When we prepare to copy this new bundle we need to consider that the OCI Image `world.io/img3@sha256:aaaaaaaaaa` can be
found in the following places:

- other.registry.io/main-bundle@sha256:aaaaaaaaaa
- other.registry.io/nested-bundle-2@sha256:aaaaaaaaaa
- yet.another.registry.io/nested-bundle-2@sha256:aaaaaaaaaa
- world.io/img3@sha256:aaaaaaaaaa
- Or any other place the OCI Image was copied to using a strategy in this proposal

To try to address this problem we can create a new OCI Image that contains 1 layer with 1 file
called `images-locations.yml` that will have the following layout

```yaml
apiVersion: imgpkg.carvel.dev/v1alpha1
kind: ImagesLocation
images:
  - image: world.io/img3@sha256:aaaaaaaaaa
    location: yet.another.registry.io/nested-bundle-2
```

Field explanation:

- `apiVersion` (required; string) Version of this configuration.
- `kind` (required; `ImagesLocation`) Type of configuration, this value should always be `ImagesLocation`. Used to
  allow `imgpkg` to understand what type of configuration this document is defining
- `images` (required; []array) Must contain on entry per OCI Images present in the ImagesLock file.
- `images[].image` (required; string) This value MUST match an OCI Image defined in ImagesLock file for the Bundle.
- `images[].location` (required; string) Is the location where this OCI Image was copied to.

When copying a Bundle between Registries and/or Repositories this new OCI Image will be created in the destination
Repository and will be tagged with the tag `sha256-{Bundle SHA}-imgpkg.locations`. This is not a perfect solution since
Tags are mutable, but this will fix the problem for now.

##### Caveat

Since `imgpkg` would be using tags to find the new location for the OCI Images of a particular Bundle there is no
guarantee that this tag is accurate. This should be considered a best effort to locate the OCI Images. When the tag is
not present in the expected location `imgpkg` will assume that the Bundle was copied with `SingleRepository` strategy
with no overrides.

### Use Cases

#### As a User When I copy a Bundle I want to ensure tags present in ImagesLock file are preserved

**How might `imgpkg` help to accomplish this Use Case:**

- Add one new annotation `imgpkg.carvel.dev/image-tags` that supports a comma separated list of tags
- While copying OCI Images `imgpkg` will read this annotation and will tag the OCI Image with the provided tags

#### As a User When I inspect my Kubernetes Manifest I can identify the origin of the OCI Images that is running

**Goals**

- Provide a way for a tool that manages the Manifests to update the images with the more up to date location of an Image

**Non-goals**

- `imgpkg` be aware of Kubernetes Manifests

**How might `imgpkg` help to accomplish this Use Case:**

- Maintaining annotations provided in ImagesLock
- Allow the user to customize the copy of OCI Images to match a naming convention that could help the user identify the
  original OCI Image using only the Repository name
- Ensure that tags present in the ImagesLock folder are maintained when OCI Images are copied

#### As a User When I can copy a Bundle to my local Registry to run one of the OCI Images

**Goals**

- Provide a way for users to know how to find the OCI Image they want to run

**Non-goals**

- `imgpkg` understand how to run images

**How might `imgpkg` help to accomplish this Use Case:**

- Allow the user to customize the copy of OCI Images to match a naming convention that could help the user identify the
  original OCI Image using only the Repository name
- Ensure that tags present in the ImagesLock folder are maintained when OCI Images are copied

#### As a User When I look at the OCI Images in a Registry I am able to easily understand what each OCI Images is used for

**Goals**

- Provide a way for users to differentiate OCI Image by looking at the Repository name

**How might `imgpkg` help to accomplish this Use Case:**

- Allow the user to customize the copy of OCI Images to match a naming convention that could help the user identify the
  original OCI Image using only the Repository name
- Ensure that tags present in the ImagesLock folder are maintained when OCI Images are copied

#### As a User When I have a Bundle with a big number of OCI Images I want to be able to create the CopyConfig file easily

**How might `imgpkg` help to accomplish this Use Case:**

- New command that generates a configuration file from a Bundle

  `imgpkg tools generate copy-config -b repo.io/bundle@sha256:aaaaaa -o /tmp/copy-config.yml`

### Implementation breakdown

This is quite a big proposal that can take some time to implement. In this section we try to split the current work
needed into multiple phases that could provide users with the most important parts of this feature as soon as possible.

#### Phase 0 - Implement Tags preservation

After this phase is complete the users would be able to:

1. `imgpkg` will tag OCI Images when copying a Bundle
1. `kbld` fill the new annotation with the tag values

   This need to happen in operations:
    - When `kbld` is creating a new OCI
      Image, [reference](https://carvel.dev/kbld/docs/latest/config/#imagedestinations)
    - When `kbld` is resolving OCI Images from configuration

New annotation:

Create Annotation `imgpkg.carvel.dev/image-tags` that will support a comma separated list of tags that should be applied
to the OCI Images when they are being copied between Registries

#### Phase 1 - Implementation of strategies (MVP-1)

After this phase is complete the users would be able to:

1. `imgpkg` will tag OCI Images when copying a Bundle
1. `kbld` fill the new annotation with the tag values
1. Copy OCI Images between registries choosing the strategy that best suites their needs.
1. When user is copying Bundles and do not select a strategy, copy will use `SingleRepository` as the default strategy.
1. Pull a Bundle to disk and see the correct locations for all OCI Images in the ImagesLock files.

New Flags:

- Copy command
    - `-c filename` and `--configuration filename` provide the configuration to select a strategy

API Implementation:
At this point in time the user can only select a Copy Strategy that will apply to all OCI Images in the Bundle

```yaml
apiVersion: imgpkg.carvel.dev/v1alpha1
kind: CopyConfig
strategy: (SingleRepository|MaintainOriginRepository)
```

#### Phase 2 - Inclusion of simple overrides (MVP-2)

After this phase is complete the users would be able to:

1. `imgpkg` will tag OCI Images when copying a Bundle
1. `kbld` fill the new annotation with the tag values
1. Copy OCI Images between registries choosing the strategy that best suites their needs.
1. When user is copying Bundles and do not select a strategy, copy will use `SingleRepository` as the default strategy.
1. Pull a Bundle to disk and see the correct locations for all OCI Images in the Images Lock files.
1. Create Overrides for the selected strategy

API Implementation:
At this point in time the user can select a Copy Strategy and Overrides that will apply to all OCI Images in the Bundle

```yaml
apiVersion: imgpkg.carvel.dev/v1alpha1
kind: CopyConfig
strategy: (SingleRepository|MaintainOriginRepository)
overrides:
  - imageRepo: public-reg.io/simple-app
    destination: my.private-registry.io/myname/simple-app
  - image: public-reg.io/exact-app@sha256:aaaaaaa
    destination: my.private-registry.io/myname/exact-app
```

Overrides available:

- `image` matches exactly the OCI Image location
- `imagesRepo` matches exactly the Registry and Repository

#### Phase 3 - Add more complex mapping functionality (Final)

After this phase is complete the users would be able to:

1. `imgpkg` will tag OCI Images when copying a Bundle
1. `kbld` fill the new annotation with the tag values
1. Copy OCI Images between registries choosing the strategy that best suites their needs.
1. When user is copying Bundles and do not select a strategy, copy will use `SingleRepository` as the default strategy.
1. Pull a Bundle to disk and see the correct locations for all OCI Images in the Images Lock files.
1. Create Overrides for the selected strategy
1. Create a mapping function using `ytt` starlark

API Implementation:
At this point in time the user can select a Copy Strategy that will apply to all OCI Images in the Bundle. To create
Overrides for the selected strategy the user will have to choose between Overrides or a Mapping function.

```yaml
apiVersion: imgpkg.carvel.dev/v1alpha1
kind: CopyConfig
strategy: (SingleRepository|MaintainOriginRepository)
mappingFunctionYttStar: |
  def process(image):
    registry = 'my.private-registry.io/'
    if regexMatch(image, '.*image1.*')
      return registry + 'myname/exact-app'
    elif repository(image) == 'other-image'
      return registry + 'some-other-repo'
    end
    return registry + repository(image)
  end
```

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

  In summary, we believe that this could help the users find the OCI Images and provide other helpful information about
  the Bundle. The first con is one of the problems that this proposal is trying to solve, because of that we decided
  that this would not be a good approach.

- Tagging with origin Registry and Repository each OCI Image and copy OCI Images to the Bundle Repository

  The idea behind this approach would be to continue with the current behavior but whenever an OCI Image is copied it
  would be tagged with a kebab-case or snake_case Tag that would represent the original Registry and Repository(ex:
  origin: `gcr.io/some/location/repo@sha256:aaa` would be tagged with `gcr-io-some-location-repo`).

  Pros:
    - No major change required to `imgpkg`
    - Easier to find if the user can read all tags associated with a particular Repository

  Cons:
    - If the Digest is not included in Tag, the Tag might be overwritten if we copy a newer version of the bundle
    - Is easier to find the original location using the tag, but when a user tries to run the OCI image
      using `docker run` or `podman run` the line would look something like
      this `docker run my.destination.io/bundle:gcr-io-some-path-to-image-my-image-sha256-aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa`
    - Kubernetes Manifest would still be hard to understand where a particular OCI Image is from

  In summary, we believe this could help find the OCI Images in question by reading the tag. Nevertheless, these tags
  would not be provided to Kubernetes Manifests. The cons for this approach do not help to solve the problem statement
  this proposal is addressing.

## Open Questions

- Can we wait for the Artifact OCI spec change to be complete?

    - If we waited what would be the drawbacks?
    - Do we need to wait for it to be able to implement this feature?

- Would it be helpful to implement some intermediate steps while we build this feature?

- Can we in a first approach just provide a command-line flag to change strategies?
    - Would this reduce the total amount of complexity of this feature?

- Should copying OCI Images to different Registries be allowed?

  My initial inclination is to do not allow and even raise an error if the Registries do not match.

## Answered Questions

- How will `imgpkg` know where to find the OCI Images after they are copied?

  The problem can happen if a user copies a Bundles from Registry A -> Registry B and afterward a different user wants
  to copy the Bundle from Registry B to Registry C, how might `imgpkg` know where to get the Images from?

  **Answer:** Created a new OCI Image that will store this information, for more details
  check [Nested bundles considerations section](#nested-bundles-considerations)
- What is the implication for Nested Bundles?

  The same problem as the above questions, but we need to understand if there is any drawback on the Nested Bundles case

  **Answer:** The main drawback is the need to retain information to about where an OCI Image can be found, for more
  details check [Nested bundles considerations section](#nested-bundles-considerations)

- Will we need to keep some records about the location of the OCI Images after the first copy?

    - If we need how might we save that in a place that `imgpkg` can easily retrieve in the future?

      **Answer:** we need to save this information. In case we do not do it we will have to assume we do not know where
      the OCI Images can be found. This will not be a problem if the OCI Images are copied with the Strategy to maintain
      all the OCI Images inside the same Repository where the Bundles are stored
    - Maybe a new OCI Image in the Bundle Repository with a TAG that we know about?

      **Answer:** This was the approach selected but this have a big caveat, for more details
      check [Nested bundles considerations section](#nested-bundles-considerations)

- What is the good initial chunk of work that would make more sense?

  **Answer:** Created [the section Implementation Breakdown](#implementation-breakdown) to try to split the work into
  manageable phases
