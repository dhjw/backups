# Backups on Linux and Windows with Restic

Who knew cross-platform backups could be so easy?

## Setup

### Install Restic
Restic is just a single binary, so you can simply grab it from its [Releases](https://github.com/restic/restic/releases/) page and put it somewhere in your path.

Note that many Linux distros install a version with the `self-update` feature disabled if you use the package manager, so it's not recommended to install that way.

On Linux (amd64):
```
cd /tmp
curl -L https://github.com/restic/restic/releases/download/v0.18.0/restic_0.18.0_linux_amd64.bz2 > restic.bz2
bzip2 -d restic.bz2
chmod +x restic
mv restic /usr/bin
restic self-update # also runs every time the backup script runs
```
On Windows, [download the ZIP](https://github.com/restic/restic/releases/download/v0.18.0/restic_0.18.0_windows_amd64.zip), extract it, rename the binary `restic.exe`, and put it somewhere in your path. `C:\Windows\System32` works. Open a Terminal and run `restic self-update`. [Install Git](https://git-scm.com/downloads/win) and put `C:\Program Files\Git\bin\bash.exe` [in your path](https://superuser.com/a/1861277).

### Install my wrapper and create a backup password

Choose a parent folder, e.g. `/usr/local/bin` or `C:\Users\d\bin`, `cd` to it and clone this repo `git clone https://github.com/dhjw/backups`, then `cd backups`.

Create a `backup-pass` file with a secure password:
```
bash -c "tr -dc 'A-Za-z0-9' < /dev/urandom | head -c 48 > backup-pass; cat backup-pass; echo"
```
Save the password somewhere safe because if you lose it you won't be able to access your backups. I use a [KeePass2](https://keepass.info/download.html) vault copied to a USB drive and some cloud services.

On Linux, files should be owned by root with restrictive permissions `chmod 600 backup*; chmod 700 backup`. For ease of use, put the `backup` wrapper script in your path, e.g. with a symlink. `cd /usr/local/bin; ln -s backups/backup`.

### Set up Storage

Currently this script expects to be used with S3-compatible endpoints like [AWS](https://aws.amazon.com/s3/), [BackBlaze](https://www.backblaze.com/cloud-storage), etc, however it should be easy to adapt to other Restic-supported services.

#### Amazon S3
- Create an [S3](https://console.aws.amazon.com/s3) bucket, e.g. `yourdevice-backups`
- Edit `backup.cfg` to contain the region and bucket name
- In [IAM](https://console.aws.amazon.com/iam), create a user with an inline policy:
 ```json
{
	"Version": "2012-10-17",
	"Statement": [
		{
			"Sid": "Stmt1390497858034",
			"Effect": "Allow",
			"Action": [
				"s3:GetObject",
				"s3:PutObject",
				"s3:ListBucket",
				"s3:DeleteObject"
			],
			"Resource": [
				"arn:aws:s3:::yourdevice-backups",
				"arn:aws:s3:::yourdevice-backups/*"
			]
		}
	]
}
```
- In IAM, generate credentials of type "Other" to create an Access Key and Secret Access Key
- Add the keys to `backup.cfg` where indicated

## Run before and after backup
To run things before and after backup, set the `BACKUP_PRE` and `BACKUP_POST` vars in `backup.cfg`.

For example, to dump MySQL databases before backup on a Debian or Ubuntu-based system, this can work:
```
export BACKUP_PRE="echo 'dumping databases'; mysqldump --defaults-extra-file=/etc/mysql/debian.cnf --all-databases -r /root/databases.sql; ls -alh /root/databases.sql"
export BACKUP_POST="rm /root/databases.sql"
```
Don't forget to add `/root/databases.sql` to `backup-includes`

Or you could create scripts and call them. The current folder at runtime is always the backup script folder.
```
export BACKUP_PRE="./backup-pre" # run my scripts
export BACKUP_POST="./backup-post"
```
Note currently only bash scripts are supported, even on Windows.

## Commands

On Windows, run all commands like `bash -c "./<command>"` (if in script folder, else use the full path).

- First, run `backup init` to initialize the repository and set the backup encryption password (the one you generated earlier).
- Edit `backup-includes` and `backup-excludes` to include and exclude files for backup. Some advanced wildcards and negation can be seen in the example files, or view the [docs](https://restic.readthedocs.io/en/latest/040_backup.html#excluding-files).
- To do a dry run, use`backup run dry` or `backup run dryv` (verbose - shows matched files). Dry runs write to stdout, normal runs write to the log file. On Windows, you can set up a `grep` alias [like this](https://g.co/gemini/share/d04ea9cc84e7) to pipe to (`bash -c "./backup run dryv" | grep something`) to check if the files you want are included.
- After a real backup with `backup run`, you can list snapshots with `backup snapshots` and view more about a snapshot with `backup ls <snapshot-id>`
- The wrapper passes through most other commands to the restic binary with your credentials and settings applied. Try `backup help` and `backup help <command>`

## Scheduled Tasks
On Linux, set up a cron job as root, e.g. `0 0 * * * backup run`


On Windows, open Task Scheduler and create a daily task with Program/script `"C:\Program Files\Git\bin\bash.exe"` and Arguments`-c 'C:\Users\d\bin\backups\backup run'` (with your path, single quotes are required). Enable the option "Run whether user is logged on or not" on the General tab to prevent a terminal window from popping up. You can Right-click > Run the task to make sure it creates a snapshot as expected.

## Restoring Files
On Linux, you can mount remote backups for browsing with the mount command. Load another terminal to browse the folder. When done, Ctrl-C in the first one to unmount.
```
mkdir mnt
backup mount ./mnt
```

On Windows, check out `bash -c "./backup help restore"`

## Notes
- The log doesn't currently rotate but shouldn't get too big with daily runs. If it does you can delete it and it will be recreated automatically.
- There's currently no error-handling. Restic should gracefully recover in most cases. Check on your backups occasionally.