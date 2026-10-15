# yara

Container images with [YARA](https://virustotal.github.io/yara/), the pattern matching tool that malware analysts use to identify and classify files. A YARA rule describes a family of files by text, hex and regular expression strings and a condition, and `yara` reports which rules match each file it scans. `yarac` compiles rule files ahead of time. YARA is compiled from the release tag with the `hash`, `magic`, `cuckoo`, `dotnet`, `macho` and `dex` modules. The default image also contains 7-Zip, for unpacking the encrypted archives that samples are usually shared in. The images are rebuilt when YARA tags a release and when the base image changes, for `linux/amd64` and `linux/arm64`.

Upstream has put YARA in maintenance mode: it still gets bug and security fixes, and new development happens in [YARA-X](https://virustotal.github.io/yara-x/), its successor, which is a separate program and not part of these images.

This is an unofficial build, not affiliated with or endorsed by VirusTotal. Report problems with the image in this repository and YARA bugs at [VirusTotal/yara](https://github.com/VirusTotal/yara/issues).

## Quick start

Scan every file under `samples` with the rules in `rules.yar`:

```sh
docker run --rm --network none --user "$(id -u):$(id -g)" -v "$PWD:/work:ro" \
  ghcr.io/randomcontainers/yara -r rules.yar samples
```

`yara` prints one line per matching rule and file, `<rule> <file>`. It exits with status 0 whether or not anything matched, and with 1 when it cannot compile the rules or scan the file it was given. When it scans a directory or a `--scan-list`, errors on individual files go to standard error and do not change the exit status. The entrypoint runs `yara` under `tini` in `/work`, so paths are relative to the directory you mount. Options may come before or after the rule file. A few more commands:

```sh
# Also print each matching string with its offset, such as 0x1a:$a: <data>
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work:ro" \
  ghcr.io/randomcontainers/yara -s rules.yar sample.bin

# Print tags and metadata, and only the rules tagged "banker"
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work:ro" \
  ghcr.io/randomcontainers/yara -g -m -t banker rules.yar sample.bin

# Scan the files listed in files.txt, one path per line, with 4 threads
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work:ro" \
  ghcr.io/randomcontainers/yara -p 4 --scan-list rules.yar files.txt

# Set the external variable "filename" that the rules use
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work:ro" \
  ghcr.io/randomcontainers/yara -d filename=invoice.pdf rules.yar invoice.pdf
```

`yara -r` follows symbolic links unless you pass `-N`. `-n` prints the rules that did not match, `-c` only the number of rules that matched, and `-D` the data that modules such as `pe` and `elf` read from each file. `yara --help` lists every option, and the [YARA documentation](https://yara.readthedocs.io/) describes the rule language and the modules.

The images contain no rules. Rule collections are published under their own licenses; mount the ones you use.

## Compiled rules

`yarac` compiles one or more rule files into a single file that `yara -C` loads without parsing the sources again, which saves time with large rule sets:

```sh
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" --entrypoint yarac \
  ghcr.io/randomcontainers/yara rules.yar rules.yarc
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work:ro" \
  ghcr.io/randomcontainers/yara -C -r rules.yarc samples
```

A YARA build loads only compiled rules in its own file format and stops with `rules were compiled with a different version of YARA` on others, so keep the sources and compile them again after an update. Load compiled rules only from sources you trust. YARA 4.5.8 has two bugs in the loader, CVE-2026-88340 and CVE-2026-88341, that let a crafted compiled rule file crash YARA or corrupt its memory; upstream fixed both after that release. For rules from other people, get the source and compile it yourself.

## Modules

| Module | What it reads | Built with |
|---|---|---|
| `pe` | Windows executables and DLLs, including Authenticode signatures | OpenSSL |
| `elf` | Linux and other ELF executables and libraries | |
| `macho` | Mach-O executables and libraries, as used on macOS and iOS | |
| `dex` | Android DEX files | |
| `dotnet` | .NET assemblies | |
| `hash` | MD5, SHA-1 and SHA-256 hashes and CRC32 checksums of the file, a range of it or a string | OpenSSL |
| `math` | Entropy and other statistics of the file or a range of it | |
| `magic` | The file type and MIME type, as `file` reports them | libmagic and the distro's magic database |
| `cuckoo` | A Cuckoo sandbox JSON report, passed with `-x cuckoo=report.json` | jansson |
| `time`, `string`, `console` | The current time, string conversions, and `console.log()` for debugging rules | |

`yara --module-names` lists them. YARA can also scan the memory of a running process (`yara rules.yar <pid>`), which needs ptrace access to that process. A container sees no processes outside it unless you share the host's PID namespace, so the images are meant for files.

## Archives (default image)

YARA does not look inside archives: a zip or 7z file is scanned as the compressed bytes it contains. Samples are usually shared in zip files encrypted with the password `infected`, so unpack them first with `7zz` from the default image:

```sh
docker run --rm --network none --user "$(id -u):$(id -g)" -v "$PWD:/work" --entrypoint sh \
  ghcr.io/randomcontainers/yara -c '7zz x -pinfected -osamples samples.zip > /dev/null && yara -r rules.yar samples'
```

`7zz x -so` writes the extracted files to standard output instead, one after another, so name the one file you want: `7zz x -so -pinfected samples.7z sample.exe > sample.bin`. The [7zip repository](https://github.com/randomcontainers/7zip) describes the formats `7zz` reads.

## Untrusted files

YARA's modules parse the structure of executables (PE, ELF, Mach-O, DEX and .NET), and that code has had out-of-bounds read bugs that crafted files trigger. Mount the samples read-only and take away what the container does not need:

```sh
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work:ro" \
  --network none --read-only --cap-drop ALL --security-opt no-new-privileges \
  --memory 2g --pids-limit 64 \
  ghcr.io/randomcontainers/yara -r rules.yar samples
```

`-a <seconds>` stops the scan of a single file that takes longer than that, and `yara` exits with 1. For a directory or a `--scan-list` it is one limit for the whole run: the files still being scanned when it runs out fail with `scanning timed out`, the files not reached yet are skipped without a message, and `yara` can still exit with 0. Use it there only with a limit well above the time the whole scan needs.

## What is in the image

| | slim | default |
|---|---|---|
| `yara` and `yarac`, with libyara linked in | yes | yes |
| 7-Zip (`7zz` and the `7z` link) | no | yes |

7-Zip is the [randomcontainers/7zip](https://github.com/randomcontainers/7zip) build.

Not included: the `libyara` library and headers for building your own programs, the manual pages, `yara-python`, and any rules. The configure options, the commit and the module list are in `/usr/local/share/randomcontainers/yara/buildinfo`.

## Default or slim

Use the default image (`latest`) when your samples come in archives: 7-Zip unpacks zip, 7z, RAR and many other formats, including encrypted ones (see [Archives](#archives-default-image)). `slim` has YARA and the libraries it needs, without 7-Zip. Use it to scan files that are already unpacked, or as the base of your own image.

The default image includes 7-Zip, which is licensed under the GNU Lesser General Public License (LGPL-2.1-or-later), and its RAR decoders carry the unRAR license restriction. Use `slim` if your policy excludes either. The default image is also published as `ghcr.io/randomcontainers/yara-7zip`, built in the [yara-7zip](https://github.com/randomcontainers/yara-7zip) repository with the same contents and a different digest.

## Tags

`<version>` is a YARA release such as `4.5.8`. `<minor>` and `<major>` are its shorter forms, `4.5` and `4`, and follow the newest release in that series.

| Default (with 7-Zip) | Slim | Base |
|---|---|---|
| `latest`, `<version>`, `<minor>`, `<major>` | `slim`, `<version>-slim`, `<minor>-slim`, `<major>-slim` | Ubuntu |
| `ubuntu`, `<version>-ubuntu`, `<minor>-ubuntu`, `<major>-ubuntu` | `slim-ubuntu`, `<version>-slim-ubuntu`, `<minor>-slim-ubuntu`, `<major>-slim-ubuntu` | Ubuntu |
| `<version>-ubuntu26.04` | `<version>-slim-ubuntu26.04` | Ubuntu 26.04 |
| `alpine`, `<version>-alpine`, `<minor>-alpine`, `<major>-alpine` | `slim-alpine`, `<version>-slim-alpine`, `<minor>-slim-alpine`, `<major>-slim-alpine` | Alpine |
| `<version>-alpine3.24` | `<version>-slim-alpine3.24` | Alpine 3.24 |

The images are currently built on Ubuntu 26.04 and Alpine 3.24. Tags without a distro version move to the next distro release when the project does; tags ending in `ubuntu26.04` or `alpine3.24` stay on that release and are no longer rebuilt once the project moves to the next one. Every tag of the current YARA version, including the exact version, is rebuilt in place (see [Updates](#updates)), so pin a digest when you need the same bytes every time.

## Platforms

`linux/amd64` and `linux/arm64`, for both Ubuntu and Alpine. Both are compiled natively on GitHub-hosted runners, without emulation.

## Files and permissions

The working directory is `/work`. The image runs as UID 1000, and any other UID works too: `HOME` is then `/`, and caches go to `/cache`, which anyone can write to. How to get output files owned by you depends on how you run containers:

| Runtime | Flag |
|---|---|
| Docker on Linux (rootful), GitHub Actions | `--user "$(id -u):$(id -g)"` |
| Rootless Podman | `--userns=keep-id` |
| Rootless Docker | `--user 0:0` (root in the container is your user on the host) |
| Docker Desktop on macOS or Windows | none, file ownership is mapped for you |

`yara` only reads the files it scans, so a read-only mount (`:ro`) is enough for it. `yarac` and `7zz x` write files and need a writable mount.

## Extending the slim image

Use a `slim` tag as the base for your own image. It has no 7-Zip, so 7-Zip updates do not rebuild it. This image carries its own rule file and scans whatever is mounted at `/work`:

```dockerfile
FROM ghcr.io/randomcontainers/yara:slim-alpine@sha256:...
COPY rules.yar /usr/local/share/rules/
CMD ["-r", "/usr/local/share/rules/rules.yar", "/work"]
```

The entrypoint is `["tini", "--", "yara"]`, so `CMD` holds the options, the rules and the target. The image runs as UID 1000, so switch to `USER root` to install distro packages, then back to `USER 1000:1000`. The packages YARA needs are listed in `/usr/local/share/randomcontainers/yara/runtime-deps`. To pick up new YARA releases and base image fixes, let Dependabot or Renovate update the digest in your `FROM` line.

## Verifying

Each image has a build provenance attestation from this repository's GitHub Actions run, signed by the shared build workflow in `randomcontainers/ci`:

```sh
gh attestation verify oci://ghcr.io/randomcontainers/yara:latest \
  --repo randomcontainers/yara --signer-repo randomcontainers/ci
```

Images from `ghcr.io/randomcontainers/yara-7zip` are built in that repository, so verify them with `--repo randomcontainers/yara-7zip` and the same `--signer-repo`.

Each platform image also carries an SPDX SBOM that lists every distro package with its version:

```sh
docker buildx imagetools inspect ghcr.io/randomcontainers/yara:latest --format '{{ json .SBOM }}'
```

YARA is compiled here, so it is not in the SBOM and image scanners do not check it. Its version is in `/usr/local/share/randomcontainers/yara/version` and the commit in `buildinfo` next to it.

YARA attaches no source tarball to its releases and does not sign its tags, so each version is pinned to the commit of its tag. The build clones the `v<version>` tag of [VirusTotal/yara](https://github.com/VirusTotal/yara) and stops unless the tag points to the commit recorded in `package.yml`, so a tag that is moved after the version was picked up fails the build. The parsers and lexers that upstream generated with Bison and flex are generated again from their sources. The build runs upstream's test suite (on Alpine without `test-arena` and `test-api`, which Alpine's own package also skips) and stops unless both tools load OpenSSL, jansson and libmagic and `yara --module-names` lists every module above.

## Updates

The project checks the tags of [VirusTotal/yara](https://github.com/VirusTotal/yara/tags) every 15 minutes. It follows tags because upstream has published some GitHub releases weeks after tagging them. A release is picked up once the commit it tags is 24 hours old; release candidates are skipped. The new version and the commit of its tag are then committed to `package.yml` and the images are rebuilt. Only the newest release is built; tags of older versions stay as they were last built.

The images of the current version are also rebuilt when the Ubuntu or Alpine base image changes, the default ones when a new 7-Zip image is published, and all of them at least every 7 days, so distro security fixes reach the current tags.

## Building

```sh
docker build -f Dockerfile.ubuntu --target slim \
  --build-arg VERSION=<version> \
  --build-arg SOURCE_COMMIT=<commit from package.yml> \
  -t yara:local .
```

Use `Dockerfile.alpine` for the Alpine image. `--build-arg JOBS=<n>` limits the number of parallel compile jobs. The default image is generated from the `combos` entry in `package.yml` by [randomcontainers/ci](https://github.com/randomcontainers/ci).

## Licenses

YARA is licensed under the [BSD 3-Clause license](https://github.com/VirusTotal/yara/blob/master/COPYING). Parts of it keep other terms:

- Apache-2.0: the `dotnet` module.
- MIT: the Authenticode parser from Avast, used by the `pe` module.
- Apache-2.0 or BSD-3-Clause, at the user's choice: the TLSH code from Trend Micro, by way of Avast's tlshc.

The license texts are in `/usr/local/share/randomcontainers/yara/licenses/`: `COPYING` (BSD-3-Clause), `LICENSE.dotnet` and `LICENSE.authenticode`, cut from the comments of those sources, `LICENSE.tlshc`, the license file of [avast/tlshc](https://github.com/avast/tlshc) with Trend Micro's notice, and `Apache-2.0`, the license text from the Apache Software Foundation. The last two come from [`licenses/`](licenses/) in this repository, since the YARA source has no copy of them. `/usr/local/share/randomcontainers/yara/source` names the commit that was compiled. The parsers and lexers are generated during the build by the distro's Bison and flex, and Bison's license exception allows its output to be used under any terms. The image's license label is `BSD-3-Clause AND Apache-2.0 AND MIT`.

The default image adds 7-Zip, licensed under LGPL-2.1-or-later with BSD-licensed parts and the unRAR license restriction; see [randomcontainers/7zip](https://github.com/randomcontainers/7zip#licenses) for its sources and license files. OpenSSL, jansson, libmagic and the other libraries from Ubuntu or Alpine keep their own licenses. The SBOM lists them.

The files in this repository are available under the MIT license, see [LICENSE](LICENSE), except the license texts in `licenses/`, which keep their own terms.

## Requesting a tool

To suggest another tool, use the [Request a tool](https://github.com/randomcontainers/.github/issues/new?template=tool-request.yml) form.
