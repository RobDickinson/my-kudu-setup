# my-kudu-setup
Using CLion on Ubuntu for Kudu development

## Getting Started

Start with fresh install of Ubuntu 16.04 (either desktop or server distribution)

Install Kudu required libraries in steps 1 & 2 here:
http://getkudu.io/docs/installation.html#ubuntu_from_source

Install Kudu optional packages:

    sudo apt-get install ccache liboauth-dev

## Running Kudu Tests

**WARNING!** Tests will start failing for hybrid_clock-test and later if NTP is not active and synced.

**WARNING!** Tests may fail randomly if pid_max is not set. (https://issues.apache.org/jira/browse/KUDU-1334)

    sudo bash -c "echo '32768' > /proc/sys/kernel/pid_max" 

Get the sources:

    git clone https://github.com/cloudera/kudu.git

vi ~/kudu-env.sh, add:

    #!/bin/bash -e
    export KUDU_HOME=$HOME/kudu
    export PATH=/usr/lib/ccache:$KUDU_HOME/thirdparty/installed/bin:$PATH

vi ~/kudu-test-debug.sh, add:

    #!/bin/bash -e
    pushd . > /dev/null
    mkdir -p $KUDU_HOME/build/debug
    cd $KUDU_HOME/build/debug
    $KUDU_HOME/thirdparty/build-if-necessary.sh
    cmake ../..
    make -j`nproc`
    http_proxy= https_proxy= HTTP_PROXY= HTTPS_PROXY= ctest "$@"
    popd > /dev/null

**WARNING!** As you can see in the script above, we need http proxy variables defined during the cmake phase (so
third party libraries can be downloaded), but no proxy variables defined while tests are running. I haven't found an easy
way to modify NO_PROXY that works across all automated tests, but the form above works.

Build and run tests:

    source ~/kudu-env.sh             (once per terminal session)
    ~/kudu-test-debug.sh -j4         (rebuild & run tests, 4 at a time)
    ~/kudu-test-debug.sh -R (name)   (run single failing test)

## Developing Kudu with CLion

### Installing CLion

Download and extract tarball to local directory, set CLION_HOME

vi $CLION_HOME/bin/clion64.vmoptions, add:

    -Xms1000m
    -Xmx4000m

sudo vi /etc/sysctl.conf, add:

    fs.inotify.max_user_watches = 524288

Now apply that last change:

    sudo sysctl -p

### Configuring CLion

Start CLion:

    $CLION_HOME/bin/clion.sh

Disable any plugins you don't intend to use, you can always add them back in later.

For Toolchain configuration, use Kudu's CMake executable:
KUDU_HOME/thirdparty/installed/bin/cmake

From Welcome screen, use Configure | Settings
* Appearance & Behavior | System Settings --> disable "Reopen last project on startup"
* Appearance & Behavior | System Settings | HTTP Proxy --> set proxy if you have one
* Click OK to close
* Configure | Check for Update (verify proxy settings are working)

### Creating CLion Project

From Welcome screen, click "Open Project", select KUDU_HOME directory

Click to ignore "Some source files are located outside of CMakeLists.txt directory" when this pops up

From the File menu, select "Power Save Mode" to disable background inspections, clear/ignore warnings about this

Wait for CLion to finish indexing the project files, this might take a while!

### CLion Performance Factors

Use the default JDK embedded with CLion. If you're keeping your CLion version up to date, then you already have a good recent JDK without any extra effort. I doubt that you'll see a performance difference by managing the JDK yourself.

CLion likes extra memory, so recommend you allocate 2-4GB to the CLion JVM. But don't set -Xms/-Xmx to the same value! You want the JVM to find the best performing heap by itself, within the safe range set by -Xms and -Xmx.

Enabling "Power Save Mode" prevents CLion from running automatic code inspections every time a file is opened and every time an open editor window gets focus back. Run code inspections manually from the Code menu when convenient for you instead. CLion has a great option for inspecting all uncommitted files at once, and I love this as an unofficial code review before committing any changes.

CLion will take advantage of multiple CPU cores in parallel. I routinely see 7 of 8 cores saturate when loading the Kudu project. If investing in extra cores is an option, you'll be pleased to see improvements in indexing performance.

Multiple disks can also boost CLion performance. I have a SSD for my boot drive, and a second SSD for my KUDU_HOME directory. When CLion builds caches and indexes, it's reading from one SSD while writing to the other.

## Benchmarking

Sometimes runtime performance is the best indication if things are running right, so here's some numbers to share.

### Machine Configurations

Ivytown:
* 2 x Xeon Ivytown CPU, 16 cores each @ 2.8GHz
* 64GB RAM (DDR3)
* 1 x 1TB SATA disk
* Ubuntu Server 15.10, with all latest updates

Skylake:
* 1 x Skylake i7-6700K CPU, 4 cores @ 4GHz
* 16GB RAM (DDR4)
* 2 x Intel SSD 540s (one for OS, one for KUDU_HOME directory)
* Ubuntu Desktop 15.10, with all latest updates

### Benchmark Times

Test Run | Ivytown | Skylake
------------ | ------------- | ------------- 
Download/rebuild third-party libraries | 40-45 minutes | 20-22 minutes
Run debug tests in series (-j1) | 914 sec | 446 sec
Run debug tests in parallel | 428 sec (-j16) | 199 sec (-j4)
Opening CLion project (cold) | n/a | 258 sec
Opening CLion project (warm) | n/a | 102 sec
