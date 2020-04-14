.. title: When Android backup and restore tools may fail
.. date: 2016-08-27 00:00:00 +0300
.. tags: android, backup-and-restore
.. category: android
.. type: text

> The original question was originally posted on August 27, 2016 at [Android StackExchange](http://android.stackexchange.com/questions/155957/cannot-restore-an-adb-backup-due-to-a-tarball-with-a-broken-header)

Recently I've faced with a problem of migrating applications and their data from one device to another.
Backing up and restoring is not a hard think, at least it should not be.
However, standard Android approach may not work as expected causing weird issues.
Let's take a look at one of them.

<!-- TEASER_END -->

I have two devices that have Android 5.0.2 and 6.0.1 respectively (LG Optimus E975, Android 5.0.2, CyanogenMod 12-2015... and Samsung Galaxy S5, Android 6.0.2, flashed to a newer Sprint stock firmware (initially Android 5.0.x earlier)).
I would like to migrate some non-cloud apps from the device with the older Android version to the new one using `adb`.
The backing up command is simple:

```
adb backup -apk -f foo.bar.baz.ab foo.bar.baz
```

Now, after connecting the device with the newer Android version, the following command is expected to work:

```
adb restore foo.bar.baz.ab
```

Unfortunately it isn't.
As usual, the restoring process fails silently just reporting that the restoring process is finished.
Actually nothing really happens.
And here are the logs:

```
adb logcat -s BackupManagerService
```

```
08-27 01:02:44.100  1300  2300 I BackupManagerService: Beginning full restore...  
08-27 01:02:44.100  1300  2300 D BackupManagerService: Starting restore confirmation UI, token=1956031088  
08-27 01:02:44.120  1300  2300 D BackupManagerService: Waiting for full restore completion...  
08-27 01:02:45.650  1300  2450 D BackupManagerService: acknowledgeFullBackupOrRestore : token=1956031088 allow=true  
08-27 01:02:45.660  1300 18181 I BackupManagerService: --- Performing full-dataset restore ---  
08-27 01:02:45.680  1300 18181 I BackupManagerService: Package foo.bar.baz not installed; requiring apk in dataset  
08-27 01:02:45.680  1300 18181 D BackupManagerService: APK file; installing  
08-27 01:02:45.680  1300 18181 D BackupManagerService: Installing from backup: foo.bar.baz  
08-27 01:02:45.690  1300 18181 E BackupManagerService: Unable to transcribe restored apk for install  
08-27 01:02:45.690  1300 18181 E BackupManagerService: Parse error in header: Invalid number in header: 'я╛В' for radix 8  
08-27 01:02:45.710  1300 18181 W BackupManagerService: io exception on restore socket read  
08-27 01:02:45.710  1300 18181 W BackupManagerService: java.io.IOException: Invalid number in header: 'я╛В' for radix 8  
08-27 01:02:45.710  1300 18181 W BackupManagerService:  at com.android.server.backup.BackupManagerService$PerformAdbRestoreTask.extractRadix(BackupManagerService.java:7380)  
08-27 01:02:45.710  1300 18181 W BackupManagerService:  at com.android.server.backup.BackupManagerService$PerformAdbRestoreTask.readTarHeaders(BackupManagerService.java:7179)  
08-27 01:02:45.710  1300 18181 W BackupManagerService:  at com.android.server.backup.BackupManagerService$PerformAdbRestoreTask.restoreOneFile(BackupManagerService.java:6396)  
08-27 01:02:45.710  1300 18181 W BackupManagerService:  at com.android.server.backup.BackupManagerService$PerformAdbRestoreTask.run(BackupManagerService.java:6254)  
08-27 01:02:45.710  1300 18181 W BackupManagerService:  at java.lang.Thread.run(Thread.java:818)  
08-27 01:02:45.710  1300  2300 I BackupManagerService: Full restore processing complete.  
08-27 01:02:45.720  1300 18181 D BackupManagerService: Full restore pass complete.  
```

It looks somewhat strange that the same backup format cannot be restored on a newer device.
I've also tried to repack the underlying tarball using [`nelenkov/android-backup-extractor`](https://github.com/nelenkov/android-backup-extractor) preserving the same file order (`tar cvf ... -T files.lst`) hoping that the broken header gets repaired (a precompiled binary, Git commit hash `fcb4ee5af3ec973a406709de889a5cf340b61ce1`, still can be downloaded [here](/downloads/abe-all-fcb4ee5af3ec973a406709de889a5cf340b61ce1.jar)).
No luck.

Trying out the backup and restore tools from [Google Play](https://play.google.com/store) didn't have a fine result as well.
Some of the tools were just weird, and some of them, like [Titanium Backup](https://play.google.com/store/apps/details?id=com.keramidas.TitaniumBackup), were unable to restore the backups.

After a whole day of various experiments with another tools I've accidentally figured out that specifying the `-apk` key was the problem.
Simply omitting the flag during a backup operation makes it fully restorable.

So my new backup/restore plan became as follows:

* device 1: backup data only
* device 2: backup APK+data retaining the APK only later
* device 2: restore the APK using `adb install`
* device 2: restore the data using `adb restore`

Here are the scripts:

#### backup.sh

```bash
#!/bin/bash

set -e

PACKAGE=$1
    
if [ -z "$PACKAGE" ]; then
	echo no package specified
	exit
fi
    
echo ">>> Backing up $PACKAGE data..."
adb backup -f "$PACKAGE".DATA.ab $PACKAGE

echo ">>> Backing up $PACKAGE APK..."
adb backup -apk -f "$PACKAGE".APK.ab $PACKAGE

echo ">>> Extracting APK..."
java -jar abe-all.jar unpack "$PACKAGE".APK.ab "$PACKAGE".APK.ab.tar
rm "$PACKAGE".APK.ab
APK_FILENAME=$(basename $(tar tf "$PACKAGE".APK.ab.tar | grep '^apps/'"$PACKAGE"'/a/'))
tar xvf "$PACKAGE".APK.ab.tar -C ./ "apps/$PACKAGE/a/$APK_FILENAME" --strip-components=3
mv "$APK_FILENAME" "$PACKAGE".apk
rm "$PACKAGE".APK.ab.tar

echo
echo "*************"
echo "*** Done ***"
echo "*************"
```

This script requires a package name as the input parameter and stores an APK and its data as `foo.bar.apk` and `foo.bar.DATA.ab` respectively.
I also implemented it as a twice-backup script (a device asks for making a backup two times) because I didn't want to re-pack underlying tarballs preserving the original file order just making sure that the `DATA.ab` file is in its original state.

#### restore.sh

```bash
#!/bin/bash

set -e

PACKAGE=$1

if [ -z "$PACKAGE" ]; then
	echo no package specified
	exit
fi

echo ">>> Installing $PACKAGE APK..."
adb install -r $PACKAGE.apk

echo ">>> Restoring $PACKAGE data..."
adb restore $PACKAGE.DATA.ab

echo
echo "************"
echo "*** Done ***"
echo "************"
```

This script simply installs an APK on a target device and restores its data.
The only input parameter is the application package name.

So the overall backup/restore process is:

* Connect device #1
* Execute `backup.sh` passing an application package name
* Click "Backup..." on the device #1 (one will be asked two times for the data and APK+data backups respectively)
* Repeat the previous two steps for other packages if necessary
* Disconnect the device #1 and connect the  device #2
* Execute `restore.sh` passing an application package name
* Click "Restore..." on the device #2 once asked
* Repeat the previous two steps for other packages if necessary

That's all.
Extracting the APK files also lets me keep applications that had been removed from Google Play.
