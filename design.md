# Bazel Module (bzlmod) Support in Renovate

```yaml
author: Chuck Grindel<chuck.grindel@gmail.com>
status: To be reviewed
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
  - Support different version resolution logic for libraries (i.e., is depended upon by other
    projects) vs executable (i.e., is not depdended upon by other projects) repositories .
- Bazel Compatibility
  - Identify the Bazel version being used:
    1. Support parsing of `.bazelversion` to detect the Bazel version.
    2. Allow the Bazel version to be specified in a Renovate configuration value.
    3. No Bazel version detected
  - If a Bazel version is present, evaluate the `bazel_compatibility` expressions for the module
    version to determine if it is an acceptable upgrade candidate.

<!-- Future Sections

## Design

The following sections describe 

### Versioning

#### Parsing and Sorting

#### Resolution: Library vs Executable

[Related Slack discussion](https://bazelbuild.slack.com/archives/C014RARENH0/p1674838476782969)

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
