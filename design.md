# Bazel Module (bzlmod) Support in Renovate

```yaml
author: Chuck Grindel<chuck.grindel@gmail.com>
status: Under review
created: 2023-04-10
```

## Abstract

[Renovate] is a popular service for updating external dependencies. It [currently supports] updating
Bazel dependencies described in [WORKSPACE-managed repositories]. With the introduction of [Bazel
modules], [Renovate] needs to be modified to support the new external dependency mechanism.

## Requirements

- External Dependency Discovery
  - Support extracting external dependency information from `MODULE.bazel` files.
    - Read external dependency information from [bazel_dep] declarations.
    - Support module overrides.
      - [Single version override]
      - [Multiple version override]
      - [Non-registry overrides]
- Registry Discovery
  1. Support parsing of `.bazelrc` and files imported in the `.bazelrc` looking for [--registry]
     flags.
  2. If no registry values are detected, default to a Renovate configuration value. 
  3. Otherwise, use the [Bazel Central Registry].
- Bazel Module Release Detection
  - Detect new module versions when they are introduced to a module registry (e.g., [Bazel Central
    Registry]).
    - Triggering updates based upon GitHub releases is problematic as the update to a registry is a
      separate step that may not complete for minutes to hours (i.e., human approval of a pull
      request).
- Versioning
  - Support [Bazel module version formats] and [compatibility level].
  - Support [yanked versions]. A version can be yanked by a maintainer when it should be avoided.

### Requirements Still Under Discussion

- Bazel Compatibility
  - Identify the Bazel version being used:
    1. Support parsing of `.bazelversion` to detect the Bazel version.
    2. Allow the Bazel version to be specified in a Renovate configuration value.
    3. No Bazel version detected
  - If a Bazel version is present, evaluate the `bazel_compatibility` expressions for the module
    version to determine if it is an acceptable upgrade candidate.
- Different Update Rules for Ruleset/Library Repositories
  - Support different version resolution logic for libraries (i.e., is depended upon by other
    projects) vs executable (i.e., is not depdended upon by other projects) repositories .

### Future Work

- Add support for [Bazel module lockfiles].
  - As of April 2023, this functionality is scheduled for Bazel 7.0.

## Design

### Versioning

Versioning for Bazel modules has two components: the `version` string and the `compatibility_level`
integer. The `version` string has two roles. First, it identifies a unique release for a Bazel
module. Second, it acts as a sort field aiding in the ordering of releases. The
`compatibility_level` groups releases identifying those which are compatible with each other.
Together, these values dictate how to identify upgrade candidates for a Bazel module.

Because none of [the existing Renovate versioning schemes] satisfies the requirements for Bazel
modules, a new versioning scheme, `bazel-module`, will be implemented. 

#### Version Formats

Bazel modules support a [relaxed SemVer specification]. Specifically, [the version documentation]
states:

> The version format we support is `RELEASE[-PRERELEASE][+BUILD]`, where `RELEASE`, `PRERELEASE`,
> and `BUILD` are each a sequence of "identifiers" (defined as a non-empty sequence of ASCII
> alphanumerical characters and hyphens) separated by dots. The `RELEASE` part may not contain
> hyphens.

Examples of valid `version` values are listed below:

```sh
1.2.3                 # SemVer
7.0.0-pre.20230330.3  # SemVer with prerelease
1.0+build2            # SemVer with two parts and a build
20230125.2            # Date-patch format
```

#### Version Sorting

To identify whether a release is an upgrade candidate, we need know how to order the version values.
The `bazel-module` versioning scheme will implement [the same Bazel module version sort] as is
implemented in Bazel.

#### Version Selection

For a given dependency, the Renovate manager will create an upgrade pull request for the highest
version for a compatibility level. This implies that zero or more pull requests could be created
depending upon the available versions for a dependency.

To explore the implications of this rule, lets look at some examples. In all of the examples, the
repository that is using Renovate has a dependency on `rules_foo` at version `1.0.0`. This version
has a `compatibility_level` value of `1`. Renovate pull requests (PR) will be created for versions
having a checkmark (✅).


