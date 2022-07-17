
# Introduction

This is an action based repo to collect, process and release current versions of the source code build dependencies for the [qbittorrent-nox static build script](https://github.com/userdocs/qbittorrent-nox-static).

They are used as for three main purposes:

- To reduce unnecessary bandwidth usage for hosts when using matrix builds as they may be doing hundreds of unique builds, all  repeatedly downloading the same remotes files over and over. The standard build and release workflow has 16 jobs downloading 15  dependencies each. This means each file is downloaded 16 times from the remote host per workflow run.

- As an alternative URL location for downloading the dependencies when this variable is set `export qbt_workflow_files=yes` and the script is run as an action or by the end user.

- To be used in workflows and uploaded as artifacts in the matrix build which allows the matrix builds to download the artifacts. This option is specific and unique to specially crafted workflows and is not meant to be used by the end user.

It runs every 2 hours and only keeps the most current versions of the dependencies.

## Build dependencies

|    Dependencies    |
| :----------------: |
|       bison        |
|        gawk        |
|       glibc        |
|       glibc        |
|        zlib        |
|       iconv        |
|        icu         |
| double conversion  |
|      openssl       |
|       boost        |
|   libtorrent 1.2   |
|   libtorrent 2.0   |
| qt5 base and tools |
| qt6 base and tools |
|     qBitorrent     |

## There is more

This repo also has a workflow file - [check_new_release.yml](https://github.com/userdocs/qbt-workflow-files/blob/main/.github/workflows/check_new_release.yml) - that will:

- check for current latest releases of core dependencies and automatically trigger the [files_download_and_release.yml](https://github.com/userdocs/qbt-workflow-files/blob/main/.github/workflows/files_download_and_release.yml) and update those new dependencies to the `qbt-workflow-files` latest release

- Check the new workflow latest release assets against the currently built `qbittorrent-nox-static` build versions and trigger a rebuild if there is an update required.

- Process and compare these data groups `Latest release` <> `qbt-workflow-files` <> `qbittorrent-nox-static`

These are the three main versions data sets I need gather and to process

It has some intelligent features via github marketplace actions like waiting for triggered local remote workflows to complete and preventing duplicate workflows.

## The drama - check tracking dependency versions

One of the main hurdles to doing this is that not all the dependencies have a latest release. Most just use tags and some are not even on Github. The problem with a tag or lack of tag, is that there is no consistent way to check the time frame of it's creation from when it is available to parse on Github.

Why? Because a tag can be created in a repo and not pushed for some time. Making any sort of check based on time alone inconsistent. Github does not provide a time for the tag via the API either.

So take this example:

A tag is created at 14:00 on the 01/01/2022 in a repo and pushed on the 14:00 03/1/2022.

If I try to check for a new tag with a 24 hour period, this tag will be too old and not included, even though it is the latest release. So the further back I look the more complicated it becomes. This does not really look like a problem worth fighting unless I can somehow manage a list of dependencies myself, which is where this is going so keep reading.

*Note: If we were only looking at latest releases I could easily just do this to check every 3 hours for new releases published in the last 3 hours.*

```bash
# credits to emanuele6 in #bash libera.chat
if curl -sL "https://api.github.com/repos/arvidn/libtorrent/releases/latest" | jq -er '.published_at | now - fromdateiso8601 < 10800' &> /dev/null; then
 printf '%s\n' "There was a new release in the last 3 hours"
 # run this command
fi
```

Latest release = very simple and tags or lack of = complicated.

So how did I deal with tags, a lack of tags or non github versions to get a consistent dependency tracking solution for my needs?

Since I need to get the current versions of any files I am uploading to this workflow and I already have all the methods I need to do that in the workflow I have all the version data I need. This can processed by the workflow and ready to be used.

So the question becomes what do I do with that information to make it easily accessible, consistent and reusable any time I need it and how do I check another repo/release to compare the versions of a range of dependencies?

The only solution involves keeping track of the dependency versions somewhere on github via the workflow. So the most sensible place would in the release information itself. There are other methods and options to consider.

- unused method - The [tag could be used](https://github.com/userdocs/qbt-workflow-files/tags) but there is limited scope with which to customise it and also it's kind of ugly.

- unused method - A file committed to the github repo. This could work and is a valid option but this is not what I did.

- used method -  A file published as a release asset `dependency-version.json`. This is another valid option I have implemented.

    The latest release for `qbt-workflow-files` has a special `dependency-version.json` found here

    https://github.com/userdocs/qbt-workflow-files/releases/latest/download/dependency-version.json

    ```json
    {
    "qbittorrent": "4.4.3.1",
    "qt5": "5.15.5",
    "qt6": "6.3.1",
    "libtorrent_1_2": "1.2.16",
    "libtorrent_2_0": "2.0.6",
    "boost": "1.79.0",
    "openssl": "3.0.5"
    }
    ```

    When using that URL with jq we can see parse the json in bash

    ```bash
    curl -sL https://github.com/userdocs/qbt-workflow-files/releases/latest/download/dependency-version.json | jq
    ```

    Using these commands to first check it does not exist and then to create the associative array we need.

    ```bash
    declare -p current_workflow_version # check the array
    declare -A current_workflow_version #  create the array
    ```

    We can use this command to populate the array

    ```bash
    # credits to geirha in #bash libera.chat
    eval "$(curl -sL "https://github.com/userdocs/qbt-workflow-files/releases/latest/download/dependency-version.json" | jq -r 'to_entries[]|@sh"current_workflow_version[\(.key)]=\(.value)"')"
    ```

    Now check it again

    ```bash
    declare -p current_workflow_version # check the array
    ```

- used method - The text body of release information itself because my workflows was already set up to do this and it only required a small adjustment of the workflow.

But who wants pointless and ugly info in their release body that no one understands but you whilst being unique enough that there is no conflict when parsing the json returned data.

## The Solution

When this [qbt-workflow-files/check_new_release.yml](https://github.com/userdocs/qbt-workflow-files/blob/main/.github/workflows/check_new_release.yml) creates or updates a release it will print a html commented section at the end that is not shown in the release body when viewed as rendered markdown on Github.

Github flavoured markdown will treat is a html comment and hide it.

So here is the latest release:

<https://github.com/userdocs/qbt-workflow-files/releases/latest>

*Note: the tag is a purely numerical version of all dependencies so that if one is modified the tag changes automatically. Tracking the entirety of the list*

What has happened is that the workflow has created this hidden block of bash associative arrays by using the versions it had to acquire to create the release.

```bash
<!--
declare -A current_workflow_version
current_workflow_version[qbittorrent]="4.4.3.1"
current_workflow_version[qt5]="5.15.5"
current_workflow_version[qt6]="6.3.1"
current_workflow_version[libtorrent_1_2]="1.2.16"
current_workflow_version[libtorrent_2_0]="2.0.6"
current_workflow_version[boost]="1.79.0"
current_workflow_version[openssl]="3.0.5"
-->
```

To see what's actually there you can view it via `jq` and this command:

```bash
curl -sL https://api.github.com/repos/userdocs/qbt-workflow-files/releases/latest | jq -r '.body'
```

Now if we do this command we will get a response that has the exactly filtered info I need to compare versions accurately:

```bash
curl -sL https://api.github.com/repos/userdocs/qbt-workflow-files/releases/latest | jq -r '.body' | awk '/<!--/,/-->/{sub("<!--", "");sub("-->", "");sub("\r", ""); print $0 }'
```

See it in action:

```bash
declare -p current_workflow_version # you will get an error - -bash: declare: current_workflow_version: not found - as this array does not exist
```

Source the remote date

```bash
source <(curl -sL https://api.github.com/repos/userdocs/qbt-workflow-files/releases/latest | jq -r '.body' | awk '/<!--/,/-->/{sub("<!--", "");sub("-->", "");sub("\r", ""); print $0 }')
```

Now show the array information again.

```bash
declare -p current_workflow_version
```

You will see it is properly created and populated the array with the information we need.

```bash
declare -A current_workflow_version=([libtorrent_1_2]="1.2.16" [qbittorrent]="4.4.3.1" [libtorrent_2_0]="2.0.6" [boost]="1.79.0" [qt6]="6.3.1" [qt5]="5.15.5" [openssl]="3.0.5" )
```

Now we can iterate over the values or get applications specific versions, like this `boost` example.

```bash
printf '%s\n' "${current_workflow_version[boost]}"
```

Giving the array version for `boost`

```bash
1.79.0
```

It can also get specific versions easily with a little tweak

```bash
curl -sL https://api.github.com/repos/userdocs/qbt-workflow-files/releases/latest | jq -r '.body' | awk '/current_workflow_version\[qbittorrent\]/{sub("\r", ""); print $0 }'
```

Which returns

```bash
current_workflow_version[qbittorrent]="4.4.3.1"
```

So by making both the [qbt-workflow-files](https://github.com/userdocs/qbt-workflow-files/releases/latest) and [qbittorrent-nox-static](https://github.com/userdocs/qbittorrent-nox-static/releases/latest) latest releases body contain an easily accessible, filterable and processable data block that is not visible we can extend the functionality of the release to provide data the API cannot handle.

Especially when using the Github tag is not an option.

## Conclusion

Using the release body to contain html commented associative bash array data for a range of applications dependencies I can easily compare versions using workflows with minimal API calls.

Using a `dependency-version.json` uploaded as a release asset we can access this file without the API and parse it with `jq`, creating the arrays that way.

Being able to comparatively process this data across multiple repos allows me to trigger workflows locally or remotely and automate the process of updating the dependencies and applications I build using them.
