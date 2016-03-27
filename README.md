# rminions

`rminions` is a package that provides functions to assist in setting up an environment for quickly running jobs across any number of servers. This is accomplished by having a central control server which is in charge of accepting jobs and distributing them among workers. The passing of jobs and results is handled via Redis message queues.

This package also contains functions to allow R-level server maintenance on all servers simultaneously. This is accomplished via the Redis PUB/SUB system.

## Installation

In order to install the `rminions` package you will first need to install the `devtools` package.

```R
install.packages("devtools")
```

From here you can install the `rminions` package directly from repository.

```R
devtools::install_github("PieceMaker/rminions")
```

## Central Server Setup

The central server must have a working [Redis](http://redis.io/) server running that has the ability to accept incoming connections. It is highly recommended that the server be running a version of Linux as this is the primary operating system supported by Redis. If desired, a build of Redis exists for [Windows](https://github.com/MSOpenTech/redis). However, this section will focus on getting Redis running in an Ubuntu environment.
  
Note, in the rest of this document the central server will be referenced as Gru-svr.

First install Redis.

```bash
sudo apt-get install redis-server
```

Once Redis is installed, it will likely only accept connections originating from the localhost. To change this you need to edit the Redis config file located at `/etc/redis/redis.conf`. Navigate to the section that is summarized below:

```
# By default Redis listens for connections for all the network interfaces
# ...
#
# Examples:
#
# bind 192.168.1.100 10.0.0.1
bind 127.0.0.1
```

If you are willing to accept connections originating from all addresses, simply comment out the last line. If you have special requirements for addresses, add a new bind line and specify whatever rules you desire.

Once you have finished updating the IP address rules, save the file and restart the Redis service.

```bash
sudo service redis-server restart
```

To test whether you can access your new Redis server from R, open a new R session from any computer that has network access to Gru-svr and run the following test:

```R
library(rredis)
conn <- redisConnect("Gru-svr", returnRef = T)
redisRPush("testKey", list(x = 1, y = 2))
# [1] "1"
# attr(,"redis string value")
# TRUE
redisRPop("testKey")
# $x
# [1] 1
#
# $y
# [1] 2
redisClose(conn)
```

If the result of your `redisRPop` command was to print the list you pushed, then your server is now working.

# PUB/SUB Server Maintenance

Unlike message queues, which run on a first-come-first-served basis, Redis has a publish/subscribe (PUB/SUB) system available. Whenever a message is published on a channel, any Redis connection that is subscribed to the channel receives the published message. A connection can be subscribed to any number of channels. Since the PUB/SUB messages can be received by any number of connections, this system has been used to allow R-level server maintenance among all connected (and listening) servers simultaneously. The `minionListener` function has been included in the package for this express purpose.

The `minionListener` is written with the idea that each type of maintenance task will be published on its own channel. To subscribe to channels with the `minionListener`, you need to pass in a function defining a channel and how to handle messages when they are received. This handling definition is a function known as a callback and it will be executed with the published message as the input parameter any time a message is broadcast. The `rminions` package comes with three predefined channel functions, all for installing packages on the servers. These are: `cranPackageChannel()`, `gitHubPackageChannel()`, and `gitPackageChannel()`.

To define a custom channel function, it should take a channel name to subscribe to and an error channel name where errors will be published. The first part of the function should then define the callback function that handles the contents of the received message. If you wish to publish errors that occur, then you should put a tryCatch in your callback an, on error, run a function similar to the following:

```R
    error = function(e) {
        e <- sprintf("An error occurred processing job on channel '%s' on listener for server '%s': %s",
            channel,
            listenerHost,
            e
        )
        rredis::redisSetContext(outputConn)
        rredis::redisPublish(errorChannel, e)
        rredis::redisSetContext(subscribeConn)
    }
```

Redis does not allow both publishing and subscribing on the same channel. Thus, the listener creates two connections and they must be switched in order to publish and then switched again in order to continue monitoring the subscribed channel. Currently `outputConn` and `subscribeConn` have been placed in the global environment and will be accessible as long as the channel definition function is being run as part of the `minionListener`. It is a goal for the near future to get away from using the globan environment for connection management and instead pass connections as arguments to the channel definition functions.

Once you have all the channel definition functions you need, you can run the listener. The following example starts the listener and subscribes to the CRAN and GitHub package installation channels.

```R
library(rminions)
minionListener(host = "Gru-svr", channels = list(cranPackageChannel(), gitHubPackageChannel()))
```

Once the listener is running it will block the running process so it is recommended that you background the process or create an Upstart job to begin the listener in your working environment.

To test server maintenance using this system, run the following in a new R process, preferably on a computer or server that is not running the minion listener. This example will install the `data.table` package from CRAN.

```R
library(rredis)
conn <- redisConnect("Gru-svr", returnRef = T)
redisPublish("cranPackage", jsonlite::serializeJSON(packageName = "data.table"))
redisClose(conn)
```

You should see install information in the process running the listener and in the listener log file.

If you wish to monitor any errors that occur while handling published messages, subscribe to the error channel(s) that were used in the channel definitions. The following will monitor the default error channel.

```R
library(rredis)
conn <- redisConnect("Gru-svr", returnRef = T)
errorChannel <- function(message) {
    print(message)
}
redisSubscribe("errorChannel")
while(1) {
    redisMonitorChannels()
}
```