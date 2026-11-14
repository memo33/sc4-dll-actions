# sc4-dll-actions

*Attested SC4 DLL builds*

This GitHub Actions workflow compiles a DLL for SC4 and publishes a [provenance attestation](https://docs.github.com/en/actions/concepts/security/artifact-attestations) for it. For end users, the attestation provides verifiable evidence that:

- the DLL they downloaded was actually built from the source in its repository
- the build ran through a repeatable and reviewed workflow
- the artifact is tied to a specific repository, commit, and workflow run
- users can independently verify it without trusting a developer claim alone

This helps establish transparency on how a DLL was built, as it gives end users an auditable trail from artifact to source.

See the reusable workflow at [.github/workflows/attested-build.yaml](.github/workflows/attested-build.yaml).

## Verify a released artifact

Anybody can independently verify a downloaded ZIP or DLL with [gh attestation verify](https://cli.github.com/manual/gh_attestation_verify):

```sh
gh attestation verify FILENAME --repo DLLOWNER/DLLREPOSITORY --deny-self-hosted-runners --signer-repo memo33/sc4-dll-actions --signer-digest <sha>
```

> [!Tip]
> Alternatively, the attestations for a project can be viewed online under `https://github.com/DLLOWNER/DLLREPO/attestations`.

A successful verification means the artifact was built with GitHub Actions using the workflow displayed by the command.
By inspecting the build steps in the workflow, this allows confirming that the DLL was built from the published source code.

Note that it does not mean that using the DLL is secure, but only that it was built transparently from the source code, which can be reviewed openly.

## Use the workflow

Create a workflow in your own repository that calls the reusable workflow:

```yaml
# .github/workflows/build.yaml
uses: memo33/sc4-dll-actions/.github/workflows/attested-build.yaml@<sha>
```

Preferably, reference the reusable workflow by a full commit `<sha>`. This pins the workflow to a persistent revision that you can review. Obtain the latest revision from the [v#-branch](https://github.com/memo33/sc4-dll-actions/branches/all).

### Example: Windows with CMake

```yaml
name: CI/CD
on:
  push:
  pull_request:
jobs:
  build:
    uses: memo33/sc4-dll-actions/.github/workflows/attested-build.yaml@<sha>
    with:
      build-os: windows-2022
      install-cmake: true
      build-script: |
        cmake -S . -B build -A Win32
        cmake --build build --config Release
        mkdir build-output
        Copy-Item build\Release\MyDll.dll MyDll.ini README.md LICENCSE.txt -Destination build-output
      build-output: build-output
      artifact-name: MyDll-${{ github.ref_type == 'tag' && github.ref_name || github.sha }}.zip
      attestation-subject-path: MyDll.dll
    permissions:
      id-token: write                # for attestation and deployment
      attestations: write            # for attestation
      artifact-metadata: write       # for attestation
      contents: write                # for deploying a release
```

### Example: Windows with MSBuild

```yaml
name: CI/CD
on:
  push:
  pull_request:
jobs:
  build:
    uses: memo33/sc4-dll-actions/.github/workflows/attested-build.yaml@<sha>
    with:
      build-os: windows-2022
      install-msbuild: true
      build-script: |
        msbuild MyProject.sln /p:Configuration=Release /p:Platform=Win32
        mkdir build-output
        Copy-Item MyProject\bin\Release\MyDll.dll MyDll.ini README.md LICENCSE.txt -Destination build-output
      build-output: build-output
      artifact-name: MyDll-${{ github.ref_type == 'tag' && github.ref_name || github.sha }}.zip
      attestation-subject-path: MyDll.dll
    permissions:
      id-token: write                # for attestation and deployment
      attestations: write            # for attestation
      artifact-metadata: write       # for attestation
      contents: write                # for deploying a release
```

### Example: Linux with Clang and xwin

```yaml
jobs:
  build:
    uses: memo33/sc4-dll-actions/.github/workflows/attested-build.yaml@<sha>
    with:
      build-os: ubuntu-26.04
      xwin-path: vendor/xwin
      build-script: |
        make CLANG=clang-cl-22
        mkdir build-output
        cp -p MyDll.{dll,ini} README.md 'Third Party Notices.txt' LICENSE.txt build-output
      build-output: build-output
      artifact-name: MyDll-${{ github.ref_type == 'tag' && github.ref_name || github.sha }}.zip
      attestation-subject-path: MyDll.dll
    permissions:
      id-token: write                # for attestation and deployment
      attestations: write            # for attestation
      artifact-metadata: write       # for attestation
      contents: write                # for deploying a release
```

Clang comes preinstalled on Linux and Windows runners.
On Linux, `clang-cl` is not available on the PATH though as there are multiple versions, so explicitly select a specific major version such as `clang-cl-22`.
For available LLVM/Clang versions, check the runner documentation, e.g.
[ubuntu-26.04](https://github.com/actions/runner-images/blob/main/images/ubuntu/Ubuntu2604-Readme.md),
[windows-2022](https://github.com/actions/runner-images/blob/main/images/windows/Windows2022-Readme.md).
The specific runner image version and included software versions are logged at the top of each job in every workflow run.

## Inputs

| Input | Default | Description |
|---|:---:|---|
| `build-script` | — | Command or inline script that builds the project. Runs in `bash` or `pwsh`, according to `build-shell` and the runner OS. |
| `build-output` | — | Path to the file or directory to package. |
| `artifact-name` | — | Name of the ZIP artifact and release file. It should end in `.zip`. |
| `attestation-subject-path` | — | Path or glob for the DLL inside the ZIP, e.g. `*.dll`. Matching files are added as attestation subjects in addition to the ZIP. Set it to an empty string to attest only the ZIP.<br>It should match only the intended DLL; it MUST NOT match any third-party binaries and SHOULD NOT match any text files, images, etc. |
| `build-os` | `ubuntu-26.04` | Runner OS label for the build, such as `ubuntu-26.04` or `windows-2022`. |
| `install-msbuild` | `false` | Whether to install MSBuild. Intended for Windows builds. |
| `install-cmake` | `false` | Whether to install CMake. |
| `xwin-path` | Empty | Optional path for Windows SDK headers, passed to `setup-xwin` on Linux. An empty value skips xwin setup. |
| `build-shell` | `pwsh` on Windows; else `bash` | Shell used to run `build-script`: `bash` or `pwsh`. |
| `skip-sign` | `false` | When `true`, skips the sign job, e.g. on pre-releases. Set this for all tags without release. For example:<br>`${{ contains(github.ref_name, '-') }}` |
| `skip-deploy` | `false` | When `true`, skips the deploy job; for example, if you want to handle deployment yourself. |
| `prerelease` | `false` | When `true`, the release created on deployment is a *prerelease*. For example:<br>`${{ contains(github.ref_name, '-') }}` |

Inputs marked with — are required.

## Outputs

These are reusable-workflow outputs, available to the calling workflow through `needs.<job-id>.outputs.<output-name>`.

| Output | Description |
|---|---|
| `artifact-id` | ID of the uploaded Actions artifact after the signing job. |
| `artifact-sha256` | SHA-256 digest of the final ZIP artifact. |
| `attestation-url` | URL of the artifact's provenance attestation. |

Signing and deployment run only for tagged builds. Consequently, these outputs are empty when the caller runs on a branch or another non-tag ref.
