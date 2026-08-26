# OpenUSD Release Versions

Copyright &copy; 2026, Pixar Animation Studios, version 1.0

## Background

OpenUSD releases are commonly described using a [CalVer](https://calver.org/) scheme,
like "v26.08". However, OpenUSD internally follows a [SemVer](https://semver.org/)
scheme using major, minor, and patch version numbers and publishes these values
through its C++ and Python API. For example, for the "v26.08" release, the values
available through the API are:

| C++ API           | Value |
| ----------------- | ----- |
| PXR_MAJOR_VERSION | 0     |
| PXR_MINOR_VERSION | 26    |
| PXR_PATCH_VERSION | 8     |
| PXR_VERSION       | 2608  |

| Python API        | Value      |
| ----------------- | ---------- |
| Usd.GetVersion()  | (0, 26, 8) |

Per SemVer, the major version has been set to 0 since OpenUSD's initial open
source release to reflect its current lack of API stability guarantees across
releases. Because of this, the major version is typically ignored when spelling
OpenUSD versions outside of code. For example, the GitHub repository and
CHANGELOG.md spell versions numbers as "v26.08" instead of "v0.26.08". Note
also that the `usd-core` PyPI package uses version numbers like "26.8".

A significant drawback to this scheme is that there is no way to identify
an OpenUSD patch release like "v25.05.01" in code. This has not been reported
as a major issue, likely because patch releases are infrequent: there have been
only 5 out of 52 total releases in OpenUSD's history. However, given
continued interest in security fixes for prior releases and long-term
support branches, we anticipate the number of patch releases will grow,
making it more important to be able to identify such releases in code.

## Proposal

To address this issue, we propose introducing a fourth version number to
OpenUSD's version API, called the "tweak" version. This version would start
at 0 for a regular release, then be incremented by 1 for each subsequent patch
release. The "tweak" name is adopted from the version specification used by
[CMake's `project` function](https://cmake.org/cmake/help/latest/command/project.html#command:project).
This may be convenient for other projects that also use CMake and want to
specify a version (or versions) of OpenUSD to use, although that convenience
is not an explicit goal.

Under this proposal, the C++ API would gain a new `PXR_TWEAK_VERSION`
macro containing the tweak version, as well as a `PXR_FULL_VERSION` macro
that combines all four version numbers in a single value. The existing
`PXR_VERSION` macro would be retained for backwards compatibility and
continue to work as it does today, but clients would need to switch to
using the `PXR_FULL_VERSION` macro to distinguish between patch releases.

In Python, the `Usd.GetVersion()` function will be updated to return a
tuple containing four values instead of three. Existing code that compares
the result of this function to a tuple will continue to work as expected.

The tweak version would be represented using three digits in contexts
where zero-padding is significant. This allows us to represent up to 999
patch releases. We do not expect to publish anywhere near this many
releases. Three digits should be more than sufficient without being
inconvenient or annoying to type out in contexts where those digits are
needed. For example, the first patch release for the "v26.08" release
would be tagged in the OpenUSD repository as "v26.08.001" and would be
referred to in documentation and the CHANGELOG as such.

Summarizing the new version API, for "v26.08.001":

| C++ API           | Value    |
| ----------------- | -------- |
| PXR_MAJOR_VERSION | 0        |
| PXR_MINOR_VERSION | 26       |
| PXR_PATCH_VERSION | 8        |
| PXR_TWEAK_VERSION | 1        |
| PXR_VERSION       | 2608     |
| PXR_FULL_VERSION  | 2608001  |

| Python API        | Value         |
| ----------------- | ------------- |
| Usd.GetVersion()  | (0, 26, 8, 1) |

This approach is the minimal change needed to represent patch versions in
code and is fully backwards compatible with existing code. It preserves the
ability per SemVer to communicate API stability guarantees in the version
number. Users today would continue to see the major version as 0 and interpret
that as "no guarantee", but that number could be increased in the future if
OpenUSD ever did provide some guarantee.

This approach does not address the inconsistency between the CalVer spelling
of versions outside of code and the SemVer version numbers in code. This hasn't
ever been raised as an issue, but one way this could show up in the future is if
OpenUSD did bump the major version to a non-zero value. For example, if the
27.02 release bumped the major version to 1, this would be represented in code
as `(major=1, minor=27, patch=2, tweak=0)`. How would this be spelled outside of
code? "v1.27.02" is the most obvious answer, but this would incorrectly be
considered an earlier version than the "v26.08" release. This would also impact
the `usd-core` PyPI package: "usd-core-1.27.2" would be considered newer than
"usd-core-26.8". If we wound up using CalVer and spelling this version as
"v27.02" instead, then we lose the primary benefit of SemVer.

## Alternatives

Alternatively, we could adjust OpenUSD's code to fully embrace CalVer. This is
appealing because it's what users are familiar with already. It would remove
confusion about versioning scheme OpenUSD uses and, for better or worse, signify
that OpenUSD's version number does not convey any meaning beyond chronological
order. If in the future there were releases with API stability guarantees or
other semantics, those would need to be explicitly documented somewhere (which
perhaps should be done regardless of which proposal is implemented).

There were two variations of this considered:

#### Introduce new API for the CalVer version numbers and deprecate the old API.

For example, for the "v26.08.001" release, the new version API would be:

| C++ API           | Value    |
| ----------------- | -------- |
| PXR_VERSION_YEAR  | 26       |
| PXR_VERSION_MONTH | 8        |
| PXR_VERSION_PATCH | 1        |
| PXR_FULL_VERSION  | 2608001  |
| **Deprecated**    |          |
| PXR_MAJOR_VERSION | 0        |
| PXR_MINOR_VERSION | 26       |
| PXR_PATCH_VERSION | 8        |
| PXR_VERSION       | 2608     |

| Python API           | Value      |
| -------------------- | ---------- |
| Usd.GetFullVersion() | (26, 8, 1) |
| **Deprecated**       |            |
| Usd.GetVersion()     | (0, 26, 8) |

Under this approach, the deprecated API would continue to be updated and work
as it does today. Although the deprecated API would likely be kept around for
a long period for backwards compatibility, it would eventually be removed and
require clients to update their code. During this period users might still be
confused about whether OpenUSD follows CalVer or SemVer. Adding explicit
documentation about OpenUSD's versioning scheme could help with that.

#### Change the major and minor version numbers to the year and month.

For example, for the "v26.08.001" release the version API would be:

| C++ API           | Value    |
| ----------------- | -------- |
| PXR_MAJOR_VERSION | 26       |
| PXR_MINOR_VERSION | 8        |
| PXR_PATCH_VERSION | 1        |
| PXR_VERSION       | 2608     |
| PXR_FULL_VERSION  | 2608001  |

| Python API        | Value      |
| ----------------- | ---------- |
| Usd.GetVersion()  | (26, 8, 1) |

This approach would allow us to encode patch versions without introducing
new, potentially confusing, API. However, even though these values were
intended to encode a CalVer version, the use of major/minor/patch could
confuse users into thinking these values represented a SemVer version
and interpret the major version greater than 0 as an API stability guarantee.
