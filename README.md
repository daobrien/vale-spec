# Vale

This repository maintains a SPEC file for the
[vale](https://github.com/errata-ai/vale) command-line tool and the Jenkins
pipeline that builds and publishes the RPM for the Red Hat Training curriculum
team. It is a fork of [m-czernek/vale-spec](https://github.com/m-czernek/vale-spec).

## Installing Vale

### From the Red Hat Training curriculum repositories

The RPM is published to the Red Hat Training curriculum yum repositories for
RHEL 9 and 10, CentOS Stream 9 and 10, and current Fedora releases. Packages
land under `<platform>/<version>/SRPMS/` and `<platform>/<version>/x86_64/`,
for example `rhel/9/x86_64/`.

Access requires a GitHub personal access token with the `read:org` scope on
the RedHatTraining organization. For instructions on configuring dnf or yum to
use these repositories, see
[curriculum-release-repos](https://github.com/RedHatTraining/curriculum-release-repos).
Once the repository is configured:

```
[user@host ~] $ sudo dnf install vale
```

### From COPR

The upstream maintainer publishes builds of this spec to the `mczernek/vale`
COPR repository, which works on Fedora and CentOS without any Red Hat Training
access:

```
[user@host ~] $ sudo dnf copr enable mczernek/vale
[user@host ~] $ sudo dnf install vale
```

COPR builds track the upstream repository, not this fork, so versions may lag.
Prebuilt RPMs are also available from the
[COPR repository](https://copr.fedorainfracloud.org/coprs/mczernek/vale/) for
manual download.

### Building the RPM yourself

With `mock` and `rpmdevtools` installed, fetch the source tarball and build for
any chroot:

```
[user@host vale-spec] $ spectool -g -C sources vale.spec
[user@host vale-spec] $ mock -r rhel-9-x86_64 --spec vale.spec --sources sources --resultdir out/rhel-9-x86_64
```

The `rhel-*` chroots need the host registered with `subscription-manager`.
The `centos-stream-*` and `fedora-*` chroots do not.

## Building and publishing

The `Jenkinsfile` defines the pipeline that runs on every push:

1. **Fetch source.** Downloads the tarball named by `Source0` in the spec.
2. **Build.** Runs `mock` for every chroot in the `CHROOTS` map, in parallel,
   writing RPMs and logs under `out/<chroot>/`.
3. **Publish.** On `main` only, uploads each SRPM and x86_64 RPM to the
   curriculum repositories. Packages that are already published are skipped,
   so rerunning a build for a released version is a no-op.

Build output and mock logs are archived on every run, including failures.

### Releasing a new version

1. Update `Version` in `vale.spec` to the new upstream release and reset
   `Release` to `1%{?dist}`. The `automation/main.py` script does this
   automatically by checking the latest upstream GitHub release.
2. Add a `%changelog` entry.
3. Open a pull request. Merging to `main` triggers the build and publish.

To publish a new build of the same vale version, bump `Release` instead.

### Adding or removing a target

Edit the `CHROOTS` map at the top of the `Jenkinsfile`. Each entry maps a mock
chroot name to the `<platform>/<version>` path it publishes to. The Jenkins
agent needs a matching config in `/etc/mock/` and the distribution's signing
key in `/usr/share/distribution-gpg-keys/`, both of which come from the
`mock-core-configs` and `distribution-gpg-keys` packages.

## Filing Bugs

For issues with packaging or the pipeline, please create an
[issue](https://github.com/RedHatTraining/vale-spec/issues).

For issues with core Vale functionality, go to
[vale](https://github.com/errata-ai/vale/issues).
