# IserB
**I**nterval and **S**napshot **E**nabled **R**sync **B**ackup

```
○ iserb --help
Usage: iserb [<switches>] [<options>] <backup-list-directory> <backup-destination-directory>

Helper script for LVM snapshots and local rsync backup.

LVM + Rsync

This script streamlines the process of looking at a source directory, marked by
following symlinks to it in <backup-list-directory>, determining if that source
is an LVM mountpoint, snapshoting it if it is, then using rsync to backup files
from that snapshot to <backup-destination-directory>, then cleaning up all
temporary snapshots and mountpoints. To keep backup size down, rysync uses hard links from previous backups to reuse
files on the harddrive that have not changed.

The script has basic resume capability in case of script stoppage or machine
power outage. If no configuration has changed in between attempts, it will
reliably resume the backup regardless of whether the backup source involves LVM
snapshots or just rsync and at any point.

The script also uses a lock file to prevent simultaneous instances (assuming both
scripts have access to the same system '/tmp' directory).

Source Directories

    Source directories to be backed up are marked by full-path symlinks to
    those directories in the <backup-list-directory>.

    Two key assumptions are made about the source directories:

        1) That the target of the symlinks is either a mountpoint for a LVM
        partition or some other normal directory on the root partition, and

        2) That regardless of partition mountpoint or normal directory, it is
        mounted and accessible by root.

    One such example of <backup-list-directory> with two directories marked for
    backup via symlinks would be:

    <backup-list-directory>
    ├── testsource -> /mnt/tmp/testsource
    └── testsource2 -> /mnt/tmp/testsource2

Backup Structure

    This script has a particular folder heirarchy it uses for the backups it
    creates.  Within <backup-destination-directory>, it will organize the
    backups by source (the name derived from their source folder), and then by
    date.

    Given the above source directories and a backup directory location of
    '/mnt/tmp/testdestination', if one were to run one backup of each interval,
    the resulting structure would look like this:

    testdestination
    └── source
        ├── testsource
        │   ├── 2017
        │   │   └── 04
        │   │       ├── 01
        │   │       └── 24-1332
        │   ├── thismonth
        │   │   └── 18
        │   └── today
        │       └── 14
        └── testsource2
            ├── 2017
            │   └── 04
            │       ├── 01
            │       └── 24-1332
            ├── thismonth
            │   └── 18
            └── today
                └── 14

    Optionally, using the '-c|--chrono' switch, you can enable the creation of
    an additional folder hierarchy organized by date first, then by source.
    This is done via symlinks to the originals in the
    '<backup-destination-directory>/source'.

    testdestination
    ├── source
    │   └──[same as above]
    └── date
        ├── 2017
        │   └── 04
        │       ├── 01
        │       │   ├── testsource -> /mnt/tmp/testdestination/source/testsource/2017/04/01
        │       │   └── testsource2 -> /mnt/tmp/testdestination/source/testsource2/2017/04/01
        │       └── 24-1332
        │           ├── testsource -> /mnt/tmp/testdestination/source/testsource/2017/04/24-1332
        │           └── testsource2 -> /mnt/tmp/testdestination/source/testsource2/2017/04/24-1332
        ├── thismonth
        │   └── 18
        │       ├── testsource -> /mnt/tmp/testdestination/source/testsource/thismonth/18
        │       └── testsource2 -> /mnt/tmp/testdestination/source/testsource2/thismonth/18
        └── today
            └── 14
                ├── testsource -> /mnt/tmp/testdestination/source/testsource/today/14
                └── testsource2 -> /mnt/tmp/testdestination/source/testsource2/today/14

Snapshots

    If the script determines that the target of the symlink is a mountpoint, it
    will:

        1) Assume it belongs to the volume group 'lvm_volume_group'

        2) Determine the volume's device path and partition name by searching
           for it among all mounted devices

        3) Snapshot and mount the partition of 'lvm_snapshot_size' in a
           temporary location in 'lvm_snapshot_mountpoint_base'

        4) Syncronize using rsync to a folder the same name as it's source
           partition in <backup-destination-directory>

        5) Unmount and delete the snapshot and its temporary location

        Three internal variables configure the LVM commands. They can be set
        permanently by editing the script and seting them under the "User
        Settings" section or manually via CLI options of the same name.

    lvm_snapshot_size
        This is the size buffer for the LVM snapshot. See the snapshot function
        of the "lvcreate" command documentation for more details. The default
        value is a pretty safe and relative value that probably won't need
        editing: "15%ORIGIN"

    lvm_volume_group
        This is the name of the volume group to which all LVM functions will
        occur. The partitions to be snapshot are looked for at
        '/dev/<lvm_volume_group>/<volume name>' and the snapshot is also created
        in the <lvm_volume_group> volume group.

    lvm_snapshot_mountpoint_base
        This is the location where the temporary LVM snapshots will be mounted
        to while rsync backs up the files. If all goes well, this directory will
        be deleted when rsync completes. The default value is '/mnt/tmp'. The
        snapshots are created with relatively unique names derrived from a
        timestamp down to the minute.

Rsync

    Rsync performs the actual file backup tasks with the following default options
    on a source directory if backed up for the first time:

        --archive
        --human-readable
        --stats
        --append-verify
        --delete-excluded
        --exclude='**/lost+found*/'
        --exclude='**/*cache*/'
        --exclude='**/*Cache*/'
        --exclude='**/*trash*/'
        --exclude='**/*Trash*/'
        --exclude='**~'
        --exclude='/mnt/*/**'
        --exclude='/media/*/**'
        --exclude='/run/**'
        --exclude='/proc/**'
        --exclude='/dev/**'
        --exclude='/sys/**'
        --exclude='**/.gvfs/'

    For all additional backups on that volume, that backup is incremental. Rsync
    will use hard links to reuse unchanged files and save on file space. There
    will be a symbolic link in '<backup-destination-directory>/source/<volume name>'
    linking to the most recent backup for that volume called '.prev'. If this
    link exists, then the "--delete" and "--link-dest" options are added to rsync.

Resume

   Symlinks to the actual snapshot devices are used to mark active snapshots
   within <backup-list-directory> using the naming format: '.testsource.snap'.
   These files are generated when the snapshot is created and then removed when
   the snapshot is deleted thereby allowing the script to resume on premature
   exit.

Switches/Options:

    -c|--chrono         Toggle to enable the making of chronographic symlinks
                        to backups.

    -d                  Toggle to enable debugging mode; print commands instead
                        of running them. See 'debug-exec' to allow command
                        execution anyway.

    --debug-exec        Toggle to actually run commands or not in debugging mode.

    --interval[=]       Specify which interval this backup is functioning as. "once" is set
                        by default. All backups have identical rsync instructions (except
                        if the backup is the first occurance or not. See the "Rsync" section.)
                        but have different naming schemes and dicrectory trees based on when
                        they run.

                        Note: Be careful about the times in which these are run. The "month"
                        and "day" intervals can run without typical worry of conflict, as they
                        never occur on the same day. But the "once" backup could potentially be run
                        while another backup is in progress. The most caution should be taken
                        with the "hour" interval. On any given day, a backup is already running from
                        either the "month" or "day" interval. Those two could be scheduled at the same
                        hour of the day, say midnight, while the "hour" interval could occur the
                        remaining 23 hours of the day starting at 0100.

            "once"      <backup-destination-directory>/source/<backup label>/YEAR/MONTH/DAY-TIME
                        Meant as an anytime or one-off backup. Compatible with backups
                        using the other labels in the folder structure.

            "month"     <backup-destination-directory>/source/<backup label>/YEAR/MONTH/01
                        Meant to be scheduled on the first day of every month; represents the
                        backup for both that month as well as the first backup of the current
                        month under the "day" interval.

            "day"       <backup-destination-directory>/source/<backup label>/thismonth/DAY
                        Meant to be scheduled daily starting on the second day of the month. That
                        months first backup is handled by the "month" interval.

            "hour"      <backup-destination-directory>/source/<backup label>/today/HOUR
                        Meant to be scheduled hourly as frequently as desired throughout the day.

    --lvm-snapshot-size[=]
                        Set the size buffer for the LVM snapshot. See the snapshot function of the
                        "lvcreate" command documentation for more details.

    --lvm-volume-group[=]
                        Set the name of the volume group to which all LVM functions will occur in.
                        For now, all partitions to be backed up and temporary snapshots are assumed
                        to be in the same volume group; multiple are not currently supported.

    --lvm-snapshot-mountpoint-base[=]
                        Set the location of where the temporary LVM snapshots will be mounted to.

    -s                  For environments that might need it, introduce a random sleep delay between
                        1-60 seconds to help prevent simultaneous instances of the script and help
                        one instance become a clear winner to create the lock file.

    --version           Version

    -h|--help           Short|Full help

Requirements:

rsync, lvm

Version:

iserb version: 0.9.3
Last modifed on: 2017.05.26-14:59
```
