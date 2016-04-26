# my-kudu-setup
Using Ubuntu to build and test Kudu

## Install Ubuntu 15.10

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
