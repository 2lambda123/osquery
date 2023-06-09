# Logging osquery

The osquery daemon uses a default **filesystem** logger plugin. Like the config, output from the filesystem plugin is written as JSON. Results from the query schedule are written to `/var/log/osquery/osqueryd.results.log`.

There are two types of logs:

- Status logs (`INFO`, `WARNING`, `ERROR`)
- Query schedule results logs, including logs from snapshot queries

If you run `osqueryd` in verbose mode, then peek at `/var/log/osquery/`:

```bash
$ ls -l /var/log/osquery/
total 24
lrwxr-xr-x   1 root  wheel    77 Sep 30 17:37 osqueryd.INFO -> osqueryd.INFO.20140930
-rw-------   1 root  wheel  1226 Sep 30 17:37 osqueryd.INFO.20140930
-rw-------   1 root  wheel   388 Sep 30 17:37 osqueryd.results.log
```

On Windows the osquery log directory defaults to `C:\Program Files\osquery\log`.

## Logger plugins

osquery includes logger plugins that support configurable logging to a variety of interfaces. The built in logger plugins are **filesystem** (default), **tls**, **syslog** (for POSIX), **windows_event_log** (for Windows), **kinesis**, **firehose**, and **kafka_producer**. Multiple logger plugins may be used simultaneously, effectively copying logs to each interface. To enable multiple loggers set the `--logger_plugin` option to a comma separated list, do not include spaces, of the requested plugins.

