---
layout: page
title: "Building Bitbucket Pipelines for Android"
date:   2021-04-06 18:20:00 -0700
categories: bitbucket cicd android docker
---

## TL;DR ##
Something weird is going on with certain versions of the Android Command Line Tools install of `ndk-build` that comes with `ndk-bundle`.  `ndk-bundle` includes a version of `make` so that, you know, they can `make` without worrying about whether it's installed, BUT it appears that at least one of the binaries included in the `ndk-bundle` package can't find `make`. You can use a Docker container to track down the location of the `make` executable.

### The Pipeline File and the Error ###
_bitbucket-pipelines.yml:_
```
image: java:8
pipelines:
  branches: 
    my-branch:
      # build the app
      - step:
          name: Build
          script:
            - wget --quiet --output-document=cmdline-tools.zip https://dl.google.com/android/repository/commandlinetools-linux-6858069_latest.zip
            - unzip -o -qq cmdline-tools.zip -d .
            - export ANDROID_HOME="/opt/atlassian/pipelines/agent/build/cmdline-tools"
            - export PATH="$ANDROID_HOME/bin:$ANDROID_HOME/platform-tools:$PATH" 
            - yes | sdkmanager --install "ndk-bundle" --sdk_root=$ANDROID_HOME --verbose
            - yes | sdkmanager --install "platform-tools" --sdk_root=$ANDROID_HOME
            ...<SNIP>
            - ./gradlew clean --full-stacktrace --scan
```

And I was getting **this error message**:
```
> Task :pluginproject:ndkClean FAILED
ERROR: Cannot find 'make' program. Please install Cygwin make package
or define the GNUMAKE variable to point to it.
/opt/atlassian/pipelines/agent/build/cmdline-tools/ndk-bundle/build/ndk-build: line 151: file: command not found
FAILURE: Build failed with an exception.
* What went wrong:
Execution failed for task ':pluginproject:ndkClean'.
> Process 'command '/opt/atlassian/pipelines/agent/build/cmdline-tools/ndk-bundle/ndk-build'' finished with non-zero exit value 1
```

### The Solution ###

Okay, so first of all, that whole Cygwin error is generic and what my friend called it in a spectacular portmanteau of colloquialisms, "smoking herring."

The solution is to set an environment variable called `GNUMAKE` and point it at the `make` executable that's in your `ndk-bundle` package.

```
- export GNUMAKE=$ANDROID_HOME/ndk-bundle/prebuilt/linux-x86_64/bin/make
```

The tricky part in all this was trying to remember how to use Docker - because lemme tell ya, I was tired of burning build minutes on Bitbucket doing things like "ls -la."


### Doing this locally ###
You basically have to create a local version of Bitbucket Pipelines' build environment:


1. First, set up the Docker container and do a bunch of work in it:

    ```
    # run your docker container in interactive, TTY mode
    $ docker run -it java:8 /bin/bash
    root@a128c0c78584:/# ls
    bin  boot  dev  etc  home  lib  lib64  media  mnt  opt  proc  root  run  sbin  srv  sys  tmp  usr  var

    # create the working directory that mimics BBP's env
    root@a128c0c78584:/# mkdir -p /opt/atlassian/pipelines/agent/build
    root@a128c0c78584:/# cd !$

    # so now you're going to run all of the installation commands that you'd have your BBP robot do.
    # setting ANDROID_HOME, wgetting and unzipping, sdkmanager installs, etc.
    root@a128c0c78584:/opt/atlassian/pipelines/agent/build# get --quiet --output-document=cmdline-tools.zip https://dl.google.com/android/repository/commandlinetools-linux-6858069_latest.zip
    ```

    ...etc.

2. Now, find the `make` executable and point the GNUMAKE variable to it. In this case I'm looking inside the `ndk-bundle` directory because I knew that the `ndk-build` tool that came with it was the package that was having a hard time finding `make`.

    ```
    # find your `make`
    root@a128c0c78584:/opt/atlassian/pipelines/agent/build/cmdline-tools/ndk-bundle# find . -name make
    ./prebuilt/linux-x86_64/bin/make

    # tell Gradle how to find it
    root@a128c0c78584:/opt/atlassian/pipelines/agent/build/cmdline-tools/ndk-bundle# export GNUMAKE=$ANDROID_HOME/ndk-bundle/prebuilt/linux-x86_64/bin/make
    ```

3. Now, in another window, run a `docker cp` command to get your files to the "build" directory"

    ```
    inger.klekacz@~/Projects/ik-android $ docker cp . musing_mendel:/opt/atlassian/pipelines/agent/build
    ```

