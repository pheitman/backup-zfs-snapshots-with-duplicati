# backup-zfs-snapshots-with-duplicati
Scripts and instructions to backup zfs snapshots with Duplicati

This project has scripts that I have written to address a specific use case. I am publishing them in case that someone else might find them useful.

If you:

  1) Have duplicati running in a docker container
  2) Want to backup filesystems that are mounted on ZFS datasets
  3) Would like to take advantage of ZFS Snapshots so that each backup is clean (that is, all of the files are unchanging during the backup)

then these scripts may be useful for you.

The top-level script is called backup-dataset. When it is invoked, it calls other scripts to:

  1) find the name of the backup job that includes the name of the dataset
  2) find the job id of that backup job
  3) create a snapshot and mount it on a sibling directory to the pool mount named snaps/\<dataset\>
  4) tell the duplicati server to start a backup task for the given id. Note that this just queues the backup. If there is already a backup running, this backup will not start until the current one finishes

I have tested these scripts on TrueNAS, debian (with ZFS added after installation) and Ubuntu 24.04


To use the scripts:

Download the scripts to a system that can access the duplicati server

Download the duplicati_client from https://github.com/Pectojin/duplicati-client/releases/download/0.6.5_beta/duplicati_client_0.6.5_gnu_linux.zip, unzip it and put duplicati_client in the same directory as the scripts

Log in to the duplicati server:

    ./duc login <duplicati server url>

Next, edit backup-dataset.env, updating ZFS_POOL_DIR to point to your pool mount point. On TrueNAS this would be /mnt/\<pool\>, on other systems it is likely to be /\<pool\>

Then create snapshots for each dataset you are going to back up

    ./create-and-mount-datset-snapshot <dataset>

Do that for each dataset

Note the location of the mounted snapshot. If your pool is mounted at /mnt/\<pool\>, the snapshot will be mounted at /mnt/snaps/\<dataset\>

Now create backup jobs in duplicati for each dataset. Note that the backup job name must include the name of the dataset (and only one backup job can include the dataset name). Note also that for the source for the backup you should select the snapshot mount point. Do not have the backup job scheduled. You will later create a cron job to invoke the backup-dataset script

From within the duplicati gui, run the backup job to make sure that it is working as expected

Then from the directory where you installed the scripts, run

    ./backup-datasets <dataset>

You should be able to follow the progress of the backup in the duplicati gui

Do this for each backup job/dataset

Once you have verified that all of the backup jobs are working correctly, create a cron job that invokes the script on all of the datasets to be backed up, e.g.

    15 2 * * *    <full path to scripts>/backup-dataset <dataset1> <dataset2>

Examples from my use case:

I have my pool mounted at /mnt/disks. I want to backup a snapshot of the filesystem /mnt/disk/apps

First, edit ./backup-dataset.env to make sure that the ZFS_POOL_DIR points to the pool (in my case to /mnt/disks)

I create a backup job named 'IDrive e2 - zfs- apps'
The key thing here is that "apps" is included in the name of the job. This is so that the scripts can find the job associated with the dataset.

The source for the backup job is /mnt/snaps/apps
The snapshot taken before the job is run will be mounted at /mnt/snaps/apps. Before creating the job, I run './create-and-mount-dataset-snapshot apps' so that the directory /mnt/snaps/apps is created and I can target it as the source of the job

I then run the backup job from the duplicati gui once to make sure that it is set up correctly.
Then I run './backup-dataset apps' which will unmount the previous snapshot, delete the previous snapshot, create a new snapshot and mount it at /mnt/snaps/apps. It will then start the backup job that includes the name of the dataset 'apps'

Then I do the same for the other datasets that I want to back up

When I have verified that everything is working correcty, I set up a cron job for the root user to run <full path>/backup-dataset apps \<second dataset\> \<third dataset\> ...

One additional suggestion. I have email confirmations set up in duplicati so that I get an email every time a backup has finished - whether it succeeds or fails. That way I know that the cron job ran correctly every night.

...