##### Example: Single Upgrade Candidate with Same Compatibility Level

| Available Versions | Compatibility Level | PR |
| ------------------ | ------------------- |:--:|
| `1.0.0` | `1` | |
| `1.1.0` | `1` | ✅ |

##### Example: Single Upgrade Candidate with Different Compatibility Level

| Available Versions | Compatibility Level | PR |
| ------------------ | ------------------- |:--:|
| `1.0.0` | `1` | |
| `2.0.0` | `2` | ✅ |

##### Example: Multiple Upgrade Candidates

| Available Versions | Compatibility Level | PR |
| ------------------ | ------------------- |:--:|
| `1.0.0` | `1` | | 
| `1.1.0` | `1` | |
| `1.1.1` | `1` | ✅ |
| `2.0.0` | `2` | ✅ |

In this example two pull requests are created. One for compatibility level `1` and the other for
compatibility level `2`. The `1.1.0` release is ignored because there is a higher version available
with the same compatibility level (`1.1.1`).

##### Example: Prerelease Versions

| Available Versions | Compatibility Level | PR |
| ------------------ | ------------------- |:--:|
| `1.0.0` | `1` | |
| `1.1.0` | `1` | ✅ |
| `2.0.0-pre.20230412` | `2` | |

Even though there is a release with compatibility level `2`, it is ignored because it is a
prerelease.

### Datasource

A [datasource in the renovate framework] encapsulates the retrieval of release information for a
package. This design introduces a new datasource, `bazel-registry`. It will query a [Bazel registry]
for the specified module. If it is found, release information for the module will be returned.
Otherwise, a `null` value will be retured.

The `bazel-registry` datasource will leverage Renovate's [custom registry support] to search across
[multiple Bazel registries]. It will be configured to use Renovate's `hunt` [registry strategy],
searching across the configured registry URLs until the named Bazel module is found.

The default versioning scheme will be `bazel-module`, as discussed earlier in the document.

In addition to the `bazel-registry` datasource, the [github-releases datasource] will be used to
reconcile upgrades for Bazel modules that have a `git_override` declaration.

### Manager
 
A [manager in the renovate framework] contains the code for extracting dependency information from
repository files. This design introduces a manager, `bazel-module`, for processing Bazel module
dependencies. Like the [`bazel` renovate manager], the `good-enough-parser`'s [Starlark language
parser] will be used to parse a repository's `MODULE.bazel` file.

#### Extract

The `bazel-module` manager will detect and process [bazel_dep], [archive_override], [git_override],
[single_version_override], and [multiple_version_override].

A [bazel_dep] declaration describes a dependent module at a specified version. The version value
will be evaluated for upgrade unless an override declaration for the module is found.

An [archive_override] declaration will prevent any upgrade activity for the module.

A [git_override] declaration describes that the module should come from a specific git commit. If
this declaration is found, the `version` from the [bazel_dep] will be ignored. The upgrade policy
that 

<!-- 

