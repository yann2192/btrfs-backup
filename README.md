![Image](https://raw.githubusercontent.com/3coma3/btrfs-backup/master/image.png)

# btrfs-backup
I know there are already a million Btrfs/Rsync scripts, I know. I checked them out. Every tool I tested was made to manage snapshots / backups for native btrfs filesystems, or had too many impositions/assumptions about how you should structure your backups: you must name or initialize your destinations in some specific way, or cron the executions in another way, or learn yet another config file syntax. Others were too restrictive in the granularity and retention of snapshots, for example: you can have only N yearly backups, or X monthly, etc. And then others had dependencies like a database in which to store metadata, or some extra language interpreter that you needed to understand, etc.

I wanted something lean and mean, that does the job and gets out of the way, and has minimal dependencies. I needed an easy to navigate snapshot structure, and I wanted a retention / rotation scheme.  I wanted ease of use and above everything, ease of understanding, so anyone can modify it.

So I wrote this little piece, and I'm pretty satisfied with the results.

## key features

* Plain bash. No DSL, no complex parameters, no nothing.
* ~50 lines of code omitting config, comments and blank lines.
* 0 impositions. It may be used as needed, manually or via cron. It does its job and gets out of the way.
* Does Incremental, in place, COW powered snapshots out of any file system Rsync can read, courtesy of Btrfs.
* Custom GFS-style rotation algorithm to maintain arbitrary copies at the snapshot, day, week, month and year levels.
* Compact data source specification in extended bash "glob" [pattern syntax](https://mywiki.wooledge.org/glob)
* Compact per-data-source filter specification in rsync's own [pattern syntax](http://man7.org/linux/man-pages/man1/rsync.1.html#FILTER_RULES) (currently testing)
* Dry-run mode.



## how to use it

### Installation

The script is contained in the file `backup` in this repo. You'll want to give it proper ownership, execution permissions, and have it in your $PATH (or invoke it with some path, as you prefer).



### Configuration

Edit the script between the lines that read `CONFIG START` and ```CONFIG END```. Tune these four variables:

```sources```

Each entry in this bash array is an bash "glob" path expansion expression. Examples of valid glob expressions are:

**/home**

Simple paths are valid globs. This would include the /home directory in the list of files to backup.

**/home/***

This would include every subdirectory or file under /home.

**/home/user/{b,d,m,p,s,t,v,w}***

This would include any file or subdirectory under /home/user, that starts with the letters b, d, m, p, s, t, v or w.

**/home/user/{bash,cinnamon,config,dmrc,fzf,g{conf,i{mp,t},n{ome,upg},tkrc},icons,l{inuxmint,ocal},m{ozilla,ysql},p{rofile,utty},ssh,th{emes,underbird},vim,znc}**

This is a complex expression that illustrates nesting.

Full documentation of the syntax is available in the bash man page, or easily found around the web. Similar examples are already included in the script (these are in fact the paths I regularly make backups of).

```destination```

This is an absolute path to a btrfs volume. The script expects that you don't create the volume, so just specify it here and on the first run it will be created.

`retention`

This associative array describes your *retention policy*. Basically you say how many snapshots you want to keep per discrete snapshot, different day, different week, different month and different year. More on this down this document.

`weekstart`

This parameter is used by the retention policy algorithm and should be set to whatever is the first day of the week where you live. Here the value 1 means monday, and 7 means sunday. Pick anything you want.



###  Invocation

```backup snap```
Will setup the base location for the snapshots if this is the first run, and then diligently rsync the files to the first snapshot. Subsequent invocations will base the new snapshots on the last one available.

```backup clean```
Will scan the snapshot storage, applying your retention policy as expressed in the ```retention``` and ```weekstart``` variables.

You can chain as many invocations of those two subcommands and they will be executed in order:

```backup clean snap```
Will first apply the retention policy and then take a snapshot.

```backup snap clean```
Will do the opposite, first take a snapshot and then "clean" the snapshot repository.

etc.

```backup -d action(s)```
To see what would be done without actually making any modifications. Use it to test your sources and retention policy (although for this last task there's another script in this repository made to that specific effect).



### Backup storage structure
As you take snapshots, each one will be stored in a mixed structure composed of btrfs subvolumes, snapshots and standard directories that'll look like this:

***base***

a btrfs subvolume as specified in the ```destination``` configuration variable

***|--year/month/day/hhmmss***

year/month/day are subdirectories, hhmmss is the snapshot for this backup. All the time related items in the structure are taken from the time you make the snapshot, so the full path to the snapshot identifies it and also *when* you made it.

This also means that a directory representing a year will have at most 12 subdirectories beneath it, a month something between 28 and 31, and a day can have theoretically 86400 snapshots, provided you don't clean it :-)

And since we're talking about cleaning...



### Retention algorithm
The algorithm is based on the classic GFS (Grandfather-Father-Son) scheme, which lets you have some number of items according to some form of nesting or leveling. Here the nesting is given by the subdirectory structure, and the leveling is given by each node in that structure: the directories for years, months and days when backups were made, and the "leaf" folders named with the hour, minute and second when the backups were made.

Actually the leafs aren't subdirectories but btrfs snapshots, but from the user perspective there's little difference when you interact with the tree.

The gist of it is that you tell the script just how many items of each level you want to exist *at the minimum*:

You might want to keep the last 15 snapshots you made, and after that save at least 1 for the last 7 days, and then at least the last one for the last 4 weeks, and so on for the months and years. You then go on making backups, in an ordered or unordered fashion, and when you apply the cleaning procedure the script will try to respect those numbers, and delete everything else, starting from the oldest snapshots available.

If you want to test how it would work for different scenarios, I recommend to use the companion script, aptly named ```test-retention```. You can tune the configuration in the same fashion as for the backup script, and then invoke it as:

```test-retention test N``` where N is the number of snapshots to simulate. It will print out a list of virtual snapshots and how would the configured retention policy affect their deletion.



### Restore procedure
The restore procedure consists of the following steps:

1. Navigate to the desired snapshot
2. Copy the files to wherever you want them to be
