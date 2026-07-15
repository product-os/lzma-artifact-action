# lzma-artifact-action

Two GitHub composite actions for LZMA-compressed — and optionally AES-256 encrypted — build
artifacts, in one repository:

- **`upload`** — `tar` + `7z` (LZMA2) compress your files into a single `artifact.tar.7z` and
  upload it with [`actions/upload-artifact`][upload-artifact].
- **`download`** — download that artifact with [`actions/download-artifact`][download-artifact],
  then decompress (and decrypt, if a password was used).

Both are thin wrappers: they add only the compression/encryption layer and pass the upstream
actions' inputs straight through, so you keep upstream features and — with Renovate — stay on
current upstream versions.

## Requirements

- **Linux runner.** The actions use `tar` and `7z`, both present on GitHub-hosted
  `ubuntu-latest`. `7z` is not guaranteed on macOS runners. On self-hosted runners install
  `7z`/`7za` (e.g. `p7zip-full`).

## Usage

### Upload

```yaml
- uses: balena-io-experimental/lzma-artifact-action/upload@v1
  with:
    source: path/to/build/*        # file, directory, or glob (required)
    password: ${{ secrets.ARTIFACT_PASSWORD }}   # optional — omit for compression only
    name: my-build                 # GitHub artifact name (default: artifact)
    compression-level: "3"         # 0-9 (default: 3)
```

Compression only (no password):

```yaml
- uses: balena-io-experimental/lzma-artifact-action/upload@v1
  with:
    source: dist/
    name: my-build
```

### Download

```yaml
- uses: balena-io-experimental/lzma-artifact-action/download@v1
  with:
    name: my-build
    password: ${{ secrets.ARTIFACT_PASSWORD }}   # required only if it was encrypted
    destination: ./restored        # default: the workspace root
```

Downloading from another workflow run or repository uses the upstream passthroughs:

```yaml
- uses: balena-io-experimental/lzma-artifact-action/download@v1
  with:
    name: my-build
    repository: my-org/other-repo
    run-id: "1234567890"
    github-token: ${{ secrets.CROSS_REPO_TOKEN }}
    destination: ./restored
```

## Inputs

### `upload`

| Input               | Required | Default    | Description                                                        |
| ------------------- | -------- | ---------- | ------------------------------------------------------------------ |
| `source`            | yes      | —          | File, directory, or glob describing what to archive.               |
| `password`          | no       | `""`       | When set, the archive is AES-256 encrypted (headers included).     |
| `compression-level` | no       | `3`        | 7-Zip LZMA level, 0 (store) to 9 (best).                           |
| `name`              | no       | `artifact` | GitHub artifact name (`actions/upload-artifact`).                  |
| `retention-days`    | no       | `0`        | Artifact retention in days; 0 uses the repo default.               |
| `overwrite`         | no       | `true`     | Overwrite an existing same-named artifact.                         |

**Outputs:** `artifact-id`, `artifact-url` (from `actions/upload-artifact`).

### `download`

| Input            | Required | Default                      | Description                                                    |
| ---------------- | -------- | ---------------------------- | -------------------------------------------------------------- |
| `destination`    | no       | `${{ github.workspace }}`    | Where the decompressed contents are extracted.                 |
| `password`       | no       | `""`                         | Required only if the artifact was encrypted.                   |
| `name`           | no       | `""`                         | Artifact name; empty downloads all artifacts.                  |
| `pattern`        | no       | `""`                         | Glob of artifact names to download (ignored if `name` is set). |
| `merge-multiple` | no       | `false`                      | Layout when extracting multiple artifacts (see below).         |
| `github-token`   | no       | `${{ github.token }}`        | Token for cross-run / cross-repo downloads.                    |
| `repository`     | no       | `${{ github.repository }}`   | Repository to download from.                                   |
| `run-id`         | no       | `${{ github.run_id }}`       | Workflow run the artifact belongs to.                          |

> **Multiple artifacts:** give them distinct `name`s on upload and download them with `pattern`.
> `merge-multiple` mirrors `actions/download-artifact`, but is applied when extracting (the
> compressed archives all share the name `artifact.tar.7z`, so they can't be merged before
> extraction): `false` (default) extracts each artifact's contents into its own
> `<destination>/<artifact-name>/` subdirectory; `true` extracts them all into `destination`. A
> single artifact is always extracted flat into `destination`.

## How it works

```text
# upload
tar --create --file=- <source> | 7z a -si -t7z -mx=<level> [-mhe=on -p<password>] artifact.tar.7z

# download
7z x -so [-p<password>] artifact.tar.7z | tar --extract --file=- --directory=<destination>
```

`7z`'s `-t7z` container uses LZMA2; `-p` adds AES-256 and `-mhe=on` encrypts the archive
headers/filenames too. Without a password those two switches are dropped and you get identical
LZMA compression with no encryption.

## Security

The password is passed to the actions via environment variables (not interpolated into the
rendered command) and never echoed to the log. One residual limitation: `7z` has no non-argv
password channel, so the password is briefly visible in the `7z` process's command-line
arguments on the runner. On single-tenant, ephemeral GitHub-hosted runners this is low risk. On
shared or self-hosted runners where other processes could read `/proc/*/cmdline`, treat it as a
consideration. Always supply the password from `secrets`, never a literal.

## Acknowledgements

Modelled on Alexandros Gidarakos's [`upload-encrypted-artifact`][upstream-upload] and
[`download-encrypted-artifact`][download-encrypted] (MIT).

[upload-artifact]: https://github.com/actions/upload-artifact
[download-artifact]: https://github.com/actions/download-artifact
[upstream-upload]: https://github.com/AlexGidarakos/upload-encrypted-artifact
[download-encrypted]: https://github.com/AlexGidarakos/download-encrypted-artifact

---

Co-authored with Claude
