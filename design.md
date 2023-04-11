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


## Design

### Versioning

Versioning for Bazel modules has two components: the `version` string and the `compatibility_level`
integer. The `version` string has two roles. First, it identifies a unique release for a Bazel
module. Second, it acts as a sort field aiding in the ordering of releases. The
`compatibility_level` groups releases identifying those which are compatible with each other.
Together, these values dictate how to identify upgrade candidates for a Bazel module.

#### Version Formats

Bazel modules support a [relaxed SemVer specification]. Examples of valid `version` values are
listed below:

```sh
1.2.3                 # SemVer
7.0.0-pre.20230330.3  # SemVer with suffix
foo_1.2.3             # Non-digit characters in SemVer
20230125.2            # Date-patch format
```

The one constant in the formats is that the period (`.`) separates the parts of a version value. For
the purposes of sorting, this will be important as the sort will occur by comparing the parts of the
version values.

#### Version Sorting

To identify whether a release is an upgrade candidate for another release, we need to sort the
version values. This will be done by splitting version values into parts identified by period (`.`)
characters. Each part will be sorted based upon the type of values found in that part across the
entire collection of version values. A part that only contains digits (`[0-9]`) will be sorted by
number. Otherwise, the part will be sorted in alphanumeric order.

Let's look at an example. The following represents an unordered collection of version values for a
Bazel module.

```sh
# Unordered list of version values
6.2.0
7.0.0-pre.20230330.3
6.3.0
6.2.1
```

We need to sort this list such that the result is:

```sh
# Ordered list of version values
6.2.0
6.2.1
6.3.0
7.0.0-pre.20230330.3
```

First, we need to identify the maximum number of parts for a version value across the entire
collection. In this case, the maximum number of parts is five as found in `7.0.0-pre.20230330.3`.

Next, we need to identify the value type for each part.

| Part | Type | Values |
| ---- | ---- | ------ |
| 0 | numeric | `6, 7` |
| 1 | numeric | `0, 2, 3` |
| 2 | alphanumeric | `0, 1, 0-pre` |
| 


#### Version Selection

#### Upgrarde Logic: Library vs Executable

[Related Slack discussion](https://bazelbuild.slack.com/archives/C014RARENH0/p1674838476782969)

<!-- Future Sections

### New Module Version Detection

## Implementation Details

### Renovate Versioning: `bazel_module`

### Renovate Datasource: `bazel_module_registry`

### Renovate Package Manager: `bazel_module`

-->

<!-- LINKS -->

[--registry]: https://bazel.build/reference/command-line-reference#flag--registry
[Bazel Central Registry]: https://github.com/bazelbuild/bazel-central-registry
[Bazel module version formats]: https://bazel.build/external/module#version_format
[Bazel modules]: https://bazel.build/external/module
[Multiple version override]: https://bazel.build/external/module#multiple-version_override
[Non-registry overrides]: https://bazel.build/external/module#non-registry_overrides
[Renovate]: https://github.com/renovatebot/renovate
[Single version override]: https://bazel.build/external/module#single-version_override
[WORKSPACE-managed repositories]: https://bazel.build/external/overview#workspace-system
[bazel_dep]: https://bazel.build/rules/lib/globals#bazel_dep
[compatibility level]: https://bazel.build/external/module#compatibility_level
[currently supports]: https://github.com/renovatebot/renovate/tree/main/lib/modules/manager/bazel
[yanked versions]:https://bazel.build/external/module#yanked_versions
[relaxed SemVer specification]: https://bazel.build/external/module#version_format
