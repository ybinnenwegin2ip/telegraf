# OmniOS ZFS

This page includes instructions on how to compile and run Telegraf on OmniOS 5.11 (omnios-r151030-f1189fc02c). Please note that I have only tested the ZFS input, it is more than likely that a lot of plugins are broken.

## Installing dependencies

First install Golang, Git and GNU-Make:

```bash
pkg install go-111 git gnu-make
```

Then install Golang's `dep`:

```bash
go get -u github.com/golang/dep/cmd/dep
```

([more detailed instructions](https://github.com/golang/dep/blob/master/README.md))

## Setting up the repositories

Now get the source for Telegraf:

```bash
go get -d github.com/influxdata/telegraf
```

And enter its source directory:

```bash
cd "$HOME/go/src/github.com/influxdata/telegraf"
```

Add this repository as a remote:

```bash
git remote add forked https://github.com/ybinnenwegin2ip/telegraf
```

Update the remotes:

```bash
git remote update
```

And checkout the `omnios_zfs` branch:

```
git checkout omnios_zfs
```

## Compiling

Now run `gmake` to compile Telegraf.

## Sample config

I personally use the following Telegraf config to push my ZFS statistics to another Telegraf instance which then inserts it into InfluxDB:

```
# Global tags can be specified here in key="value" format.
[global_tags]
  # dc = "us-east-1" # will tag all metrics with dc=us-east-1
  # rack = "1a"
  ## Environment variables can be used as tags, and throughout the config file
  # user = "$USER"

# Configuration for telegraf agent
[agent]
  ## Default data collection interval for all inputs
  interval = "10s"
  ## Rounds collection interval to 'interval'
  ## ie, if interval="10s" then always collect on :00, :10, :20, etc.
  round_interval = true

  ## Telegraf will send metrics to outputs in batches of at most
  ## metric_batch_size metrics.
  ## This controls the size of writes that Telegraf sends to output plugins.
  metric_batch_size = 1000

  ## Maximum number of unwritten metrics per output.
  metric_buffer_limit = 10000

  ## Collection jitter is used to jitter the collection by a random amount.
  ## Each plugin will sleep for a random time within jitter before collecting.
  ## This can be used to avoid many plugins querying things like sysfs at the
  ## same time, which can have a measurable effect on the system.
  collection_jitter = "0s"

  ## Default flushing interval for all outputs. Maximum flush_interval will be
  ## flush_interval + flush_jitter
  flush_interval = "10s"
  ## Jitter the flush interval by a random amount. This is primarily to avoid
  ## large write spikes for users running a large number of telegraf instances.
  ## ie, a jitter of 5s and interval 10s means flushes will happen every 10-15s
  flush_jitter = "0s"

  ## By default or when set to "0s", precision will be set to the same
  ## timestamp order as the collection interval, with the maximum being 1s.
  ##   ie, when interval = "10s", precision will be "1s"
  ##       when interval = "250ms", precision will be "1ms"
  ## Precision will NOT be used for service inputs. It is up to each individual
  ## service input to set the timestamp at the appropriate precision.
  ## Valid time units are "ns", "us" (or "µs"), "ms", "s".
  precision = ""

  ## Logging configuration:
  ## Run telegraf with debug log messages.
  debug = false
  ## Run telegraf in quiet mode (error log messages only).
  quiet = false
  ## Specify the log file name. The empty string means to log to stderr.
  logfile = ""

  ## Override default hostname, if empty use os.Hostname()
  hostname = ""
  ## If set to true, do no set the "host" tag in the telegraf agent.
  omit_hostname = false

# Configuration for sending metrics to InfluxDB
[[outputs.influxdb]]
  urls = ["http://HOSTNAME:8187"]

# # Read metrics of ZFS from arcstats, zfetchstats, vdev_cache_stats, and pools
[[inputs.zfs]]
  poolMetrics = true
```

And that should be it!

## Deploying

These are some quick deployment instructions:

Run the following commands:
```
useradd telegraf
groupadd telegraf
mkdir /etc/telegraf
```

Copy the telegraf binary to `/bin/` and mark it as executable: `chmod +x /bin/telegraf`.

Set the `hostname` & `urls` in the sample config above and paste it into `/etc/telegraf/telegraf.conf`.

Then edit `/etc/init.d/telegraf` and paste in the following:

```bash
#! /usr/bin/env bash 

USER=telegraf 
GROUP=telegraf 
config="/etc/telegraf/telegraf.conf" 
PID=`ps -e|grep tele|awk '{print $1}'` 
daemon=/usr/bin/telegraf 
logfile="/var/log/telegraf/telegraf.log" 
log_dir="/var/log/telegraf/" 
if [ ! -d $log_dir ] 
    then 
    mkdir -p $log_dir 
    chown -R $USER:$GROUP $log_dir 
fi

case $1 in 
    start) 
        # Checked the PID file exists and check the actual status of process 
        if [ ! -z $PID ]; then 
            echo "process already running" 
            elif [  -z $PID ]; then 
                echo "Starting Telegraf agent" 
                   su  $USER  -c "$daemon  --config $config  >> $logfile 2>&1 &" 

                exit 0 # Exit 
        fi 
        ;; 

    stop) 
        # Stop the daemon. 
        if [ -z $PID ]; then 
            echo "telegraf already stopped" 
        elif [ ! -z $PID ]; then 
            echo "stopping telegraf" 
            kill -9 $PID 
            if [ $? = 0 ] ; then 
                echo "telegraf stopped" 
            else 
                echo "problem in stopping telgraf" 
            fi 
        fi 
        ;; 
        status) 
        if [ ! -z $PID ]; then 
        echo "telegraf is running on PID $PID" 
        else 
        echo "telegraf not running" 
        fi 
        ;; 
        restart) 
        $0 stop 
        $0 start 
;; 
    *) 
        # For invalid arguments, print the usage message. 
        echo "Usage: $0 {start|stop|status|restart}" 
        exit 2 
        ;; 
esac 
```

Then run `chmod +x /etc/init.d/telegraf`. To make it start automatically run `ln -s /etc/init.d/telegraf /etc/rc3.d/S93telegraf`.

