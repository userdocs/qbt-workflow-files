
# Introduction

This is an action based repo to collect, process and release current versions of the source code build dependencies for the [qbittorrent-nox static build script](https://github.com/userdocs/qbittorrent-nox-static).

They are used as for three main purposes:

- To reduce unnecessary bandwidth usage for hosts when using matrix builds as they may be doing hundreds of unique builds, all  repeatedly downloading the same remotes files over and over. The standard build and release workflow has 32 jobs downloading 15  dependencies each. This means each file is downloaded 32 times from the remote host per workflow run.

- As an alternative URL location for downloading the dependencies when this variable is set `export qbt_workflow_files=yes` and the script is run as an action or by the end user.

- To be used in workflows and uploaded as artifacts in the matrix build which will allow the matrix builds to download the artifacts. This option is specific and unique to specially crafted workflows and it not meant to be used by the end user.

It runs 4 times a day and only keeps the most current versions of the dependencies.

## Build dependencies

bison
gawk
glibc
glibc
zlib
iconv
icu
double conversion
openssl
boost
libtorrent 1.2
libtorrent 2.0
qt5 base and tools
qt6 base and tools
qBitorrent