4. In the terminal window where you're inside the Docker container, you can try the build now.

    ```
    root@a128c0c78584:/opt/atlassian/pipelines/agent/build/cmdline-tools/ndk-bundle# cd $AGENT_HOME

    # make sure your buildfiles are there, the ones you copied over from your parent host
    # here you can see some files that are decidedly not native to Docker - ones you downloaded and ones you copied.
    root@a128c0c78584:/opt/atlassian/pipelines/agent/build/cmdline-tools# ls -l
    drwxr-xr-x  4  504 dialout     4096 Mar 31 14:58 app
    drwxr-xr-x  3  504 dialout     4096 Mar 31 14:58 bash_scripts
    -rw-r--r--  1  504 dialout     2310 Apr  7 00:16 bitbucket-pipelines.yml
    -rw-r--r--  1  504 dialout     1074 Mar 31 14:58 build.gradle
    drwxr-xr-x 14 root root        4096 Apr  7 01:01 cmdline-tools
    -rw-r--r--  1 root root    87259900 Apr  7 00:50 cmdline-tools.zip
    drwxr-xr-x  2  504 dialout     4096 May 26  2020 failnet
    drwxr-xr-x  3  504 dialout     4096 May 26  2020 gradle
    -rw-r--r--  1  504 dialout       21 May 26  2020 gradle.properties
    -rwxr-xr-x  1  504 dialout     5299 May 26  2020 gradlew
    -rw-r--r--  1  504 dialout     2260 May 26  2020 gradlew.bat

    # run the clean!
    root@a128c0c78584:/opt/atlassian/pipelines/agent/build/cmdline-tools# ./gradlew clean


    Downloading https://services.gradle.org/distributions/gradle-6.1.1-all.zip
    <SNIP - downloading dots>
    Unzipping /root/.gradle/wrapper/dists/gradle-6.1.1-all/cfmwm155h49vnt3hynmlrsdst/gradle-6.1.1-all.zip to /root/.gradle/wrapper/dists/gradle-6.1.1-all/cfmwm155h49vnt3hynmlrsdst
    Set executable permissions for: /root/.gradle/wrapper/dists/gradle-6.1.1-all/cfmwm155h49vnt3hynmlrsdst/gradle-6.1.1/bin/gradle

    Welcome to Gradle 6.1.1!

    <SNIP>

    Starting a Gradle Daemon (subsequent builds will be faster)

    > Configure project :pluginproject
    executing ndkBuild
    executing ndkBuild clean

    > Task :pluginproject:ndkClean
    /opt/atlassian/pipelines/agent/build/cmdline-tools/ndk-bundle/build/ndk-build: line 151: file: command not found
    make: Entering directory '/opt/atlassian/pipelines/agent/build/pluginproject/src/main'

    Android NDK: WARNING: APP_PLATFORM android-16 is higher than android:minSdkVersion 1 in ./AndroidManifest.xml. NDK binaries will *not* be compatible with devices older than android-16. See https://android.googlesource.com/platform/ndk/+/master/docs/user/common_problems.md for more information.    
    make: Leaving directory '/opt/atlassian/pipelines/agent/build/pluginproject/src/main'
    make: Entering directory '/opt/atlassian/pipelines/agent/build/pluginproject/src/main'

    ...<SNIP - tests running>...

    make: Leaving directory '/opt/atlassian/pipelines/agent/build/pluginproject/src/main'

    BUILD SUCCESSFUL in 1m 14s
    3 actionable tasks: 1 executed, 2 up-to-date
    ```
&nbsp;
---
## MOAR TIPS ##

![Ya filthy animals](https://media.giphy.com/media/3ofT5Mzq0fk0ts1vig/giphy.gif)


Some bonus Docker commands for people like me who only directly interact with Docker about once a quarter (though it looks like I'll be doing it more often _now_...), and who ALWAYS piss away a nontrivial amount of minutes trying to remember how to start a container that doesn't just quit immediately:

```
# fire up a Docker container that will keep running even once it's "done," like an Ubuntu image or some other OS
# it will stop when you `exit` it
$ docker run -it <image name>:<version> /bin/bash 
# you MUST specify -it the first time you build this container. if it's already built, and you try to run it with -it, it'll get confused.

# find it again in the future - it will have all your files, but unless you put your variable declarations in your .bash_rc, you'll lose those
$ docker container ls -a

# Log into it again:
$ docker start <container id>
$ docker attach <container id>

# be advised that you may have to hit Enter/Return once you've done "docker attach" or the interface won't look like it's done anything. 
# got me good a couple times.
```
