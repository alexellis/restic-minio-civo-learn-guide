## Backup your data using Restic, Minio, and Civo

In this guide we will explore how to backup our local files to remote storage hosted on Civo, a developer-cloud in the UK. The tool of choice is [restic](https://restic.net/#quickstart) and [Minio](https://min.io) will be providing object storage.

![](/image/restic-civo.png)

*Conceptual overview*

> ✅ Follow Civo on Twitter [@civocloud](https://twitter.com/civocloud)

### An introduction the tools:

Civo offers 50 USD free credits to new users. This will be enough to run a Medium-sized Instance with 50 GB of local SSD storage for over a month.

#### Restic

Here's a few words on restic from [the project homepage](https://restic.net/#quickstart):

> restic is a program that does backups right. The design goals are:
> Easy, Fast, Verifiable, Secure, Efficient and Free
>
> restic is free software and licensed under the BSD 2-Clause License and actively developed on GitHub.

#### Minio

[Minio](https://min.io) is a drop-in replacement for [Amazon S3](https://aws.amazon.com/s3/) for backing up files, as a storage back-end for tools such as a container registry, or even to host static websites. 

Minio describes itself as:

> The 100% Open Source, Enterprise-Grade, Amazon S3 Compatible Object Storage

Civo will be used to host Minio and will provide a public IP address that our laptop or personal computer can connect to over the Internet to back up files using restic. Restic is a client tool which we will run locally.

### Provision your Instance

* Log into your Civo dashboard

* Create a Medium sized Instance and call it `minio-backup`.

![](/image/instance.png)

* Select Ubuntu 18.04 for the Operating System, add your SSH key for login and the default firewall.

![](/image/os.png)

Your Instance will be ready in around 45s.

#### Install Minio server

Log into your Instance using ssh and install the Minio Server.

We will using `/mnt/data` for Minio's datastore.

```
sudo mkdir -p /mnt/data

wget https://dl.min.io/server/minio/release/linux-amd64/minio
chmod +x minio
sudo mv ./minio /usr/local/bin/minio
```

Now start the server component with the following:

```sh
sudo minio server /mnt/data &
```

You will see your *access key* and *secret key* printed on the console, these are required for restic later on, so take a note of them.

![](/image/minio-info.png)

### Install restic on your client

According to the [Installation page](https://restic.readthedocs.io/en/stable/020_installation.html) for restic, some of the versions available in package management tools are out of date, or running a few revisions behind. For the latest and greatest, use the GitHub releases page.

In this guide [0.9.5 was used](https://github.com/restic/restic/releases/tag/v0.9.5).

Depending on whether you are using MacOS, Linux or Windows, pick the corresponding binary along with the suffix "amd64" which is the standard architecture of most CPUs.

Run this on your laptop or PC:

```sh
wget https://github.com/restic/restic/releases/download/v0.9.5/restic_0.9.5_darwin_amd64.bz2
bzip2 -d restic_0.9.5_darwin_amd64.bz2
chmod +x restic_0.9.5_darwin_amd64 
sudo install restic_0.9.5_darwin_amd64 /usr/local/bin/restic
```

Check that the installation worked:

```sh
restic version
restic 0.9.5 compiled with go1.12.4 on darwin/amd64
```

#### Prepare a repository

According to restic's documentation:

> The place where your backups will be saved is called a "repository".

Our repository will be the remote minio server.

Fill out the following using the secret and access key from the step where you ran `minio server` on your Civo Instance:

```sh
export AWS_ACCESS_KEY_ID=<YOUR-MINIO-ACCESS-KEY-ID>
export AWS_SECRET_ACCESS_KEY="<YOUR-MINIO-SECRET-ACCESS-KEY>"
```

Now set the `MINIO_IP`:

```sh
export MINIO_IP="185.136.233.182"
```

Now run the command to prepare the repository:

```sh
restic -r s3:http://$MINIO_IP:9000/restic init

enter password for new repository: 
enter password again: 
created restic repository 760b0971f4 at s3:http://185.136.233.182:9000/restic

Please note that knowledge of your password is required to access
the repository. Losing your password means that your data is
irrecoverably lost.
```

You will be asked to enter a password. Make sure you take a note of it and use something that is considered a strong password.

> Note: The bucket name we will use is called `restic`, but that can be changed and multiple buckets or repositories can be created for different clients.

A note on automated backups.

When running an automated backup, you cannot type in passwords to an interactive prompt. For these instances there are three options available:

* Set the environment variable `RESTIC_PASSWORD`
* Pass the path to a file containing the password `--password-file`
* Or pass a command to run which gives the password via stdout `--password-command`

#### Run your first backup

This is a sample backup command:

```sh
restic -r s3:http://$MINIO_IP:9000/restic --verbose backup ~/dev
```

* the `-r` flag is used to pass in the repository
* the `backup` verb synchronises files
* the final `~/dev` command is used to specify which files to synchronise into the repository

Let's get some sample files to play with by cloning the restic source code:

```sh
cd /tmp/
git clone https://github.com/restic/restic
rm -rf restic/.git

find restic/|wc -l
 2427                # That's 2.5k files

du -h -d 0 restic
 46M	restic      # And around 50MB of code
```

We can now backup the restic source code to our Civo server.

```sh
restic -r s3:http://$MINIO_IP:9000/restic --verbose backup ./restic
```

We need the password again:

```sh
open repository
enter password for repository: 


repository 760b0971 opened successfully, password is correct
created new cache in /Users/alex/Library/Caches/restic
lock repository
load index files
start scan on [./restic]
start backup on [./restic]
scan finished in 0.350s: 2060 files, 40.671 MiB

Files:        2060 new,     0 changed,     0 unmodified
Dirs:            0 new,     0 changed,     0 unmodified
Data Blobs:   2045 new
Tree Blobs:      1 new
Added to the repo: 40.591 MiB

processed 2060 files, 40.671 MiB in 1:06
snapshot 44217521 saved
```

The total speed depends on your broadband connection and the latency between your Civo Instance and your current location.

If we run the backup again, this time we will see it complete almost instantly:

```sh
repository 760b0971 opened successfully, password is correct
lock repository
load index files
using parent snapshot 44217521
start scan on [./restic]
start backup on [./restic]
scan finished in 0.337s: 2060 files, 40.671 MiB

Files:           0 new,     0 changed,  2060 unmodified
Dirs:            0 new,     0 changed,     0 unmodified
Data Blobs:      0 new
Tree Blobs:      0 new
Added to the repo: 0 B  

processed 2060 files, 40.671 MiB in 0:00
snapshot 39eea727 saved
```

#### Restore from your backup

The opposite of backing-up data is recovering it or restoring it.

```sh
mkdir -p /tmp/restic-source

$ restic -r s3:http://$MINIO_IP:9000/restic --verbose restore latest --target /tmp/restic-source
enter password for repository: 
repository 760b0971 opened successfully, password is correct
restoring <Snapshot ef8ac197 of [/tmp/restic] at 2019-08-02 09:56:12.068743 +0100 BST by alex@space-mini.local> to /tmp/restic-source
```

When running the `du` utility we can see that the total size is the same as what we pushed up:

```sh
$ du -h -d 0 restic-source
 46M	restic-source
```

You can also restore a specific snapshot or point in time from the restic tool.

To list specific snapshots, or backup jobs:

```sh
restic -r s3:http://$MINIO_IP:9000/restic --verbose snapshots

enter password for repository: 
repository 760b0971 opened successfully, password is correct
ID        Time                 Host              Tags        Paths
------------------------------------------------------------------------
44217521  2019-08-02 09:14:39  space-mini.local              /tmp/restic
39eea727  2019-08-02 11:16:32  space-mini.local              /tmp/restic
ef8ac197  2019-08-02 16:56:12  space-mini.local              /tmp/restic
------------------------------------------------------------------------
3 snapshots
```

> Read more: [restoring a backup](https://restic.readthedocs.io/en/stable/050_restore.html)

### Taking things further

Now that you can backup your data at any time over the Internet, let's look at how to take things further and what else you need to consider.

#### Other backup targets

There are a number of backup targets supported such as:

* Local mount, such as a USB HDD
* SFTP - this is an encrypted file transfer which runs over SSH, you can use it with any Civo Instance
* Amazon S3 - a storage bucket hosted on AWS
* REST and HTTP

OpenStack, Azure Blob Storage, Google Cloud Storage and another of other options are also supported.

> See also: [Preparing a new repository](https://restic.readthedocs.io/en/stable/030_preparing_a_new_repo.html)

#### Backup your backup with snapshots

Your Civo Instance makes 50 GB of SSD-backed storage available, but what if you delete your Instance by accident? Your data would be lost.

Fortunately a feature of the Civo platform is that whenever you delete a Instance, it will be kept in archival storage for 30 days, meaning that if you change your mind, the team can help you get it back.

The other feature Civo offers is the use of Snapshots. A snapshot of a Instance is a fast and efficient way to restore your Instance back to a known-state.

Snapshots can be taken on a manual, or periodic basis using the "Snapshot" button onn the instances page. 

![](/image/snapshot.png)

For piece of mind, you can select "Automated".

Here's the snapshot I just took:

![](/image/snapshots.png)

#### Turn minio into a service with systemd

We started Minio's server as a simple binary, but if it crashes, it will not restart on its own. Similarly, if we had a power-cycle on the Instance, the server won't restart.

On the Civo Instance, let's create a systemd unit file as `minio.service`:

```ini
[Unit]
Description=minio
Documentation=https://docs.min.io
Wants=network-online.target
After=network-online.target
AssertFileIsExecutable=/usr/local/bin/minio

[Service]
WorkingDirectory=/usr/local/

User=minio-user
Group=minio-user

EnvironmentFile=/etc/default/minio

ExecStart=/usr/local/bin/minio server $MINIO_OPTS $MINIO_VOLUMES

# Let systemd restart this service always
Restart=always

# Specifies the maximum file descriptor number that can be opened by this process
LimitNOFILE=65536

# Disable timeout logic and wait until process is stopped
TimeoutStopSec=infinity
SendSIGKILL=no

[Install]
WantedBy=multi-user.target
```

* Now create `/etc/default/minio`

```sh
MINIO_VOLUMES="/mnt/data"
MINIO_OPTS=""
MINIO_ACCESS_KEY="FI1BIGZTLAGI0BAZSAO0"
MINIO_SECRET_KEY="WPn6pgoIo+g5q+gXmNsnontZFbsvfr+y+eqxnyYN"
```

* Kill the server we started manually. `sudo killall minio`

* Create a new user `minio-user` and give it permissions to the data-store:

```sh
sudo useradd minio-user
sudo chown minio-user -R /mnt/data
```

* Install the service and start it:

```sh
sudo cp minio.service /lib/systemd/system/
sudo systemctl enable minio.service
sudo systemctl start minio.service
```

#### Turn on TLS for Minio

Whilst restic does use encryption to store data, we should also have encryption enabled at the link level. This can be achieved by turning TLS on for Minio.

> See also: Minio [How to secure access to MinIO server with TLS](https://docs.min.io/docs/how-to-secure-access-to-minio-server-with-tls.html)

#### Turn on Erasure Code for Minio

One of the features of Minio, when running in a distributed (clustered) mode is [Erasure Code](https://en.wikipedia.org/wiki/Erasure_code).

According to the Minio documentation this feature can help mitigate against "bit rot", where one or more bits may get silently corrupted without an error or notification.

> See also: [Erasure Code Quickstart](https://docs.min.io/docs/minio-erasure-code-quickstart-guide.html)

#### Try Amazon S3 and another region

As part of my testing for this guide, I tried backing up the restic code to an S3 bucket on the West Coast of America. This clear has a much longer trip to make and higher latency, but the syntax is almost identical:

![](/image/aws-s3.png)

In this case the uploads took a similar amount of time, and this is likely due to the upload speed of my broadband connection. The download from Civo's location in the UK, is likely to be much quicker.

### Need help?

Civo prides itself on being a developer-cloud run by developers who can provide technical support and expert help via Intercom and the [community forums](https://www.civo.com/community).

* If you want to find out more about Minio, join the [Minio Slack](https://slack.min.io/) workspace.

### Wrapping up

We now have around 50GB of SSD-backed storage that we can backup our local files to from anywhere in the world. We can restart our server at any time thanks to the systemd file, we can get regular snapshots from Civo and we have the option to enable link-level encryption through TLS.

* Find out more about [restic in the FAQ](https://restic.readthedocs.io/en/stable/faq.html)

* Find out what [else you can do with Minio](https://min.io/)

✅ Follow Civo on Twitter [@civocloud](https://twitter.com/civocloud)
