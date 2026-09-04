# ci-actions

Reusable GitHub Actions shared across the Cucumber C++ projects.

## `install-cpp-dep`

Checks out a Cucumber C++ sub-project at a given ref, builds it with a CMake
workflow preset, and installs it so downstream projects find it via
`find_package()`. This replaces the copy-pasted "Checkout + Build and install
cucumber_*" step pairs currently duplicated across every `test-cpp.yaml` job.

### Inputs

| Input        | Required | Default        | Description                                             |
| ------------ | -------- | -------------- | ------------------------------------------------------- |
| `repository` | yes      | —              | `owner/name` to build (e.g. `cucumber/messages`).       |
| `ref`        | yes      | —              | Git ref (tag, branch, or SHA) to check out and build.   |
| `path`       | no       | `<name>-src`   | Checkout directory. Defaults to the repo name + `-src`. |
| `preset`     | no       | `host-system`  | CMake workflow preset used to build the dependency.     |

### Usage

A downstream workflow references it by `owner/repo@ref`. For example, `query`'s
`test-cpp.yaml` only needs to declare its direct dependency — `messages`:

```yaml
env:
  MESSAGES_REF: e85a9eaf3fa3787c25bc62b6ea67672e709ee4fa

steps:
  - uses: actions/checkout@v7
    with:
      persist-credentials: false

  # ... install system packages (nlohmann-json, gtest, ...) ...

  - name: Install cucumber_messages
    uses: cucumber/ci-actions/install-cpp-dep@v1
    with:
      repository: cucumber/messages
      ref: ${{ env.MESSAGES_REF }}

  - name: Build and unit test
    working-directory: cpp
    run: cmake --workflow --preset test-system
```

`pretty-formatter`, which depends on `query` (and transitively `messages`),
would add one call per dependency it builds:

```yaml
  - name: Install cucumber_messages
    uses: cucumber/ci-actions/install-cpp-dep@v1
    with:
      repository: cucumber/messages
      ref: ${{ env.MESSAGES_REF }}

  - name: Install cucumber_query
    uses: cucumber/ci-actions/install-cpp-dep@v1
    with:
      repository: cucumber/query
      ref: ${{ env.QUERY_REF }}
```

## Keeping the pinned refs up to date with Renovate

The `*_REF` values are plain git commits, which Renovate does not track out of
the box. Two pieces are needed:

1. **An annotation comment** on each ref in the workflow, declaring the
   datasource, the repository, and the branch to follow:

   ```yaml
   env:
     # renovate: datasource=git-refs depName=cucumber/messages packageName=https://github.com/cucumber/messages currentValue=main
     MESSAGES_REF: e85a9eaf3fa3787c25bc62b6ea67672e709ee4fa
   ```

   `git-refs` + `currentValue=main` tells Renovate to bump the pinned commit to
   the latest `main` (use a tag name here and `datasource=github-tags` to track
   releases instead).

2. **A custom manager** in the repo's `.github/renovate.json` that reads those
   annotations (add it once per repo, or centrally in
   `cucumber/renovate-config`):

   ```json
   {
     "customManagers": [
       {
         "customType": "regex",
         "managerFilePatterns": ["/^\\.github/workflows/.+\\.ya?ml$/"],
         "matchStrings": [
           "# renovate: datasource=(?<datasource>\\S+) depName=(?<depName>\\S+) packageName=(?<packageName>\\S+) currentValue=(?<currentValue>\\S+)\\s+\\w+_REF:\\s*(?<currentDigest>[0-9a-f]{7,40})"
         ]
       }
     ]
   }
   ```

The `actions/checkout` pin inside `install-cpp-dep/action.yml` is handled
automatically by Renovate's built-in `github-actions` manager — no annotation
required.
