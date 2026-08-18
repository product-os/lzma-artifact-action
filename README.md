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
- uses: product-os/lzma-artifact-action/upload@v1
  with:
    source: path/to/build/**       # files, directories, or globs (required)
    encrypt: true                  # AES-256 encrypt (default: false)
    password: ${{ secrets.ARTIFACT_PASSWORD }}   # required when encrypt is true
    name: my-build                 # GitHub artifact name (default: artifact)
    compression-level: "3"         # 0-9 (default: 3)
```

Compression only (no encryption):

```yaml
- uses: product-os/lzma-artifact-action/upload@v1
  with:
    source: dist/
    name: my-build
```

`encrypt` is a plain boolean input, separate from `password`, so encryption can be toggled by a
workflow expression while the password secret stays permanently set — no need to conditionally
call the action:

```yaml
- uses: product-os/lzma-artifact-action/upload@v1
  with:
    source: dist/
    encrypt: ${{ github.ref == 'refs/heads/main' }}   # encrypt only on main
    password: ${{ secrets.ARTIFACT_PASSWORD }}         # always set; used only when encrypt is true
    name: my-build
```

Strip a leading path prefix with `base-directory` — `tar` runs from it, so the stored (and
therefore extracted) paths are relative to it. Here `build/out/image.json` is archived as
`image.json`, not `build/out/image.json`:

```yaml
- uses: product-os/lzma-artifact-action/upload@v1
  with:
    base-directory: build/out
    source: "*"                    # globs resolve inside base-directory
    name: my-build
```

#### Source patterns

`source` takes a whitespace- or newline-separated list of files, directories and glob patterns.
`globstar` is enabled, so `**` spans any number of directory levels:

```yaml
- uses: product-os/lzma-artifact-action/upload@v1
  with:
    source: |
      licenses/**
      cyclonedx-export/**/*.json
      build/tmp/deploy/images/*/*.img
    name: my-build
```

Individual patterns that match nothing are tolerated; `if-no-files-found` governs only the case
where the whole list matches nothing.

Two behaviours to be aware of:

- **A matched directory brings its whole tree.** `tar` recurses, so when a pattern matches a
  directory every file below it is archived, whether or not the pattern would have matched those
  files. `licenses/**` and `licenses` produce the same archive. This differs from
  `actions/upload-artifact`, which uploads only the files a pattern matched. Nested matches are
  pruned before `tar` runs, so a file is never stored twice.
- **Dot-prefixed names are skipped.** Bash globs do not match names beginning with `.`, so no
  pattern reaches `.hidden/x.json`. Name such a path literally, or point `source` at a directory
  that contains it.

### Download

```yaml
- uses: product-os/lzma-artifact-action/download@v1
  with:
    name: my-build
    password: ${{ secrets.ARTIFACT_PASSWORD }}   # required only if it was encrypted
    destination: ./restored        # default: the workspace root
```

Downloading from another workflow run or repository uses the upstream passthroughs:

```yaml
- uses: product-os/lzma-artifact-action/download@v1
  with:
    name: my-build
    repository: my-org/other-repo
    run-id: "1234567890"
    github-token: ${{ secrets.CROSS_REPO_TOKEN }}
    destination: ./restored
```

## Inputs

### `upload`

| Input               | Required | Default    | Description                                                                      |
| ------------------- | -------- | ---------- | -------------------------------------------------------------------------------- |
| `source`            | yes      | —          | Files, directories or globs to archive; `**` spans any depth (see above).        |
| `base-directory`    | no       | `""`       | Run `tar` from this directory so `source` globs and stored/extracted paths are relative to it (strips a leading prefix). Empty archives paths relative to the workspace. |
| `if-no-files-found` | no       | `error`    | When `source` matches nothing: `error`, `warn` (upload nothing), or `ignore`. Individual misses are always tolerated; governs only the all-empty case (like `upload-artifact`). |
| `follow-symlinks`   | no       | `true`     | Archive the target of a symlink instead of the link (`tar --dereference`), matching `upload-artifact`'s `follow-symbolic-links`. Set `false` to keep symlinks as-is. |
| `encrypt`           | no       | `false`    | AES-256 encrypt the archive (headers included); requires `password`.             |
| `password`          | no       | `""`       | Key material for encryption. Required when `encrypt` is true; ignored otherwise. |
| `compression-level` | no       | `3`        | 7-Zip LZMA level, 0 (store) to 9 (best).                                         |
| `name`              | no       | `artifact` | GitHub artifact name (`actions/upload-artifact`).                                |
| `retention-days`    | no       | `0`        | Artifact retention in days; 0 uses the repo default.                             |
| `overwrite`         | no       | `true`     | Overwrite an existing same-named artifact.                                       |

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
printf '%s\0' <pruned source paths> \
  | tar --create --file=- --null --files-from=- \
  | 7z a -si -t7z -mx=<level> [-mhe=on -p<password>] artifact.tar.7z

# download
7z x -so [-p<password>] artifact.tar.7z | tar --extract --file=- --directory=<destination>
```

The path list reaches `tar` on stdin rather than as arguments, because a `**` expansion over a
large tree exceeds `ARG_MAX`. Paths that already have an ancestor in the list are dropped first, so
`tar`'s own recursion cannot store the same file once per ancestor directory.

`7z`'s `-t7z` container uses LZMA2. When `encrypt` is true, `-p` adds AES-256 and `-mhe=on`
encrypts the archive headers/filenames too; when it is false those switches are omitted and you
get identical LZMA compression with no encryption. On download a supplied password is used only
if the archive is actually encrypted — 7-Zip ignores it on an unencrypted archive — so it is
safe to always pass the password from a secret.

## Decrypting an artifact manually

The artifact is a single `artifact.tar.7z` (LZMA, optionally AES-256). To recover its contents
outside a workflow, download it and reverse the pipeline yourself with `7z` and `tar`.

Download it with the GitHub CLI (this unwraps GitHub's outer zip for you):

```bash
gh run download <run-id> --name <artifact-name>
```

Or use the run's **Artifacts** section in the web UI, which gives you `<artifact-name>.zip`;
unzip that to get `artifact.tar.7z`. Then extract:

```bash
# encrypted (uploaded with encrypt: true)
7z x -so artifact.tar.7z -p'<password>' | tar --extract --file=-

# not encrypted
7z x -so artifact.tar.7z | tar --extract --file=-
```

This extracts into the current directory. Requires `7z` (7-Zip; `7za` or `7zz` work too) and
`tar`. If you omit `-p` on an encrypted archive, 7-Zip prompts for the password interactively.

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

_Co-authored with Claude_
