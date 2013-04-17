An rsync-based snapshotting script. Includes a handy-dandy rotation script as well.

Introduction
------------

`backup` helps me take browseable, Time Machine&trade;-like snapshots of important stuff on my Linux and Mac boxes. It can also do pure incrementals, has basic error handling, can email you reports, and maintains a simple catalog of all operations.

`prune` helps me maintain the snapshots generated by `backup` according to some retention policy.

Both scripts were written during a more innocent time and could use a rewrite, but they've worked well for many years :)

Prerequisites
-------------

You'll need the `bash` interpreter, `rsync`, and `mail` commands to use the scripts[1]. Untested on Cygwin. 

Usage
-----

Basic usage is quite straightforward:

    ./backup -s /source -d /destination

`/destination` must exist beforehand. By default,

* The script operates in snapshot mode
* *All* files in source are backed up
* A backup report is sent to `STDOUT`

Once the script's done, this is what the backup folder will look like:

    /destination
    ├── 2013-04-08T13.48.47
    │   └── source
    ├── latest -> 2013-04-08T13.48.47
    └── logs
        ├── backuplog.2013-04-08T13.48.47.full.gz
        ├── backuplog.2013-04-08T13.48.47.gz
        └── catalog

A bit 'noisy', but let me explain:

* The backup source is placed inside an ISO8601-timestamped folder[2]. It's named after when the backup *starts*.
* You can access the most recent snapshot with `/destination/latest`
* All transfer logs are written to `/destination/logs`. The "`full`" log contains the record of individual transfers[3]. 
* `/destination/logs/catalog` contains a record of all passed *and* failed backup operations. 
* A simple process lock is maintained for the duration of the backup. It's called `/tmp/backuplock`. This is for the *whole system*. That is, you cannot use multiple instances of this script to snapshot multiple sources in parallel[4].

If you ran that `backup` command again, this is what `/destination` would look like:

    /destination
    ├── 2013-04-08T13.48.47
    │   └── source
    ├── 2013-04-08T14.07.03
    │   └── source
    ├── latest -> 2013-04-08T14.07.03
    └── logs
        ├── backuplog.2013-04-08T13.48.47.full.gz
        ├── backuplog.2013-04-08T13.48.47.gz
        ├── backuplog.2013-04-08T14.07.03.full.gz
        ├── backuplog.2013-04-08T14.07.03.gz
        └── catalog

A few more things have happened:

* The `latest` symlink now points to the most recent timestamped folder
* You have two additional log files in `logs`
* The `catalog` file has been updated.

### The Catalog

For both the operations above, here's what `catalog` looks like:

    Operation      Started              Finished
    full           2013-04-08T13.48.47  2013-04-08T13.48.49  
    snap           2013-04-08T14.07.03  2013-04-08T14.07.03

The first one says "`full`" since it was the first *full* backup. Yeah.

### Backup modes

Whichever mode you use, the script will `sync` if it's your *first* backup. Subsequent hard-linked snapshots and incrementals don't make sense if there's nothing to compare them to.

#### Snapshot (`-t snap`)

This is default, although you can specify it explicitly like so

    ./backup -t snap -s /source -d /destination

#### Sync (`-t sync`)

Synchronizes the source and destination. Nothing too fancy, except that the timestamped target of the `latest` symlink is updated to match the new backup time.

    ./backup -t sync -s /source -d /destination

#### Incremental (`-t diff`)

Only copies files which have *changed from the last snapshot or sync*. I've found this very useful in a few situations. 

    ./backup -t diff -s /source -d /destination

### Filters

Use `-i` to specify an includes file, `-e` for an excludes file. I mostly just use excludes[5]. For example:

    .DS_Store*
    ._*
    .Spotlight-V100
    .Trashes
    ehthumbs.db
    Thumbs.db

### Rotation & Maintenance

Easy as pie! Just run the `prune` script against the backup destination. For example, if I wanted to keep 7 most recent snapshots in `/destination`,

    ./prune -d /destination -n 7

The number of snapshots is specified with `-n`. It's a default of 10 if (a) you don't specify it, or (b) you're silly and specify a non-integer value.

After removing older snapshots, three things then happen:

* the `logs` folder gets cleaned of older logs
* a rotation log called `rotatelog-{timestamp}.gz` file is written to `/destination/logs`
* finally, the `catalog` is updated

In the catalog, you'll see an entry like this

    rote ( 12,  7) 2013-04-08T15.48.53  2013-04-08T15.49.06

This means that the `rotate` script found 12 snapshots (or incrementals) and was asked to *keep* 7. The left and right timestamps correspond to when the rotation started and finished respectively.

I usually run both scripts as `cron` jobs. 

### Other options

If you use the `-q` option, you won't see a report.If you'd like to get email, specify an address with `-m`. In either case, the script will send everything into the scary void that is `/dev/null`.

### Error-handling

You're notified of errors in two cases:

1. The `rsync` command has a non-zero exit status. You'll be told what the status is, and [what it means](http://wpkg.org/Rsync_exit_codes).
2. If the process is interrupted for any reason (e.g. you hit `Ctrl + c` on your keyboard).

When you run the script, and if you're doing a `sync` or `diff`, a timestamped snapshot folder is created with the suffix "`.incomplete`". It's only after a successful backup that the suffix is removed, and the `latest` symlink changed.

For example, here's a listing of some incomplete snapshots:

    /destination
    ├── 2013-04-08T13.48.47
    ├── 2013-04-08T13.51.14.incomplete
    ├── 2013-04-08T14.07.03
    ├── 2013-04-08T15.10.11.incomplete
    └── latest -> 2013-04-08T14.07.03

The `catalog` looks like this

    Operation      Started              Finished
    full           2013-04-08T13.48.47  2013-04-08T13.48.49  
    snap           2013-04-08T13.51.14  No
    snap           2013-04-08T14.07.03  2013-04-08T14.07.03
    snap           2013-04-08T15.10.11  No

**Note**: `rsync` will exit non-zero if a few files have vanished before it could transfer them. I consider this as normal, and the script doesn't complain about it either.

Log files from erroneous `backup` or `prune` runs can be seen in `/tmp`

Future Work
-----------

* Rewrite the whole thing. Properly this time ;)
* A Python port of both scripts for lulz.
* Think about a Windows version.
* Think about how noone will ever use this except myself, so I should leave it alone and go for a walk.

License
-------

MIT

Footnotes
---------

1. If you're on a Mac, I highly recommend [using](http://mxcl.github.io/homebrew/) `brew` to install the latest version of `rsync`.
2. [Not *really*](http://ss64.com/dates.html), since I'd have to use colons instead of periods. This is icky since you'd have to escape them.<br /> So, although my naming isn't 100% compliant, I wanted the folders to sort well and  not have characters that would require escaping or double-quotes. If you have a better nomenclature, let me know.
3. Basically the output of using the `--progress` flag with `rsync`
4. I realize it can be ridiculous, but it works for me. Do fork and improve!
5. See the "*Multiple files and folders*" section of [this document](http://articles.slicehost.com/2007/10/10/rsync-exclude-files-and-folders)
