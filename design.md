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

- Support extracting external dependency information from `MODULE.bazel` files.
  - Read external dependency information from [bazel_dep] declarations.
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
- Support module overrides.
  - [Single version override]
  - [Multiple version override]
  - [Non-registry overrides]

## Design

The following sections describe 

### Versioning

#### Parsing and Sorting

#### Resolution: Library vs Executable

[Related Slack discussion](https://bazelbuild.slack.com/archives/C014RARENH0/p1674838476782969)


<!-- Future Sections

### New Module Version Detection

## Implementation Details

### Renovate Versioning: `bazel_module`

### Renovate Datasource: `bazel_module_registry`

### Renovate Package Manager: `bazel_module`

-->

<!-- LINKS -->

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
