---
title: "Redis Slowlog"
date: 2022-09-10T11:40:00+05:45
tags: ["redis", "slowlog"]
author: "thapabishwa"
summary: "Track and log all Redis queries that exceed the allocated execution time"
---

# What is Redis `SLOWLOG`?
The Redis `SLOWLOG` command configures and displays the content of a log of slow queries detected by Redis. On most Redis deployments, you're able to get this log from the Redis command line. If you are experiencing slow query execution or high CPU usage in your Redis server, this is the tool to use.

It's worth remembering that slow logs are concerned with only the execution time of the command. The I/O involved in processing the command itself is not measured, only the time spent doing the actual work of the command. The I/O is multithreaded so it can take place alongside other work, but the execution happens on a single thread, blocking other commands from running, which is why it's the critical thing to measure from the database's point of view. Another thing to keep in mind is that the Slow log is transient; there's no persistence for it so in the case of failover, the slow log is lost. If you are looking to rely on a persistent slow log, you'll be wanting to reconsider your design choices.

# How does `SLOWLOG` work?
Once a query is issued, the server keeps track of the time elapsed when executing the command. If the command exceeds the allocated time, it is logged using the slowlog system. There are two configurable settings, `slow-log-slower-than` and `slowlog-max-len`. These directives are used to filter slow events and control the size of the slowlog queue respectively.

## slowlog-log-slower-than
The other way to capture that tricky slow event is to filter out all the slow, but not that slow, log events. That's where `slowlog-log-slower-than` comes in. It sets the threshold on what qualifies as a slow event.

The default value is 10000 microseconds, and when you are trying to track down a slow event you should begin with this default. You can reduce the value all the way down to 0 if you want - as a result everything will be logged as it all takes more than zero microseconds. To disable slow logging completely set it to -1.

If you are trying to track down a particularly slow event, it's then worth raising the `slowlog-log-slower-than` to isolate those events in the log. In the example below, it is set to 10 to catch more slow queries.

## slowlog-max-len
The slow log is actually a queue of slow log events. You can control the size of that queue with `slowlog-max-len`. The bigger you make it, the more memory it consumes. Ideally, it should be big enough for you to catch your problematic slow commands, but not so big that it becomes an issue itself. The default is 128 and you should start with that until you are certain you need to increase it.

# `SLOWLOG` in action

## Fetching the parameters
Using the config set command, you can also configure the slowlog parameters at runtime.

{{< highlight text >}} 
127.0.0.1:6379> config get slow*
1) "slowlog-log-slower-than"
2) "10000"
3) "slowlog-max-len"
4) "128"
{{< /highlight >}}

## Modifying the parameters
Using the config set command, you can configure the log parameters at runtime.
{{< highlight text >}} 
127.0.0.1:6379> config set slowlog-log-slower-than 10
OK

127.0.0.1:6379> config set slowlog-max-len 100
OK
{{< /highlight >}}

## Fetching `SLOWLOG` Entries

To fetch all the entries in the Redis Slow log, run the `SLOWLOG GET` command. 

### Syntax
{{< highlight text >}} 
redis 127.0.0.1:6379> SLOWLOG subcommand [argument] 
{{< /highlight >}}

###  Sample Output
{{< highlight text >}} 
127.0.0.1:6379> slowlog get
 1) 1) (integer) 2         
    2) (integer) 1662782518
    3) (integer) 17
    4) 1) "exists"
       2) "keyabc"
    5) "127.0.0.1:54172"
    6) ""

 2) 1) (integer) 1
    2) (integer) 1662782520
    3) (integer) 155
    4) 1) "expire"
       2) "keyxyz"
       3) "0"
    5) "127.0.0.1:54172"
    6) ""
{{< /highlight >}}

## Components of a log entry

Each Slow Log entry is comprised of 6 main parts.

{{< highlight text >}} 
1) (integer) 1              # unique identifier for the log entry.
2) (integer) 1662782518     # unix timestamp denoting the time at which the entry was added
3) (integer) 17             # time taken by the query in microseconds.
4) 1) "exists"              # array of the command used by the client and arguments passed
   2) "keyabc"
5) "127.0.0.1:54172"        # client address and port that issued the command.
6) ""                       # client name as specified by the client setname command.
{{< /highlight >}}

## Resetting `SLOWLOG` entries
To clean up the slow log entries, use the SLOWLOG RESET command.
{{< highlight text >}} 
127.0.0.1:6379> slowlog reset
OK
{{< /highlight >}}

<!---
# References
https://www.tutorialspoint.com/redis/server_showlog.htm
https://linuxhint.com/redis-slow-log-commands
https://help.compose.com/docs/redis-slow-logs)
-->