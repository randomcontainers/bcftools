# bcftools

Container images with [BCFtools](https://samtools.github.io/bcftools/), the samtools project's program for calling, filtering, merging and querying genetic variants in VCF and BCF files. The images also have `bgzip` and `tabix` from the HTSlib release bundled with BCFtools. BCFtools is compiled from the release tarball on Ubuntu and Alpine, for `linux/amd64` and `linux/arm64`, and the images are rebuilt when a new BCFtools release comes out and when the base image changes.

This is an unofficial build, not affiliated with or endorsed by the BCFtools developers or Genome Research Ltd. Report problems with the image in this repository and BCFtools bugs at [samtools/bcftools](https://github.com/samtools/bcftools/issues).

## Quick start

Print the header of a VCF file in the current directory:

```sh
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" ghcr.io/randomcontainers/bcftools view -h in.vcf.gz
```

The entrypoint runs `bcftools` under `tini`, so the bcftools command and its options go after the image name. A few more commands:

```sh
# Keep the variants that pass all filters, as compressed VCF with an index
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" ghcr.io/randomcontainers/bcftools \
  view -f PASS -Oz -o pass.vcf.gz -W in.vcf.gz

# Split multiallelic sites and left-align indels against the reference
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" ghcr.io/randomcontainers/bcftools \
  norm -m-any -f ref.fa -Oz -o norm.vcf.gz -W in.vcf.gz

# Print position and alleles of each site as tab-separated text
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" ghcr.io/randomcontainers/bcftools \
  query -f '%CHROM\t%POS\t%REF\t%ALT\n' in.vcf.gz > sites.tsv

# Call SNPs and indels from aligned reads
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" --entrypoint sh ghcr.io/randomcontainers/bcftools \
  -c 'bcftools mpileup -Ou -f ref.fa in.bam | bcftools call -mv -Oz -o calls.vcf.gz'
```

The [BCFtools manual](https://samtools.github.io/bcftools/bcftools.html) describes every command, and running the image without arguments lists them.

## Plugins

The plugins that come with BCFtools, such as `fill-tags`, `setGT`, `split-vep` and `trio-dnm3`, are in `/usr/local/libexec/bcftools`. bcftools looks there by default, so `+<name>` runs a plugin without setting `BCFTOOLS_PLUGINS`:

```sh
# Add AC, AN and AF to the INFO column
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" ghcr.io/randomcontainers/bcftools \
  +fill-tags -Oz -o tagged.vcf.gz in.vcf.gz -- -t AC,AN,AF

# List the plugins with a short description of each
docker run --rm ghcr.io/randomcontainers/bcftools plugin -lv
```

To add plugins compiled for the same BCFtools version, mount their directory and end `BCFTOOLS_PLUGINS` with a colon, which keeps the default directory in the search path: `-v "$PWD/plugins:/plugins" -e BCFTOOLS_PLUGINS=/plugins:`.

## bgzip and tabix

`bgzip` compresses VCF and other text files into the block-gzip format that indexed formats need, and `tabix` indexes them. Run them with `--entrypoint`:

```sh
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" --entrypoint bgzip ghcr.io/randomcontainers/bcftools in.vcf
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" --entrypoint tabix ghcr.io/randomcontainers/bcftools -p vcf in.vcf.gz
```

`bgzip in.vcf` replaces `in.vcf` with `in.vcf.gz`; add `-k` to keep the original.

## Remote files

HTSlib is built with libcurl, so bcftools also reads files from `https://` URLs and the other protocols libcurl supports, from Amazon S3 (`s3://`) and from Google Cloud Storage (`gs://`). S3 credentials come from `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY` and `AWS_SESSION_TOKEN`, or from the file named by `AWS_SHARED_CREDENTIALS_FILE`. Google Cloud Storage takes an access token in `GCS_OAUTH_TOKEN`. Pass them with `-e`:

```sh
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" -e AWS_ACCESS_KEY_ID -e AWS_SECRET_ACCESS_KEY \
  ghcr.io/randomcontainers/bcftools view -H -r chr20:1000000-1100000 s3://<bucket>/calls.vcf.gz
```

For CRAM input, give `bcftools mpileup` the reference with `-f`. HTSlib does not download CRAM references from a server unless `REF_PATH` names one.

## What is in the image

- `bcftools` in `/usr/local/bin`, with HTSlib linked in statically. Apart from the distro's C library it loads zlib, libbz2, liblzma, libdeflate, libcurl and OpenSSL's libcrypto.
- The BCFtools plugins in `/usr/local/libexec/bcftools`.
- `bgzip` and `tabix` in `/usr/local/bin`, from the same HTSlib.

Not included:

- The `polysomy` command and perl filter expressions, which need the GNU Scientific Library and libperl and would put BCFtools under the GPL.
- The scripts in BCFtools' `misc/` directory, such as `plot-vcfstats`, `vcfutils.pl` and `run-roh.pl`, which need perl or python3.
- HTSlib's `htsfile` and `annot-tsv`, and the man page.

The configure flags, the HTSlib version and the HTSlib features are in `/usr/local/share/randomcontainers/bcftools/buildinfo`.

## Default or slim

BCFtools' default image adds no other tools, so `latest` and `slim` are the same image, with the contents listed above. Use `latest` to run it and the `slim` tags as a base for your own image. The default image of [samtools](https://github.com/randomcontainers/samtools) includes this build of BCFtools, for calling variants from the reads samtools sorts and indexes.

## Tags

`<version>` is a BCFtools release such as `1.24`, or `1.24.1` for a point release. `<minor>` and `<major>` are its shorter forms, `1.24` and `1`, and follow the newest release in that series. For a release without a point number, such as `1.24`, `<version>` and `<minor>` are the same tag. Each row lists the default tag and its `slim` twin, which point to the same image.

| Tags | Base |
|---|---|
| `latest`, `slim` | Ubuntu |
| `<version>`, `<version>-slim` | Ubuntu |
| `<minor>`, `<minor>-slim`, `<major>`, `<major>-slim` | Ubuntu |
| `ubuntu`, `slim-ubuntu` | Ubuntu |
| `<version>-ubuntu`, `<version>-slim-ubuntu` | Ubuntu |
| `<minor>-ubuntu`, `<minor>-slim-ubuntu`, `<major>-ubuntu`, `<major>-slim-ubuntu` | Ubuntu |
| `<version>-ubuntu26.04`, `<version>-slim-ubuntu26.04` | Ubuntu 26.04 |
| `alpine`, `slim-alpine` | Alpine |
| `<version>-alpine`, `<version>-slim-alpine` | Alpine |
| `<minor>-alpine`, `<minor>-slim-alpine`, `<major>-alpine`, `<major>-slim-alpine` | Alpine |
| `<version>-alpine3.24`, `<version>-slim-alpine3.24` | Alpine 3.24 |

The images are currently built on Ubuntu 26.04 and Alpine 3.24. Tags without a distro version move to the next distro release when the project does; tags ending in `ubuntu26.04` or `alpine3.24` stay on that release and are no longer rebuilt once the project moves to the next one. Every tag of the current BCFtools version, including the exact version, is rebuilt in place (see [Updates](#updates)), so pin a digest when you need the same bytes every time.

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

Mount the directory that holds your files read-write: `-W` and `bcftools index` write the index next to the output, and `mpileup -f` creates `ref.fa.fai` next to the reference when it is missing. `bcftools sort` keeps up to 768 MB in memory (`-m`) and writes the rest to a temporary directory under `/tmp`, so a container started with `--read-only` also needs `--tmpfs /tmp` or `-e TMPDIR=/work`.

## Extending the slim image

Use a `slim` tag as the base for your own image. `slim`, `slim-ubuntu` and `slim-alpine` move to each new BCFtools release and are rebuilt when the base image changes. Switch to root to install more, then back:

```dockerfile
FROM ghcr.io/randomcontainers/bcftools:slim-ubuntu@sha256:...
USER root
RUN apt-get update \
 && apt-get install -y --no-install-recommends procps \
 && rm -rf /var/lib/apt/lists/*
USER 1000:1000
ENTRYPOINT []
```

This prepares the image for Nextflow, which needs `bash` and `ps` in a task's container and starts tasks with its own `/bin/bash` command, so the example clears the `bcftools` entrypoint. On Alpine, start from `slim-alpine` and use `apk add --no-cache bash procps`. Leave out `ENTRYPOINT []` to keep the slim image's entrypoint, `["tini", "--", "bcftools"]`. To pick up new BCFtools releases and base image fixes, let Dependabot or Renovate update the digest in your `FROM` line.

Everything the image adds is under `/usr/local`. `/usr/local/share/randomcontainers/bcftools/` holds the version, the source URL, the build options, the license files and `runtime-deps`, the list of distro packages bcftools needs at run time.

## Verifying

Each image has a build provenance attestation from this repository's GitHub Actions run, signed by the shared build workflow in `randomcontainers/ci`:

```sh
gh attestation verify oci://ghcr.io/randomcontainers/bcftools:latest \
  --repo randomcontainers/bcftools --signer-repo randomcontainers/ci
```

Each platform image also carries an SPDX SBOM that lists every distro package with its version:

```sh
docker buildx imagetools inspect ghcr.io/randomcontainers/bcftools:latest --format '{{ json .SBOM }}'
```

HTSlib is compiled into bcftools, so the SBOM does not list it and vulnerability scanners that read the SBOM miss it. `bcftools --version` and `buildinfo` name its version. Before compiling, the build checks the tarball against the SHA-256 recorded in `package.yml`. After compiling, it runs the BCFtools test suite with the plugins, `bgzip` and `tabix`.

## Updates

The project checks the releases of [samtools/bcftools](https://github.com/samtools/bcftools/releases) every 15 minutes and follows the 1.x series. A release is picked up once it is 24 hours old. The BCFtools developers publish no checksum file or signature, so the tarball is checked against the SHA-256 digest GitHub records for the release asset. The new version and the tarball's SHA-256 are then committed to `package.yml`, and the images are rebuilt. Only the newest release is built; tags of older versions stay as they were last built, and a point release of an older series, such as 1.23.2 after 1.24, is not built.

HTSlib security fixes reach the image with the BCFtools release that bundles them. HTSlib sometimes has a security release of its own: HTSlib 1.22.1 came out in July 2025, and the next BCFtools release followed five months later. In between, the image keeps the older HTSlib.

The images of the current version are also rebuilt when the Ubuntu or Alpine base image changes and at least every 7 days, so distro security fixes reach the current tags.

## Building

```sh
docker build -f Dockerfile.ubuntu --target slim \
  --build-arg VERSION=<version> \
  --build-arg SOURCE_SHA256=<sha256 from package.yml> \
  -t bcftools:local .
```

Use `Dockerfile.alpine` for the Alpine image. `--build-arg JOBS=<n>` limits the number of parallel compile jobs.

## Licenses

BCFtools may be used under the MIT/Expat license or under the GNU GPL version 3 or later. A build with the GNU Scientific Library is GPL only. This build leaves out GSL and the perl filters, so it is under the MIT/Expat license, which `bcftools --version` prints. The bundled HTSlib is MIT, except its `cram/` directory (BSD-3-Clause, from Genome Research Ltd. and the Medical Research Council) and its htscodecs library (BSD-3-Clause). Other code compiled in keeps its own terms:

- MIT: code from the Broad Institute and the Massachusetts Institute of Technology, Heng Li, Intel, the University of Glasgow, Attractive Chaos (klib), Martin Šošić (edlib), GENOMICS plc and Nicola Asuni (VariantKey), and plugins by other authors.
- BSD-1-Clause: `utlist.h` from uthash, by Troy D. Hanson.
- CC0-1.0: `rANS_byte.h` and `rANS_word.h` in htscodecs, from Fabian Giesen's ryg_rans.
- Public domain: HTSlib's `md5.c` by Alexander Peslyak, the parts of `hts_time_funcs.h` taken from NetBSD's `localtime.c`, and `c_range_coder.h` in htscodecs by Eugene Shelwien.

`/usr/local/share/randomcontainers/bcftools/licenses/` holds BCFtools' `LICENSE` (with the MIT, GPL, VariantKey, uthash and edlib texts), HTSlib's `LICENSE` as `LICENSE.htslib`, htscodecs' `LICENSE.md` as `LICENSE.htscodecs.md`, and `NOTICES`, which the build fills with the first copyright comment of each C file outside the test suites when that comment names a holder other than Genome Research Ltd. The source tarball URL is in `/usr/local/share/randomcontainers/bcftools/source`. The image's license label is `MIT AND BSD-3-Clause AND BSD-1-Clause AND CC0-1.0 AND LicenseRef-Public-Domain`. The Ubuntu and Alpine packages in the image keep their own licenses, for example the curl license for libcurl and Apache-2.0 for OpenSSL. The SBOM lists them.

The files in this repository are available under the MIT license, see [LICENSE](LICENSE).

## Requesting a tool

To suggest another tool, use the [Request a tool](https://github.com/randomcontainers/.github/issues/new?template=tool-request.yml) form.
