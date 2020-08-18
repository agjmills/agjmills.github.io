---
layout: post
title: Simple & Cheap Server Backups with Amazon S3
---

Amazon S3 is a cheap, resilient and reliable storage platform, so it makes a great choice for storing backups.

I rolled my own backup script, which backs up a couple of WordPress sites, and 14 or so Laravel applications to S3 - with a cost of only $2.40 per month!

You'll need to install the [AWS CLI tools](https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-install.html), [create an S3 bucket and some credentials for it](https://docs.aws.amazon.com/AmazonS3/latest/dev/walkthrough1.html).

I've then written a script which does a few things:
* Dump MySQL databases individually and compress them
* Compress the webroot, excluding installed composer packages, node_modules and generated PDFs
* Combines them to one file, which is also compressed
* Upload to Amazon S3, using the storage class STANDARD_IA

```
#!/bin/bash
export AWS_CONFIG_FILE="/root/.aws/config"
export AWS_SHARED_CREDENTIALS_FILE="/root/.aws/credentials"
date=$(date '+%Y-%m-%d')
time=$(date '+%H:%M:%S')
hostname=$(hostname)

# Database backup
echo "[$(date '+%Y-%m-%d_%H:%M:%S')] Dumping database"
mkdir -p /tmp/databases
mysql -N -e 'show databases' | grep -Ev 'information_schema|performance_schema|sys' | while read dbname; do mysqldump --events --add-drop-database --add-drop-table --complete-insert --routines --triggers --single-transaction "$dbname" > /tmp/databases/"$dbname-$hostname".sql; done
echo "[$(date '+%Y-%m-%d_%H:%M:%S')] Setting database to read only"
chmod -R 400 "/tmp/databases/"
echo "[$(date '+%Y-%m-%d_%H:%M:%S')] Creating database tarball"
tar -cf "/tmp/$date-$hostname.sql.tar" /tmp/databases

# /var/www backup
echo "[$(date '+%Y-%m-%d_%H:%M:%S')] Archiving /var/www"
tar -cf "/tmp/$date-$hostname.www.tar" --exclude='**/vendor' --exclude='**/node_modules' --exclude='**/storage/**.pdf' /var/www &>/dev/null
echo "[$(date '+%Y-%m-%d_%H:%M:%S')] Setting /var/www archive to readonly"
chmod 400 "/tmp/$date-$hostname.www.tar"

# Combining all backups
echo "[$(date '+%Y-%m-%d_%H:%M:%S')] Creating single archive backup"
tar -zcf "/tmp/$date-$hostname.backup.tgz" "/tmp/$date-$hostname.www.tar" "/tmp/$date-$hostname.sql.tar" &>/dev/null
echo "[$(date '+%Y-%m-%d_%H:%M:%S')] Setting single archive to readonly"
chmod 400  "/tmp/$date-$hostname.backup.tgz"

# Upload to S3
echo "[$(date '+%Y-%m-%d_%H:%M:%S')] Uploading backup archive to S3"
/usr/local/bin/aws s3 cp --storage-class STANDARD_IA --quiet "/tmp/$date-$hostname.backup.tgz" "s3://asdfx-backups/$hostname/$date/$time-$hostname.backup.tgz"

# Cleanup
echo "[$(date '+%Y-%m-%d_%H:%M:%S')] Cleaning up dumps"
rm -rf "/tmp/databases"
rm -rf "/tmp/$date-$hostname.sql.tar"
rm -rf "/tmp/$date-$hostname.www.tar"
rm -rf "/tmp/$date-$hostname.backup.tgz"
```

It's relatively straightforward to add additional services to be backed up, such as elasticsearch via elasticdump, as long as it is added to the final backup tarball it will be uploaded to S3.

I have played with various forms of compression for backups, but found that gzipping the tarball was the best tradeoff between size and CPU resource.

I run the backup script from a cron, which runs every couple of hours, but you can have it more or less frequent.

You can also opt for even cheaper backups, by changing the storage class to GLACIER / DEEP_ARCHIVE - however this can take several hours to be retrieved.

I also have a [lifecycle rule on my backup buckets](https://docs.aws.amazon.com/AmazonS3/latest/user-guide/create-lifecycle.html), which deletes backups after 90 days - further reducing the cost.
