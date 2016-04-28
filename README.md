# my-kudu-setup
Using Ubuntu to build and test Kudu

## Background

The Kudu documentation ([here](http://getkudu.io/docs/installation.html#ubuntu_from_source) and [here](https://github.com/cloudera/kudu)) is pretty good, but I wanted to create a reproducible procedure for building & testing Kudu on Ubuntu. (starting from scratch)

This procedure also includes necessary tweaks for a proxied network environment like we have here at Intel.

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

In All Settings | Brightness & Lock:
* Disable turning screen off when inactive
* Disable lock

In All Settings | Security & Privacy:
* For Files & Applications, disable "record file and application usage", clear usage data
* For Search, disable "Include online search results"

Configure Firefox for blank initial page, not to retain history

Unlock extra icons from launcher

## Configure Shell Environment

Fix keymapping for vi editor:

    echo "set nocompatible" >> $HOME/.vimrc

Configure /etc/apt/apt.conf to use proxy:

    Acquire::http::Proxy "http://proxy-us.intel.com:911";

Configure and test ntp:

    sudo apt-get install ntp ntpdate
    sudo vi /etc/ntp.conf                (set server entry)
    sudo service ntp restart
    ntpdate -vdu (server)                (to debug connectivity)

## Install Software Packages

Install and configure git:

    sudo apt-get install git
    git config --global http.proxy http://proxy-us.intel.com:911
    git config --global user.name “Robert A Dickinson”
    git config --global user.email “robert.a.dickinson@intel.com”
    git config --global branch.autosetuprebase always
    git config --global branch.master.rebase always

Install Kudu required libraries in steps 1 & 2 linked here:
http://getkudu.io/docs/installation.html#ubuntu_from_source

:warning: Install liboauth-dev library, or you'll get a warning when using CMake later ("liboauth not found on system.  Skipping twitter demo") and two automated tests will be skipped.

    sudo apt-get install liboauth-dev

## Update Installed Packages

    sudo apt-get update
    sudo apt-get upgrade
    sudo apt-get autoclean
    sudo apt-get clean
    sudo apt-get autoremove

## Run Kudu Tests

Get the sources:

    git clone https://github.com/cloudera/kudu.git

vi ~/kudu-configure.sh, add:

    #!/bin/bash -e
    export KUDU_HOME=$HOME/kudu
    export PATH=$KUDU_HOME/thirdparty/installed/bin:$PATH

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

:warning: Tests may fail randomly if pid_max is not set. (https://issues.apache.org/jira/browse/KUDU-1334)

    sudo bash -c "echo '32768' > /proc/sys/kernel/pid_max" 

Build and run tests:

    source ~/kudu-configure.sh        (once per terminal session)
    ~/kudu-test.sh                    (rebuild & run all tests)
    ~/kudu-test.sh -R (name)          (run single failing test)

## Benchmarking Kudu Tests

**446 seconds** average for all tests on high-end desktop machine:
* Skylake i7-6700K CPU, 4 cores @ 4GHz
* 16GB RAM (DDR4)
* Intel SSD 540s
* Ubuntu Desktop 15.10 (with all latest updates)

**914 seconds** average for all tests on mid-range server machine:
* Dual Xeon Ivytown CPUs, 16 cores @ 2.8GHz
* 64GB RAM (DDR3)
* 1TB SATA disk
* Ubuntu Server 15.10 (with all latest updates)
