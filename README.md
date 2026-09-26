# mediainfo

Container images with the command-line version of [MediaInfo](https://mediaarea.net/en/MediaInfo), which reports the codecs, bit rates, frame rates, HDR formats, subtitle tracks and tags of video and audio files. It is compiled from MediaArea's source release with libcurl, so it can also read files from http, https and ftp URLs. The default image also contains FFmpeg. The images are built for `linux/amd64` and `linux/arm64` and rebuilt when the base image changes. A new MediaInfo release can take a few days to appear (see [Updates](#updates)).

This is an unofficial build, not affiliated with or endorsed by MediaArea. Report problems with the image in this repository and problems with MediaInfo itself [upstream](https://github.com/MediaArea/MediaInfo/issues).

## Quick start

```sh
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" \
  ghcr.io/randomcontainers/mediainfo input.mkv
```

The same images can also be pulled as `randomcontainers.com/mediainfo`.

Write the full report as JSON. `--Output=XML` gives MediaArea's XML format instead:

```sh
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" \
  ghcr.io/randomcontainers/mediainfo --Output=JSON input.mp4 > input.json
```

Print only the fields you need with a template. `--Info-Parameters` lists every field name:

```sh
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" \
  ghcr.io/randomcontainers/mediainfo --Output='Video;%Width%x%Height% %FrameRate% fps' input.mp4
```

A URL needs no mount. When the server supports range requests, mediainfo reads only the parts of the file it needs:

```sh
docker run --rm ghcr.io/randomcontainers/mediainfo https://example.com/video.mp4
```

The entrypoint runs `mediainfo` under `tini`. The default image also has `ffprobe` and `ffmpeg`; override the entrypoint to run them:

```sh
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" \
  --entrypoint ffprobe ghcr.io/randomcontainers/mediainfo -hide_banner input.mp4
```

## What is in the image

| Area | Details |
|---|---|
| Build | The mediainfo command with MediaInfoLib and ZenLib linked in statically, from MediaArea's all-in-one source archive |
| Input | Local files, and http, https and ftp URLs through the distro's libcurl with the system CA certificates |
| Output | Text, JSON, XML, HTML, CSV, EBUCore, PBCore 2 and templates (`--Output='Audio;%Format%'`) |
| Default image | FFmpeg (`ffmpeg` and `ffprobe`), which `mediainfo --Enable_FFmpeg=1` also uses |

Not included: the GUI, the shared `libmediainfo` library and its headers, and SVG graphs (`--Output=Graph_Svg` needs Graphviz; `--Output=Graph_Dot` works). The compiler and autotools versions, the configure summary and the `mediainfo --Version` output are in `/usr/local/share/randomcontainers/mediainfo/buildinfo`.

## Default or slim

Use the default image (`latest`) for general work. It adds FFmpeg, so you can check a file with both `mediainfo` and `ffprobe` or convert it with `ffmpeg` in the same image. With `--Enable_FFmpeg=1`, mediainfo also runs FFmpeg's `cropdetect` filter on four keyframes from the middle of the file and reports the picture inside black borders as `Active_Width` and `Active_Height`. A field stays empty when it equals the full width or height:

```sh
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" \
  ghcr.io/randomcontainers/mediainfo --Enable_FFmpeg=1 \
  --Output='Video;%Width%x%Height%, active %Active_Width%x%Active_Height%' input.mkv
```

`slim` has mediainfo and the libraries it needs, without FFmpeg. Use it to build your own image, or when you only need mediainfo. The default image of [MKVToolNix](https://github.com/randomcontainers/mkvtoolnix) includes the `slim` build.

The default image includes FFmpeg, which is licensed under the GNU General Public License (GPL-3.0-or-later). Use `slim` if your policy excludes GPL. The default image is also published as `ghcr.io/randomcontainers/mediainfo-ffmpeg`, built in the [mediainfo-ffmpeg](https://github.com/randomcontainers/mediainfo-ffmpeg) repository with the same contents and a different digest.

## Tags

`<version>` is a MediaInfo release such as `26.05`. `<x>` is its first part, `26`, and follows the newest release that starts with it. A point release such as `26.05.1` also gets `<x.y>` tags (`26.05`, `26.05-slim-alpine` and so on), which follow the newest point release.

| Default (with FFmpeg) | Slim | Base |
|---|---|---|
| `latest`, `<version>`, `<x>` | `slim`, `<version>-slim`, `<x>-slim` | Ubuntu |
| `ubuntu`, `<version>-ubuntu`, `<x>-ubuntu` | `slim-ubuntu`, `<version>-slim-ubuntu`, `<x>-slim-ubuntu` | Ubuntu |
| `<version>-ubuntu26.04` | `<version>-slim-ubuntu26.04` | Ubuntu 26.04 |
| `alpine`, `<version>-alpine`, `<x>-alpine` | `slim-alpine`, `<version>-slim-alpine`, `<x>-slim-alpine` | Alpine |
| `<version>-alpine3.24` | `<version>-slim-alpine3.24` | Alpine 3.24 |

The images are currently built on Ubuntu 26.04 and Alpine 3.24. Tags without a distro version move to the next distro release when the project does; tags ending in `ubuntu26.04` or `alpine3.24` stay on that release and are no longer rebuilt once the project moves to the next one. Every tag of the current MediaInfo version, including the exact version, is rebuilt in place (see [Updates](#updates)), so pin a digest when you need the same bytes every time.

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

## Extending the slim image

Use a `slim` tag as the base for your own image. It has no FFmpeg, so FFmpeg updates do not rebuild it. The packages mediainfo needs are listed in `/usr/local/share/randomcontainers/mediainfo/runtime-deps`. Switch to root to install more, then back:

```dockerfile
FROM ghcr.io/randomcontainers/mediainfo:slim-ubuntu@sha256:...
USER root
RUN apt-get update \
 && apt-get install -y --no-install-recommends jq \
 && rm -rf /var/lib/apt/lists/*
USER 1000:1000
```

On Alpine, use `apk add --no-cache jq`. The entrypoint is `["tini", "--", "mediainfo"]`; set your own `ENTRYPOINT` if your image runs something else. To pick up new MediaInfo releases and base image fixes, let Dependabot or Renovate update the digest in your `FROM` line.

## Verifying

Each image has a build provenance attestation from this repository's GitHub Actions run, signed by the shared build workflow in `randomcontainers/ci`:

```sh
gh attestation verify oci://ghcr.io/randomcontainers/mediainfo:latest \
  --repo randomcontainers/mediainfo --signer-repo randomcontainers/ci
```

Images from `ghcr.io/randomcontainers/mediainfo-ffmpeg` are built in that repository, so verify them with `--repo randomcontainers/mediainfo-ffmpeg` and the same `--signer-repo`.

Each platform image also carries an SPDX SBOM that lists every distro package with its version:

```sh
docker buildx imagetools inspect ghcr.io/randomcontainers/mediainfo:latest --format '{{ json .SBOM }}'
```

The build downloads MediaArea's source archive `MediaInfo_CLI_<version>_GNU_FromSource.tar.xz` and checks it against the SHA-256 recorded in `package.yml` before compiling. MediaArea publishes no checksums or signatures for that archive, so before a version is pinned, the sources in it are compared with MediaArea's git repositories:

- MediaInfo and MediaInfoLib with the `v<version>` tags of [MediaArea/MediaInfo](https://github.com/MediaArea/MediaInfo) and [MediaArea/MediaInfoLib](https://github.com/MediaArea/MediaInfoLib).
- ZenLib with the commit of [MediaArea/ZenLib](https://github.com/MediaArea/ZenLib) named in `package.yml`. ZenLib has had no release since 0.4.41 in 2023, and the archive carries newer code from its master branch, so there is no release to compare it with.

Only the files the build uses are compared; the archive's Windows project files and HTML documentation are not. Its configure scripts and the other files that autotools generates are not in git, so the build deletes them and generates them again with the distro's autoconf, automake and libtool.

## Updates

MediaArea publishes no checksum for the source archive, so each MediaInfo release is added by hand after its sources are compared as described in [Verifying](#verifying), and can take a few days to appear. Only the newest release is built; tags of older versions stay as they were last built.

The images of the current version are also rebuilt when the Ubuntu or Alpine base image changes, the default ones when a new FFmpeg image is published, and all of them at least every 7 days, so distro security fixes reach the current tags.

## Building

```sh
docker build -f Dockerfile.ubuntu --target slim \
  --build-arg VERSION=<version> \
  --build-arg SOURCE_SHA256=<sha256 from package.yml> \
  -t mediainfo:local .
```

Use `Dockerfile.alpine` for the Alpine image. `--build-arg JOBS=<n>` limits the number of parallel compile jobs. The default image is generated from the `combos` entry in `package.yml` by [randomcontainers/ci](https://github.com/randomcontainers/ci).

## Licenses

MediaInfo and MediaInfoLib are distributed under the BSD 2-Clause license, and ZenLib under the zlib license. The mediainfo binary also contains code that MediaInfoLib bundles: tinyxml2 (zlib license), fmt and tfsxml (MIT), Brian Gladman's AES, SHA and HMAC code (SPDX `Brian-Gladman-2-Clause` and `Brian-Gladman-3-Clause`) and a public domain MD5 implementation. MediaInfoLib also bundles a base64 routine by Markus Ewald that carries no license statement; its header comment is kept as `NOTICE.base64`. The license files and the notices of the bundled code are in `/usr/local/share/randomcontainers/mediainfo/licenses/`, and the source archive URL is in `/usr/local/share/randomcontainers/mediainfo/source`.

The default image adds FFmpeg, licensed under GPL-3.0-or-later; see [randomcontainers/ffmpeg](https://github.com/randomcontainers/ffmpeg#licenses) for its sources and license files. libcurl, zlib and the other libraries from Ubuntu or Alpine keep their own licenses. The SBOM lists them.

The files in this repository are available under the MIT license, see [LICENSE](LICENSE).

## Requesting a tool

To suggest another tool, use the [Request a tool](https://github.com/randomcontainers/.github/issues/new?template=tool-request.yml) form.
