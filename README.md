# Apple Unified Logs Add-on for Splunk

## Table of Contents

- [Introduction](#introduction)
- [Installation](#installation)
- [Usage](#usage)
    - [Collection](#collection)
    - [Conversion](#conversion)
    - [Filtering](#filtering)
- [Reference material](#reference-material)

## Introduction

Apple Unified Logs (AUL) is a common logging system used across iOS, MacOS, tvOS, and watchOS systems.  These logs are stored locally on the system in a binary format, however they may be converted to a parsable format with the `log show` command.  This add-on provides sourcetypes for the default and json output formats of this command.

## Installation

1. Install this TA under `$SPLUNK_HOME/etc/apps/`.

## Usage

This add-on provides two sourcetypes:

1. `aul`, for the output of `log show` e.g.

    ```
    2023-07-01 20:44:27.214238+1000 0x24314    Default     0xa724b              631    0    cloudd: (CFNetwork) [com.apple.CFNetwork:ATS] Task <DB0FD7AC-C006-431E-9118-20771B476850>.<27> {strength 0, tls 4, sub 0, sig 1, ciphers 0, bundle 1, builtin 0}
    ```


2. `aul:json`, for the output of `log show --style json` e.g.

    ```json
    {
    "traceID" : 593327990818209796,
    "eventMessage" : "Task <649CB0DA-1B1A-438C-BB54-DCBF2B5AD33F>.<12> RELEASING power assertion.",
    "eventType" : "logEvent",
    "source" : null,
    "formatString" : "%{public}@ RELEASING power assertion.",
    "activityIdentifier" : 0,
    "subsystem" : "",
    "category" : "",
    "threadID" : 14227,
    "senderImageUUID" : "6AAFE7C4-F1C4-3020-AD16-70591C86D7B0",
    "backtrace" : {
        "frames" : [
        {
            "imageOffset" : 57924,
            "imageUUID" : "6AAFE7C4-F1C4-3020-AD16-70591C86D7B0"
        }
        ]
    },
    "bootUUID" : "05E2BF6D-3612-4209-8E97-E09983BCB28F",
    "processImagePath" : "\/System\/Library\/PrivateFrameworks\/CloudKitDaemon.framework\/Support\/cloudd",
    "timestamp" : "2023-06-30 18:11:29.650678+1000",
    "senderImagePath" : "\/System\/Library\/Frameworks\/CFNetwork.framework\/CFNetwork",
    "machTimestamp" : 14792988353,
    "messageType" : "Default",
    "processImageUUID" : "837A08CC-16FB-3CC7-A2F0-C6D138C046ED",
    "processID" : 146,
    "senderProgramCounter" : 57924,
    "parentActivityIdentifier" : 0,
    "timezoneName" : ""
    }
    ```

### Collection

 Log archives can be collected in one of three ways:

1. With the `log collect` command, either locally or from a connected device.
2. Directly from the filesystem, as an amalgam of two folders:

    ```bash
    mkdir ./system_logs.logarchive
    cp /private/var/db/diagnsotics/ ./system_logs.logarchive/
    cp /private/var/db/uuidtext/ ./system_logs.logarchive/
    ```

3. (on iOS devices) with the [Sysdiagnose process](https://it-training.apple.com/tutorials/support/sup075#Run-Sysdiagnose-and-Find-the-Log-File).

### Conversion

The `log show` command (available on macOS devices) may be used to show the contents of a log archive and filter the logs contained therein.  Full usage instructions for this command can be viewed with `log help show`, however the two most common uses will be:

1. Converting the archive to the default output format, with all optional message types enabled.

    ```bash
    log show --backtrace --debug --info --loss --signpost --timezone utc system_logs.logarchive > system_logs.default
    ```

2. Converthing the archive to the json output format, with all optional message types enabled.

    ```bash
    log show --backtrace --debug --info --loss --signpost --timezone utc --style json system_logs.logarchive > system_logs.json
    ```

In both cases, the output is redirected to a file - which may then be ingested into Splunk using one of the appropriate sourcetyle (`aul` or `aul:json` respectively).  Both sourcetypes have their merits, however `aul` typically requires 1/4 of the storage as `aul:json`, and presents the processes and libraries in a format which is easier to query.

### Filtering

The `log show` command also includes an option for filtering events with a _predicate_.  Predicates follow the **NSPredicate** format as outlined in [Apple's developer documentation](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/Predicates/AdditionalChapters/Introduction.html).  Valid predicate fields can be displayed with the `log help predicates` command.

Field extractions within the `aul` sourcetype have been named in line with predicate fields such that any predicates which you are familiar with can be translated to SPL with minimal change. For example:

This `log show` command:

```bash
log show system_logs.logarchive --predicate 'process == "mediaserverd" AND subsystem == "com.apple.coreaudio" AND category == "as_server" AND composedMessage CONTAINS "create_session"'
```

Is equivalent to this SPL:

```
sourcetype=aul source="system_logs.default" process="mediaserverd" subsystem="com.apple.coreaudio" category="as_server" "create_session"
```

## Reference material

There isn't a significant amount of reference material for the use of Apple Unified Logs, however the following sources may server as a starting point:

1. [Crowdstrike Blog - Finding Waldo: Leveraging the Apple Unified Log for Incident Response.](https://www.crowdstrike.com/blog/how-to-leverage-apple-unified-log-for-incident-response/)
2. [Mandiant Blog - Reviewing macOS Unified Logs](https://www.mandiant.com/resources/blog/reviewing-macos-unified-logs)
3. [mac4n6 Blog - Introducing 'Analysis of Apple Unified Logs: Quarantine Edition' [Entry 0]](http://www.mac4n6.com/blog/2020/4/19/introducing-analysis-of-apple-unified-logs-quarantine-edition-entry-0).  This is the start of a 12-part series which walks through some basic use cases for unified logs.
4. [Bsides Presentation - Logs Unite! - Forensic Analysis of Apple Unified Logs](https://github.com/mac4n6/Presentations/blob/master/Logs%20Unite!%20-%20Forensic%20Analysis%20of%20Apple%20Unified%20Logs/LogsUnite.pdf).  This is by the author of the mac4n6 Blog and is the single best resource currently available online for understanding and analysing Apple Unified Logs.