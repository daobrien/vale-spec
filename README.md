# Vale

This repository maintains a SPEC file for the 
[vale](https://github.com/errata-ai/vale) command-line tool.

## Installing Vale

On Fedora and CentOS, you can enable the `mczernek/vale` COPR repository and install the `vale` package:


```
[user@host ~] $ sudo dnf copr enable mczernek/vale
[user@host ~] $ sudo dnf install vale
```

On other systems, download and install the RPM manually.
You can either build the RPM or download the RPM from the [COPR repository](https://copr.fedorainfracloud.org/coprs/mczernek/vale/).

## Red Hat Training curriculum repositories

The Jenkins pipeline in this repository (see `Jenkinsfile`) builds the RPM in
mock for RHEL 9 and 10, CentOS Stream 9 and 10, and current Fedora releases,
then publishes the packages to the Red Hat Training curriculum yum repositories
on every push to `main`. Packages land under `<platform>/<version>/SRPMS/` and
`<platform>/<version>/x86_64/`, for example `rhel/9/x86_64/`.

Access to the curriculum repositories requires a GitHub personal access token
with the `read:org` scope on the RedHatTraining organization. For instructions
on configuring dnf or yum to use these repositories, see
[curriculum-release-repos](https://github.com/RedHatTraining/curriculum-release-repos).

## Filing Bugs

For issues with packaging, please create an [issue](https://github.com/daobrien/vale-spec/issues).

For issues with core Vale functionality, go to [vale](https://github.com/errata-ai/vale/issues).