> TODO (grindel): Use [getRangeStrategy](https://github.com/renovatebot/renovate/blob/main/docs/development/adding-a-package-manager.md#getrangestrategyconfig-optional)
> to support the library vs executable repository strategy?

-->

### Other Changes

#### Update the Renovate Bazel Documentation

The renovate repository contains [a document related to Bazel]. This document will be updated to
contain information about Bazel module support.

## References

- [bzlmod Version.java]
- [bzlmod VersionTest.java]
- [bzlmod Selection.java]
- [Slack discussion about library vs executable repositories]
- [Bazel registries]
- [ManagerApi]

<!-- LINKS -->

[--registry]: https://bazel.build/reference/command-line-reference#flag--registry
[Bazel Central Registry]: https://github.com/bazelbuild/bazel-central-registry
[Bazel module version formats]: https://bazel.build/external/module#version_format
[Bazel modules]: https://bazel.build/external/module
[Bazel registries]: https://bazel.build/external/registry
[Bazel registry]: https://bazel.build/external/registry
[Multiple version override]: https://bazel.build/external/module#multiple_version_override
[Non-registry overrides]: https://bazel.build/external/module#non-registry_overrides
[Renovate]: https://github.com/renovatebot/renovate
[Single version override]: https://bazel.build/external/module#single_version_override
[Slack discussion about library vs executable repositories]: https://bazelbuild.slack.com/archives/C014RARENH0/p1674838476782969
[WORKSPACE-managed repositories]: https://bazel.build/external/overview#workspace-system
[bazel_dep]: https://bazel.build/rules/lib/globals#bazel_dep
[bzlmod Selection.java]: https://cs.opensource.google/bazel/bazel/+/master:src/main/java/com/google/devtools/build/lib/bazel/bzlmod/Selection.java
[bzlmod Version.java]: https://cs.opensource.google/bazel/bazel/+/master:src/main/java/com/google/devtools/build/lib/bazel/bzlmod/Version.java
[bzlmod VersionTest.java]: https://cs.opensource.google/bazel/bazel/+/master:src/test/java/com/google/devtools/build/lib/bazel/bzlmod/VersionTest.java
[compatibility level]: https://bazel.build/external/module#compatibility_level
[currently supports]: https://github.com/renovatebot/renovate/tree/main/lib/modules/manager/bazel
[custom registry support]: https://github.com/renovatebot/renovate/blob/main/lib/modules/datasource/types.ts#L95-L98
[datasource in the renovate framework]: https://github.com/renovatebot/renovate/tree/main/lib/modules/datasource
[github-releases datasource]: https://github.com/renovatebot/renovate/blob/ffbf6e929d6af0b4910942027d09ab971ce43587/lib/modules/datasource/github-releases/index.ts
[multiple Bazel registries]: https://bazel.build/external/registry#selecting_registries
[registry strategy]: https://github.com/renovatebot/renovate/blob/main/lib/modules/datasource/types.ts#L87-L93
[relaxed SemVer specification]: https://bazel.build/external/module#version_format
[the existing Renovate versioning schemes]: https://github.com/renovatebot/renovate/tree/main/lib/modules/versioning
[the same Bazel module version sort]: https://cs.opensource.google/bazel/bazel/+/master:src/main/java/com/google/devtools/build/lib/bazel/bzlmod/Version.java
[the version documentation]: https://cs.opensource.google/bazel/bazel/+/master:src/main/java/com/google/devtools/build/lib/bazel/bzlmod/Version.java;l=34-37;bpv=0;bpt=1
[yanked versions]:https://bazel.build/external/module#yanked_versions
[Bazel module lockfiles]: https://docs.google.com/document/d/1HPeH_L-lRK54g8A27gv0q7cbk18nwJ-jOq_14XEiZdc/edit#heading=h.5mcn15i0e1ch
[manager in the renovate framework]: https://github.com/renovatebot/renovate/blob/main/docs/development/adding-a-package-manager.md
[`bazel` renovate manager]: https://github.com/renovatebot/renovate/blob/main/lib/modules/manager/bazel
[Starlark language parser]: https://github.com/zharinov/good-enough-parser/blob/main/lib/lang/starlark.ts
[bazel_dep]: https://bazel.build/rules/lib/globals/module#bazel_dep
[archive_override]: https://bazel.build/rules/lib/globals/module#archive_override
[git_override]: https://bazel.build/rules/lib/globals/module#git_override
[single_version_override]: https://bazel.build/rules/lib/globals/module#single_version_override
[multiple_version_override]: https://bazel.build/rules/lib/globals/module#multiple_version_override
[ManagerApi]: https://github.com/renovatebot/renovate/blob/ffbf6e929d6af0b4910942027d09ab971ce43587/lib/modules/manager/types.ts#L228-L267
[a document related to Bazel]: https://github.com/renovatebot/renovate/blob/main/docs/usage/bazel.md
