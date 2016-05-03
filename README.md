# my-kudu-setup
Using Ubuntu and CLion for Kudu development

## Background

The Kudu documentation ([here](http://getkudu.io/docs/installation.html#ubuntu_from_source) and [here](https://github.com/cloudera/kudu)) is quite good, but I'm attempting to capture all steps to configure Ubuntu for Kudu development in one comphrensive and easily reproducible procedure.

Highlights of this procedure include:
* Configuring Ubuntu OS and Kudu build for required HTTP proxy (like we have at Intel)
* Pitfalls to avoid in running [Kudu automated tests](https://github.com/RobDickinson/my-kudu-setup#run-kudu-tests)
* [Configuring CLion](https://github.com/RobDickinson/my-kudu-setup#developing-kudu-with-clion) for Kudu development
* [Benchmark results](https://github.com/RobDickinson/my-kudu-setup#benchmarking) for long-running operations

## Install Ubuntu 15.10

* Use either Desktop or Server version (both tested)
* Install from USB drive
* Skip download updates while installing
* Skip installing MP3 third-party software
* Use default partitioning scheme (ext4 by default)

## Configure Network Proxy

### For Desktop

In All Settings | Network | Network Proxy:
* Method = Manual
* HTTP Proxy = proxy-us.intel.com:911
* HTTPS Proxy = proxy-us.intel.com:911

### For Server

vi /etc/environment, add:

    http_proxy="http://proxy-us.intel.com:911"
    https_proxy="http://proxy-us.intel.com:911"
    HTTP_PROXY="http://proxy-us.intel.com:911"
    HTTPS_PROXY="http://proxy-us.intel.com:911"
    no_proxy="localhost,127.0.0.0/8,::1"
    NO_PROXY="localhost,127.0.0.0/8,::1"

## Configure Desktop Environment

Download Intel graphics installer for linux, then run using proxy:

    sudo http_proxy=$http_proxy intel-graphics-linux-installer

In All Settings | Security & Privacy:
* For Files & Applications, disable "record file and application usage", clear usage data
* For Search, disable "Include online search results"

Configure Firefox for blank initial page, not to retain history

## Configure OS Packages

Fix keymapping for vi editor:

    echo "set nocompatible" >> $HOME/.vimrc

Configure /etc/apt/apt.conf to use proxy:

    Acquire::http::Proxy "http://proxy-us.intel.com:911";

Configure and test ntp:

    sudo apt-get install ntp ntpdate
    sudo vi /etc/ntp.conf                (set server entry)
    sudo service ntp restart
    ntpdate -vdu (server)                (to debug connectivity)

Install and configure git:

    sudo apt-get install git
    git config --global http.proxy http://proxy-us.intel.com:911
    git config --global user.name “Robert A Dickinson”
    git config --global user.email “robert.a.dickinson@intel.com”
    git config --global branch.autosetuprebase always
    git config --global branch.master.rebase always

Install Kudu required libraries in steps 1 & 2 linked here:
http://getkudu.io/docs/installation.html#ubuntu_from_source

Install ccache using default configuration:

    sudo apt-get install ccache

## Run Kudu Tests

:warning: Tests may fail randomly if pid_max is not set. (https://issues.apache.org/jira/browse/KUDU-1334)

    sudo bash -c "echo '32768' > /proc/sys/kernel/pid_max" 

:warning: Install liboauth-dev library, or you'll get a CMake warning ("liboauth not found on system.  Skipping twitter demo") and two automated tests will be skipped.

    sudo apt-get install liboauth-dev

Get the sources:

    git clone https://github.com/cloudera/kudu.git

vi ~/kudu-env.sh, add:

    #!/bin/bash -e
    export KUDU_HOME=$HOME/kudu
    export PATH=/usr/lib/ccache:$KUDU_HOME/thirdparty/installed/bin:$PATH

vi ~/kudu-test.sh, add:

    #!/bin/bash -e
    pushd . > /dev/null
    mkdir -p $KUDU_HOME/build/debug
    cd $KUDU_HOME/build/debug
    $KUDU_HOME/thirdparty/build-if-necessary.sh
    cmake ../..
    make -j`nproc`
    http_proxy= https_proxy= HTTP_PROXY= HTTPS_PROXY= ctest "$@"
    popd > /dev/null

:warning: As you can see in the script above, we need http proxy variables defined during the cmake phase (so
third party libraries can be downloaded), but no proxy variables defined while tests are running. I haven't found an easy
way to modify NO_PROXY that works across all automated tests, but the form above works.

Build and run tests:

    source ~/kudu-env.sh          (once per terminal session)
    ~/kudu-test.sh                (rebuild & run all tests)
    ~/kudu-test.sh -R (name)      (run single failing test)

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

Mid-range server machine:
* 40-45 minutes to download/rebuild all Kudu third-party libraries
* 914 seconds to run all Kudu tests
* Dual Xeon Ivytown CPUs, 16 cores @ 2.8GHz
* 64GB RAM (DDR3)
* 1 x 1TB SATA disk
* Ubuntu Server 15.10, with all latest updates
* (did not test CLion, headless server)

High-end workstation machine:
* 20-22 minutes to download/build all Kudu third-party libraries
* 446 seconds to run all Kudu tests
* 258 seconds to open CLion project the first time, 102 seconds subsequently
* Single Skylake i7-6700K CPU, 4 cores @ 4GHz
* 16GB RAM (DDR4)
* 2 x Intel SSD 540s (one for OS, one for KUDU_HOME directory)
* Ubuntu Desktop 15.10, with all latest updates

(all times were averaged over multiple attempts)

## Update Installed Packages

    sudo apt-get update
    sudo apt-get upgrade
    sudo apt-get autoclean
    sudo apt-get clean
    sudo apt-get autoremove
