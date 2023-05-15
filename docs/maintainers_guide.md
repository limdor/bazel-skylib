# Skylib Maintainer's Guide

## The Parts of Skylib

*   `bzl_library.bzl` - used by almost all rule sets, and thus requiring
    especial attention to maintaining backwards compatibility. Ideally, it ought
    to be moved out of Skylib and and into Bazel's bundled `@bazel_tools` repo
    (see https://github.com/bazelbuild/bazel-skylib/issues/127).
*   Test libraries - `rules/analysis_test.bzl`, `rules/build_test.bzl`,
    `lib/unittest.bzl`; these are under more active development than the rest of
    Skylib, because we want to provide rule authors with a good testing story.
    Ideally, these ought to be moved out of Skylib and evolved at a faster pace.
*   A kitchen sink of utility modules (everything else). Formerly, these
    features were piled on in a rather haphazard manner. For any new additions,
    we want to be more conservative: add a feature only if it is widely needed
    (or was already independently implemented in multiple rule sets), if the
    interface is unimpeachable, if level of abstraction is not shallow, and the
    implementation is efficient.

## PR Review Standards

Because Skylib is so widely used, breaking backwards compatibility can cause
widespread pain, and shouldn't be done lightly. Therefore:

1.  In the first place, avoid adding insufficiently thought out, insufficiently
    tested features which will later need to be replaced in a
    backwards-incompatible manner. See the criteria in README.md.
2.  Given a choice between breaking backwards compatibility and keeping it, try
    to keep backwards compatibility. For example, if adding a new argument to a
    function, add it to the end of the argument list, so that existing callers'
    positional arguments continue to work.
3.  Keep Skylib out-of-the-box compatible with the current stable Bazel release
    (ideally - with two most recent stable releases).
    *   For example, when adding a new function which calls the new
        `native.foobar()` method which was introduced in the latest Bazel
        pre-release or is gated behind an `--incompatible` flag, use an `if
        hasattr(native, "foobar")` check to keep the rest of your module (which
        doesn't need `native.foobar()`) working even when `native.foobar()` is
        not available.

In addition, make sure that new code is documented and tested.

If a PR changes any docstring in an existing module, the corresponding
`stardoc_with_diff_test` in `docs` will fail. To fix the test, ask the PR
author to run `bazel run //docs:update`.

If a PR adds a new module, make sure that the PR also adds a corresponding
`stardoc_with_diff_test` target in `docs/BUILD` and a corresponding `*doc.md`
file under `docs` (generated by `bazel run //docs:update`).

## Making a New Release

1.  Update CHANGELOG.md at the top. You may want to use the following template:

--------------------------------------------------------------------------------

Release $VERSION

**New Features**

-   Feature
-   Feature

**Incompatible Changes**

-   Change
-   Change

**Contributors**

Name 1, Name 2, Name 3 (alphabetically from `git log`)

--------------------------------------------------------------------------------

2.  Bump `version` in version.bzl, MODULE.bazel, *and* gazelle/MODULE.bazel to
    the new version.
    TODO(#386): add a test to make sure all the versions are in sync.
3.  Ensure that the commits for steps 1 and 2 have been merged. All further
    steps must be performed on a single, known-good git commit.
4.  `bazel build //distribution`
5.  Copy the `bazel-skylib-$VERSION.tar.gz` and
    `bazel-skylib-gazelle-plugin-$VERSION.tar.gz` tarballs to the mirror (you'll
    need Bazel developer gcloud credentials; assuming you are a Bazel developer,
    you can obtain them via `gcloud init`):

```bash
gsutil cp bazel-bin/distribution/bazel-skylib{,-gazelle-plugin}-$VERSION.tar.gz gs://bazel-mirror/github.com/bazelbuild/bazel-skylib/releases/download/$VERSION/
gsutil setmeta -h "Cache-Control: public, max-age=31536000" gs://bazel-mirror/github.com/bazelbuild/bazel-skylib/releases/download/$VERSION/bazel-skylib{,-gazelle-plugin}-$VERSION.tar.gz
```

6.  Obtain checksums for release notes:

```bash
sha256sum bazel-bin/distribution/bazel-skylib-$VERSION.tar.gz
sha256sum bazel-bin/distribution/bazel-skylib-gazelle-plugin-$VERSION.tar.gz
````

7.  Draft a new release with a new tag named $VERSION in github. Attach
    `bazel-skylib-$VERSION.tar.gz` and
    `bazel-skylib-gazelle-plugin-$VERSION.tar.gz` to the release. For the
    release notes, use the CHANGELOG.md entry plus the following template:

--------------------------------------------------------------------------------

**WORKSPACE setup**

```
load("@bazel_tools//tools/build_defs/repo:http.bzl", "http_archive")

http_archive(
    name = "bazel_skylib",
    sha256 = "$SHA256SUM"
    urls = [
        "https://mirror.bazel.build/github.com/bazelbuild/bazel-skylib/releases/download/$VERSION/bazel-skylib-$VERSION.tar.gz",
        "https://github.com/bazelbuild/bazel-skylib/releases/download/$VERSION/bazel-skylib-$VERSION.tar.gz",
    ],
)

load("@bazel_skylib//:workspace.bzl", "bazel_skylib_workspace")

bazel_skylib_workspace()
```

***Additional WORKSPACE setup for the Gazelle plugin***

```starlark
http_archive(
    name = "bazel_skylib_gazelle_plugin",
    sha256 = "$SHA256SUM_GAZELLE_PLUGIN",
    urls = [
        "https://mirror.bazel.build/github.com/bazelbuild/bazel-skylib/releases/download/$VERSION/bazel-skylib-gazelle-plugin-$VERSION.tar.gz",
        "https://github.com/bazelbuild/bazel-skylib/releases/download/$VERSION/bazel-skylib-gazelle-plugin-$VERSION.tar.gz",
    ],
)

load("@bazel_skylib_gazelle_plugin//:workspace.bzl", "bazel_skylib_gazelle_plugin_workspace")

bazel_skylib_gazelle_plugin_workspace()

load("@bazel_skylib_gazelle_plugin//:setup.bzl", "bazel_skylib_gazelle_plugin_setup")

bazel_skylib_gazelle_plugin_setup()
```

**Using the rules**

See [the source](https://github.com/bazelbuild/bazel-skylib/tree/$VERSION).

--------------------------------------------------------------------------------

8.  Obtain [Subresource Integrity](https://w3c.github.io/webappsec-subresource-integrity/#integrity-metadata-description)
    format checksums for bzlmod:

```bash
echo -n sha256-; cat bazel-bin/distribution/bazel-skylib-$VERSION.tar.gz | openssl dgst -sha256 -binary | base64
echo -n sha256-; cat bazel-bin/distribution/bazel-skylib-gazelle-plugin-$VERSION.tar.gz | openssl dgst -sha256 -binary | base64
```

9.  Create a PR at [Bazel Central Registry](https://github.com/bazelbuild/bazel-central-registry)
    to update the registry's versions of bazel_skylib and
    bazel_skylib_gazelle_plugin.

    Use https://github.com/bazelbuild/bazel-central-registry/pull/403 as the
    model; you will need to update `modules/bazel_skylib/metadata.json` and
    `modules/bazel_skylib_gazelle_plugin/metadata.json` to list the new version
    in `versions`, and create new $VERSION subdirectories for the updated
    modules, using the latest existing version subdirectories as the guide. Use
    Subresource Integrity checksums obtained above in the new `source.json`
    files.

    Ensure that the MODULE.bazel files you add in the new $VERSION
    subdirectories exactly match the MODULE.bazel file packaged in
    bazel-skylib-$VERSION.tar.gz and bazel-skylib-gazelle-plugin-$VERSION.tar.gz
    tarballs - or buildkite checks will fail.

10. Once the Bazel Central Registry PR is merged, insert in the release
    description after the WORKSPACE setup section:

--------------------------------------------------------------------------------

**MODULE.bazel setup**

```starlark
bazel_dep(name = "bazel_skylib", version = "$VERSION")
```

And for the Gazelle plugin:

```starlark
bazel_dep(name = "bazel_skylib_gazelle_plugin", version = "$VERSION", dev_dependency = True)
```

--------------------------------------------------------------------------------