For information on configuring logger plugins, see [logging/results flags](../installation/cli-flags.md#loggingresults-flags). Developing new logger plugins is explored in the [development docs](../development/logger-plugins.md). We recommend setting the logger plugin and logger settings via the osquery *flagfile*.

### Status logs

Status logs are generated by the [Glog logging framework](https://github.com/google/glog/). The default **filesystem** logger plugin writes these logs to disk the same way Glog would. Logger plugins may intercept these status logs and write them to system or otherwise.

As the above directory listing reveals, `osqueryd.INFO` is a symlink to the most recent execution's `INFO` log. The same is true for the `WARNING`, `ERROR` and `FATAL` logs. For more information on the format of Glog logs, please refer to the [Glog documentation](https://github.com/google/glog/blob/master/README.rst).

Note: The `osqueryi` shell only shows `WARNING` and `ERROR` status logs, the `INFO` logs are silenced for a better shell-like experience.

By default the `osqueryd` daemon sends `INFO`, `WARNING`, and `ERROR` logs to the configured logger plugins and to the process's `stderr`. You may configure this behavior using several flags documented in [CLI flags](../installation/cli-flags.md).

- To disable writing status logs to stderr, use `--logger_stderr=false`
- To set the minimum status log severity (`INFO=0`) written to stderr, use `--logger_min_stderr=0`
- To set the minimum status log written to stderr and logger plugins, use `--logger_min_status=0`

Note: In the LaunchDaemon, systemd, and initscript provided in the osquery packages, the minimum `stderr` reporting is limited to `WARNING` to help minimize the content duplicated to syslog.

### Results logs

#### Differential logs

The results of your scheduled queries are logged to the "results log". These are differential changes between the last (most recent) query execution and the current execution. Each log line is a JSON string that indicates what data has been added/removed by which query. The first time the query is executed (there is no "last" run), the last run is treated as having null results, so the differential consists entirely of log lines with the added indication. There are two format options, *single*, or event, and *batched*. Some queries do not make sense to log "removed" events like:

```sql
SELECT i.*, p.resident_size, p.user_time, p.system_time, t.minutes AS c
  FROM osquery_info i, processes p, time t
  WHERE p.pid = i.pid;
```

By adding an outer join of `time` and using `time.minutes` as a counter, this query will always log a single `"added"` and a single `"removed"` line. The purpose is to create a continuous monitor of osquery's performance. For these cases add a `"removed": false` to the scheduled query.

```json
{
  "schedule": {
    "osquery_monitor": {
      "query": "SELECT ... t.minutes AS c FROM time t WHERE ...",
      "interval": 60,
      "removed": false
    }
  }
}
```

#### Snapshot logs

Snapshot logs are an alternate form of query result logging. A snapshot is an 'exact point in time' set of results, with no differentials. For instance if you always want a *list* of mounts, not the *added and removed* mounts, then use a snapshot. In the mounts case, where differential results are seldom emitted (assuming hosts do not often mount and unmount), a complete snapshot will log after every query execution. This *will* be a lot of data amortized across your fleet.

Data snapshots may generate *a large amount* of output. For log collection safety, output is written to a dedicated sink. The **filesystem** logger plugin writes snapshot results to `/var/log/osquery/osqueryd.snapshots.log`.

To schedule a snapshot query, use:

```json
{
  "schedule": {
    "mounts": {
      "query": "SELECT * FROM mounts;",
      "interval": 3600,
      "snapshot": true
    }
  }
}
```

### Logging as a Kafka producer

Users can configure logs to be directly published to a Kafka topic.

#### Configuration

There are 3 Kafka configurations exposed as options: a comma-delimited list of brokers with or without the port (by default `9092`) [default value: `localhost`], a default topic [default value: `""`], and acks (the number acknowledgments the logger requires from the Kafka leader before the considering the request complete) [default: `all`; valid values: `0`, `1`, `all`]. [See the official Kafka documentation for more details.](https://kafka.apache.org/documentation/#producerconfigs)

To publish queries to specific topics, add a `kafka_topics` field at the top level of `osquery.conf` (see example below). If a given query was not explicitly configured in `kafka_topics` then the base topic will be used.  If there is no base topic configured, then that query will not be logged. There is however a performance cost for the falling back of unconfigured queries to the base topic, so it is advised that when using multiple topics to explicitly configure all scheduled queries in `kafka_topics`.

The configuration parameters are exposed via command-line options and can be set in a JSON configuration file:

```json
{
  "options": {
    "logger_kafka_brokers": "some.example1.com:9092,some.example2.com:9092",
    "logger_kafka_topic": "base_topic",
    "logger_kafka_compression": "gzip",
    "logger_kafka_acks": "1"
  },
  "packs": {
    "system-snapshot": {
      "queries": {
        "some_query1": {
          "query": "select * from system_info",
          "snapshot": true,
          "interval": 60
        },
        "some_query2": {
          "query": "select * from md_devices",
          "snapshot": true,
          "interval": 60
        },
        "some_query3": {
          "query": "select * from md_drives",
          "snapshot": true,
          "interval": 60
        }
      }
    }
  },
  "kafka_topics": {
    "test1_topic": [
      "pack_system-snapshot_some_query1"
    ],
    "test2_topic": [
      "pack_system-snapshot_some_query2"
    ],
    "test3_topic": [
      "pack_system-snapshot_some_query3"
    ],
  }
}
```

Client ID and msg key used are a concatenation of the OS hostname and binary name (`argv[0]`).  Currently there can only be one topic passed into the configuration, so all logs will be published to the same topic.

## Schedule results

### Event format

Event is the default result format. Each log line represents a state change. This format works best for log aggregation systems like Logstash or Splunk.

Example output of `SELECT name, path, pid FROM processes;` (whitespace added for readability):

```json
{
  "action": "added",
  "columns": {
    "name": "osqueryd",
    "path": "/opt/osquery/bin/osqueryd",
    "pid": "97830"
  },
  "name": "processes",
  "hostname": "hostname.local",
  "calendarTime": "Tue Sep 30 17:37:30 2014",
  "unixTime": "1412123850",
  "epoch": "314159265",
  "counter": "1",
  "numerics": false
}
```

```json
{
  "action": "removed",
  "columns": {
    "name": "osqueryd",
    "path": "/opt/osquery/bin/osqueryd",
    "pid": "97650"
  },
  "name": "processes",
  "hostname": "hostname.local",
  "calendarTime": "Tue Sep 30 17:37:30 2014",
  "unixTime": "1412123850",
  "epoch": "314159265",
  "counter": "1",
  "numerics": false
}
```

This tells us that a binary called `osqueryd` was stopped and a new binary with the same name was started (note the different `pid`). The data is generated by keeping a cache of previous query results and only logging when the cache changes. If no new processes are started or stopped, the query won't log any results.

### Snapshot format

Snapshot queries attempt to mimic the differential event format, instead of emitting "columns", the snapshot data is stored using "snapshot". An action is included as, you guessed it, "snapshot"!

Consider the following example:

```json
{
  "action": "snapshot",
  "snapshot": [
    {
      "parent": "0",
      "path": "/sbin/launchd",
      "pid": "1"
    },
    {
      "parent": "1",
      "path": "/usr/sbin/syslogd",
      "pid": "51"
    },
    {
      "parent": "1",
      "path": "/usr/libexec/UserEventAgent",
      "pid": "52"
    },
    {
      "parent": "1",
      "path": "/usr/libexec/kextd",
      "pid": "54"
    }
  ],
  "name": "process_snapshot",
  "hostIdentifier": "hostname.local",
  "calendarTime": "Mon May  2 22:27:32 2016 UTC",
  "unixTime": "1462228052",
  "epoch": "314159265",
  "counter": "1",
  "numerics": false
}
```

### Batch format

If a query identifies multiple state changes, the batched format will include all results in a single log line. If you're programmatically parsing lines and loading them into a backend datastore, this is probably the best solution.

To enable batch log lines, launch osqueryd with the `--logger_event_type=false` argument.

Example output of `SELECT name, path, pid FROM processes;` (whitespace added for readability):

```json
{
  "diffResults": {
    "added": [
      {
        "name": "osqueryd",
        "path": "/opt/osquery/bin/osqueryd",
        "pid": "97830"
      }
    ],
    "removed": [
      {
        "name": "osqueryd",
        "path": "/opt/osquery/bin/osqueryd",
        "pid": "97650"
      }
    ]
  },
  "name": "processes",
  "hostname": "hostname.local",
  "calendarTime": "Tue Sep 30 17:37:30 2014",
  "unixTime": "1412123850",
  "epoch": "314159265",
  "counter": "1",
  "numerics": false
}
```

Most of the time the **Event format** is the most appropriate. The next section in the deployment guide describes [log aggregation](log-aggregation.md) methods. The aggregation methods describe collecting, searching, and alerting on the results from a query schedule.

## Special top-level fields

### Schedule epoch

When [differential logs](#differential-logs) were described above, we mentioned that after the initial execution of a scheduled query, only differential results are logged. While this is very efficient from a size-of-logs perspective, it introduces some challenges. To begin with, if the logs are stored in a log management system of some kind, it becomes difficult or impossible to identify which log results are from the initial run of the query, and which ones are differentials to the initial results. In some situations, this becomes problematic. For example, for some tables like the `users` table that don't change very often and thus don't generate differential results very often, one would have to search far into historical logs to find the last results returned by osquery. Conversely, for tables like `processes` that change frequently, one would have to do a fair amount of logic, applying the effects of added and removed rows to reconstruct the current state of running processes.

To aid with this, osquery maintains an `epoch` marker along with each scheduled query execution, and calculates differentials only if the epoch of the last run matches the current epoch. If it doesn't, then it treats the current execution of the query as an initial run. You can set the epoch marker by starting osquery with the `--schedule_epoch=<some 64bit int>` flag or by updating the `schedule_epoch` flag remotely from a TLS backend. The epoch is transmitted with each log result, so that it is easy to identify which results belong to which execution of the scheduled query.

### Schedule counter

When setting up alerts for [differential logs](#differential-logs) data you might want to skip the initial `added` records. `counter` can be used to identify if the added records are all records from initial query or if they are new records. For initial query results that include all records counter will be **"0"**, while initial results without all records (like event tables) will start at **"1"**. For subsequent query executions counter will be incremented by **1**. When `epoch` changes, counter will be reset back to the initial query state.

### Numerics

This is an indicator for all results, `true` if osquery attempted to log numerics as numbers, otherwise `false` indicates they were logged as strings.

### Unique host identification

If you need a way to uniquely identify hosts embedded into `osqueryd`'s results log, then the `--host_identifier` flag is what you're looking for.
By default, **host_identifier** is set to "hostname". The host's hostname will be used as the host identifier in results logs. If hostnames are not unique or consistent in your environment, you can launch osqueryd with `--host_identifier=uuid`.
