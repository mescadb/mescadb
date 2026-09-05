# Move Data into OCI Cloud Storage Services using Rclone
## Introduction

This is tutorial 2 of a four tutorial series that shows you various ways to migrate data into Oracle Cloud Infrastructure (OCI) cloud storage services. The series is set up so you can review [Tutorial 1: Use Migration Tools to Move Data into OCI Cloud Storage Services](https://docs.oracle.com/en/learn/migr-ocistorage-p1) to get a broad understanding of the various tools and then proceed to the related tutorial(s) or documents relevant to your migration needs. This tutorial will focus on using Rclone to migrate data into OCI cloud storage services.

OCI provides customers with high-performance computing and low-cost cloud storage options. Through on-demand local, object, file, block, and archive storage, Oracle addresses key storage workload requirements and use cases.

OCI cloud storage services offers fast, secure, and durable cloud storage options for all your enterprise needs. Starting with the high performance options such as OCI File Storage with Lustre and OCI Block Volumes service; fully managed exabyte scale filesystems from OCI File Storage service with high performance mount targets; to highly durable and scalable OCI Object Storage. Our solutions can meet your demands, ranging from performance intensive applications such as AI/ML workloads, to exabyte-scale data lakes.

[Rclone](https://rclone.org/) is an open source, command-line utility to migrate data to the cloud, or between cloud storage vendors. Rclone can be used to do one-time migration as well as periodical synchronization between source and destination storage. Rclone can migrate data to and from object storage, file storage, mounted drives, and between 70 supported storage types. OCI Object Storage is natively supported as a Rclone backend provider. Rclone processes can be scaled up and scaled out to increase the transfer performance using parameter options.

Determine the amount of data that needs to be migrated, and the downtime available to cut-over to the new OCI storage platform. Batch migrations are a good choice to break down the migration into manageable increments. Batch migrations will enable you to schedule downtime for specific applications across different windows. Some customers have the flexibility to do a one-time migration over a scheduled maintenance window over 2-4 days. [OCI FastConnect](https://docs.oracle.com/en-us/iaas/Content/Network/Concepts/fastconnect.htm) can be used to create a dedicated, private connection between OCI and your environment, with port speeds from 1G to 400G to speed up the data transfer process. OCI FastConnect can be integrated with partner solutions such as Megaport and [ConsoleConnect](https://www.consoleconnect.com/clouds/connect-to-oracle-cloud/) to create a private connection to your data center or cloud-to-cloud interconnection to move data more directly from another cloud vendor into OCI cloud storage service. For more information, see [FastConnect integration with Megaport Cloud Router](https://blogs.oracle.com/cloud-infrastructure/post/fastconnect-integration-megaport-cloud-router).

### Audience

DevOps engineers, developers, OCI cloud storage administrators and users, IT managers, OCI power users, and application administrators.

### Objective

Learn how to use Rclone to copy and synchronize data into OCI cloud storage services.

- Use Rclone for migrating filesystem data (local, NAS, cloud hosted) into OCI Object Storage.
- Migrate data from another cloud object or blob storage into OCI Object Storage.
- Use Rclone on Oracle Cloud Infrastructure Kubernetes Engine (OKE) to migrate data from OCI File Storage to OCI Object Storage.

### Prerequisites

- An OCI account.
- Virtual machine (VM) instance on OCI to deploy the migration tools or a system where you can deploy and use migration tools.
- Oracle Cloud Infrastructure Command Line Interface (OCI CLI) installed with a working config file in your home directory in a subdirectory called `.oci`. For more information, see [Setting up the Configuration File](https://docs.oracle.com/en-us/iaas/Content/API/SDKDocs/cliinstall.htm#configfile).
- Access to an OCI Object Storage bucket.
- User permissions in OCI to use OCI Object Storage, have access to manage buckets and objects or manage object-family for at least 1 bucket or compartment. For more information, see [Common Policies](https://docs.oracle.com/en-us/iaas/Content/Identity/Concepts/commonpolicies.htm) and [Policy Reference](https://docs.oracle.com/en-us/iaas/Content/Identity/Reference/policyreference.htm).
- User permission to create, export, and mount OCI File Storage, or access to an OCI File Storage mount target that is already mounted on a VM, or another NFS mount or local file system to use for copying data to and from. For more information, see [Manage File Storage Policy](https://docs.oracle.com/en-us/iaas/Content/Identity/Concepts/commonpolicies.htm#).
- Familiarity with using a terminal/shell interface on Mac OS, Linux, Berkeley Software Distribution (BSD), or Windows PowerShell, command prompt, or bash.
- Familiarity with installing software on a Linux system and have some experience or understanding of Kubernetes.
- Basic knowledge of Oracle Cloud Infrastructure Identity and Access Management (OCI IAM) and working with dynamic groups with a Ubuntu host in a dynamic group. For more information, see [Managing Dynamic Groups](https://docs.oracle.com/en-us/iaas/Content/Identity/Tasks/managingdynamicgroups.htm).
- Review [Migration Essentials for Moving Data into OCI Cloud Storage](https://livelabs.oracle.com/pls/apex/r/dbpm/livelabs/run-workshop?p210_wid=3995&p210_wec=&session=977542126588) to install Rclone and other migration tools.
- To know the migration tools we can use, see [Tutorial 1: Use Migration Tools to Move Data into OCI Cloud Storage Services](https://docs.oracle.com/en/learn/migr-ocistorage-p1).

## Overview of Rclone and Basic Terms

Rclone is a helpful migration tool because of the many protocols and cloud providers it supports and ease of configuration. It is a good general purpose migration tool for any type of data set. Rclone works particularly well for data sets that can be split up into batches to scale-out across nodes for faster data transfer.

Rclone can be used to migrate:

- File system data (OCI File Storage, OCI Block Storage, OCI File Storage with Lustre, on-prem file system, and on-prem NFS) to other file system storage types and to/from object storage (including OCI Object Storage).
- Object storage from supported cloud providers to and from OCI Object Storage.

**Rclone Commands and Flags:**

- **Understand Rclone Performance**
	Rclone is a good general-purpose tool for syncing or copying files between filesystem data, other cloud providers, and OCI cloud storage services. The performance will depend on how much you can scale-up and scale-out. We recommend running various test on your migration systems with a sample migration set to determine when you hit the thresholds of your network bandwidth.
	For example, your source bucket has 10 folders/prefixes, each having about 1TB. You could split the migration across 2 VMs of large CPU/RAM capacity and trigger multiple Rclone copy processes in parallel from the two VMs. Depending on each folder’s topology and the compute capacity, the Rclone parameters can be adjusted to improve transfer speed.
	You could start with running the following commands on 2 VM’s and then adjust the transfer count and checker count until you saturate the NIC on each VM.
	```md
	rclone copy --progress --transfers 10 --checkers 8 --no-check-dest aws_s3_virgina:/source_bucket_name/folder1 iad_oss_native:/destination_bucket_name/folder1
	rclone copy --progress --transfers 50 --checkers 8 --no-check-dest aws_s3_virgina:/source_bucket_name/folder2 iad_oss_native:/destination_bucket_name/folder2
	```
	- Systems or VM instances with more CPU, memory, and network bandwidth can run more file transfers and checkers in parallel. Scaling up to systems with more resources will allow faster performance.
		- If your data can be split up into various batches based on structure, you can also run Rclone on multiple systems or VM instances to scale-out.
	We recommend scaling up and out to improve Rclone performance. Our testing included 2 VM’s to run Rclone transfers in parallel to scale out. If you have a larger data set, you may want to use up to 4 machines or even use Bare Metal (BM) instances.
- **Rclone Copy and Sync Commands**
	- Rclone copy command copies source files or objects to the destination. It will skip files that are identical on source and destination, testing by size and modification time or md5sum. The copy command does not delete files from the destination.
		- Rclone sync command synchronizes the source with the destination, also skipping identical files. The destination will be modified to match the source, which means files not matching the source will be deleted.
	> **Note:** Be careful using sync, and use it only when you want the destination to look exactly like the source. Use the copy command when you just want to copy new files to the destination.
- **Use Right Rclone Command Line Flags**
	There are several Rclone command line flags that can be used with Rclone that affect how fast data migration occurs. It is important to understand how some of these flags work to get the best data transfer throughput.
	- **`--no-traverse`:** This only works with the copy command, do not travel through the destination file system. This flag saves time because it will not conduct lists on the destination to determine which files to copy over. It will check files one at a time to determine if it needs to be copied. One-by-one may seem slow but can be faster when you have a very small number of files/objects to copy over to a destination with many files already present.
		- **`--no-check-dest`:** This only works with the copy command and will not check or list the destination files to determine what needs to be copied or moved which minimizes API calls. Files are always transferred. Use this command when you know you want everything on the source copied regardless of what is on the destination or if you know the destination is empty.
		> **Note:** Use `no-traverse` or `no-check-dest` commands, many users put both on the command line which is not necessary.
		> 
		> - If your target is empty or you want all files copied from the source to the destination no matter what, use `no-check-dest`.
		> - When you have a few very large files that need to be migrated, use `no-traverse` which will check each file to see if it is current on the destination before copying it to the source; this could save on list API calls and the amount of data copied to the target.
		- **`--ignore-checksum`:** This will really speed up the transfer, however Rclone will not check for data corruption during transfer.
		- **`--oos-disable-checksum`:** Do not store MD5 checksum with object metadata. Rclone calculates the MD5 checksum of the data before uploading and adds it to the object metadata, which is great for data integrity, however it causes delays before large files begin the upload process.
		- **`--transfers <int>`:** Number of file transfers to run in parallel (default 4). Scale this number up based on the size of the system where you are running rclone, you can do test runs and increase the integer until you reach max transfer speed for your host. We really recommend testing and raising this number until you get acceptable performance, we have seen customers raise this number between 64-3000 to get desired performance.
		- **`--checkers <int>`:** Number of checks to run in parallel (default 8). Amount of file checkers to run in parallel, be cautious as it can drain server health and cause problems on the destination. If you have a system with very large memory, bump this number up by increments of 2. The maximum number we tested this setting with good results in the test environment is 64, typically 8-10 is sufficient. Checkers can be anywhere from 25-50% of the transfer number; when the transfer number is higher this number tends to be closer to 25%.
		> **Note:** When scaling out with multiple hosts running Rclone transfers and checkers you may hit a **429 “TooManyRequests”** error, should this happen start by lowering the amount of checkers in increments of 2 until you reach 10. If lowering the checkers is not enough, you will also need to lower the number of transfers.
		- **`--progress`:** This will show progress during transfer.
		- **`--fast-list`:** Use recursive list if available; uses more memory but fewer transactions/API calls. This is a good option to use when you have a moderate number of files in nested directories. Do not use with `no-traverse` or `no-check-dest` since they are contrary flags. Can be used with the copy or sync command.
		- **`--oos-no-check-bucket`:** Use this when you know the bucket exists, it reduces the number of transactions Rclone conducts, it sets Rclone to assume the bucket exists and to start moving data into it.
		- **`--oos-upload-cutoff`:** Files larger than this size will be uploaded in chunks, the default is 200MiB.
		- **`--oos-chunk-size`:** When uploading files larger than the upload cutoff setting or files with unknown size they will be uploaded as multipart uploads using this chunk size. Rclone will automatically increase chunk size when uploading a large file of known size to stay below the 10,000 chunks limit. The default is 5MiB.
		- **`--oos-upload-concurrency <int>`:** This is used for multipart uploads and is the number of chunks uploaded concurrently. If you are uploading small numbers of large files over high-speed links and these uploads do not fully utilize your bandwidth, then increasing this may help to speed up the transfers. The default is 8, if this is not utilizing bandwidth increase slowly to improve bandwidth usage.
		> **Note:** Multi-part uploads will use extra memory when using the parameters: `--transfers <int>`, `--oos-upload-concurrency <int>` and `--oos-chunk-size`. Single part uploads do not use extra memory. When setting these parameters consider your network latency, the more latency, the more likely single part uploads will be faster.
- **Rclone Configuration File Example for OCI Object Storage**
	```md
	[oci]
	type = oracleobjectstorage
	namespace = xxxxxxxxxxx
	compartment = ocid1.compartment.oc1..xxxxxxxxx
	region = us-ashburn-1
	provider = user_principal_auth
	config_file = ~/.oci/config
	config_profile = Default
	```
- **Basic Rclone Command Format**
	```md
	rclone <flags> <command> <source> <dest>
	```
	- Example of running Rclone copy from a local filesystem source or OCI File Storage source to OCI Object Storage destination.
		```md
		rclone copy /src/path oci:bucket-name
		```
		- Example of running Rclone copy from OCI Object Storage source to a local filesystem or OCI File Storage destination.
		```md
		rclone copy oci:bucket-name /src/path
		```
		- Example of running Rclone copy from an S3 source to a OCI Object Storage destination.
		```md
		rclone copy s3:s3-bucket-name oci:bucket-name
		```
		> **Note:** When migrating from AWS and using server-side encryption with KMS, make sure rclone is configured with `server_side_encryption = aws:kms` to avoid checksum errors. For more information, see [Rclone S3 KMS](https://rclone.org/s3/#key-management-system-kms) and [Rclone S3 configuration](https://rclone.org/s3/#configuration)
	> **Note:** Format of the sync command will be basically the same, simply replace copy with sync.

## Rclone Usage Examples

- **Example 1:** Use Rclone to migrate a small number of small files copied into a destination already containing data with a high file or object count.
	```md
	rclone --progress  --transfers 16  --oos-no-check-bucket --checkers 8  --no-traverse copy <source> <dest>
	```
- **Example 2:** Rclone with fewer large files with multi-part uploads.
	```md
	rclone --progress  --oos-no-check-bucket --fast-list --no-traverse --transfers 8 --oos-chunk-size 10M --oos-upload-concurrency 10 --checkers 10  copy <source> <dest>
	```
	> **Note:** These are starting points for the options `--transfers`, `--oos-chunk-size`, `--oos-upload-concurrency`, and `--checkers`, you will need to adjust them based on your file/object size, memory and resources available on the systems you are using for migrating data. Adjust them up until you get sufficient bandwidth usage to migrate your data optimally. If your system is very small, you may need to adjust these numbers down to conserve resources.
- **Example: 3** Use Rclone for scale-out run on 3 BM machines with 100 Gbps NIC, data set mixed size with multi-part uploads with petabytes of data, bucket not empty, OCI File Storage service to OCI Object Storage service.
	```md
	rclone --progress --stats-one-line --max-stats-groups 10 --fast-list --oos-no-check-bucket --oos-upload-cutoff 10M --transfers 64 --checkers 32 --oos-chunk-size 512Mi --oos-upload-concurrency 12 --oos-disable-checksum --oos-attempt-resume-upload --oos-leave-parts-on-error --no-check-dest /src/path oci:bucket
	```
	Additional flags used:
	- **`--stats-one-line`:** Make the stats fit on one line.
		- **`--max-stats-group`:** Maximum number of stats groups to keep in memory, on max oldest is discarded (default 1000).
		- **`--oos-attempt-resume-upload`:** Attempt to resume previously started multi-part upload for the object.
		- **`--oos-leave-parts-on-error`:** Avoid calling abort upload on a failure, leaving all successfully uploaded parts for manual recovery.

## Migrate a Large Number Files using Rclone

Rclone syncs on a directory-by-directory basis. If you are migrating tens of millions of files/objects, it is important to make sure the directories/prefixes are divided up into around 10,000 files/objects or lower per directory. This is to prevent Rclone from using too much memory and then crashing. Many customers with a high count (100’s of millions or more) of small files often run into this issue. If all your files are in a single directory, divide them up first.

1. Run the following command to get a list of files in the source.
	```md
	rclone lsf --files-only -R src:bucket | sort > src
	```
2. Break up the file into chunks of 1,000-10,000 lines, using split. The following split command will divide up the files into chunks of 1,000 and then put them in files named `src_##` such as `src_00`.
	```md
	split -l 1000 --numeric-suffixes src src_
	```
3. Distribute the files to multiple VM instances to scale out the data transfer. Each Rclone command should look like:
	```md
	rclone --progress --oos-no-check-bucket --no-traverse --transfers 500 copy remote1:source-bucket remote2:dest-bucket --files-from src_00
	```
	Alternatively, a simple for loop can be used to iterate through the file lists generated from the split command. During testing with ~270,000 files in a single bucket, we saw copy times improve 40x, your mileage may vary.
	> **Note:** Splitting up the files by directory structure or using the split utility is an important way to optimize transfers.

## Use Rclone, OKE and fpart together for Moving Data from File Systems to OCI Object Storage

Multiple Kubernetes pods can be used to scale out data transfer between file systems and object storage. Parallelization speeds up data transfers to storage systems that are relatively high latency and are high throughput. The approach combining Rclone, OKE and fpart partitions directory structures into multiple chunks and runs the data transfer in parallel on containers either on the same compute node or across multiple nodes. Running across multiple nodes aggregates the network throughput and compute power of each node.

- [Filesystem partitioner (Fpart)](http://www.fpart.org/#fpsync) is a tool that can be used to partition the directory structure. It can call tools such rsync, tar, and Rclone with a file system partition to run in parallel, and independent of each other. We will use fpart with Rclone.
- [fpsync](http://www.fpart.org/fpsync/) is a wrapper script that uses fpart to runs the transfer tools (rsync, Rclone) in parallel. The `fpsync` command is run from an fpsync operator host. The fpsync tool also has options to use separate worker nodes. The modified fpsync supports Rclone and also Kubernetes pods.
- [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/#install-kubectl-binary-with-curl-on-linux) manages Kubernetes jobs.

Follow the steps:

1. Identify a host that will be your fpsync operator host that has access to the migration source data and Rclone is installed.
2. Run the following command to install kubectl.
	```md
	# curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
	# chmod 755 kubectl
	# cp -a kubectl /usr/bin
	```
3. Create an OCI IAM policy for the fpsync operator host to manage the OKE cluster.
	The following policy can be used for this purpose. A more granular permission can be configured to achieve the bare minimum requirement to control the pods.
	```md
	Allow dynamic-group fpsync-host to manage cluster-family in compartment storage
	```
4. Setup the `kubeconfig` file to have access to the OKE cluster. For more information, see [Setting Up Local Access to Clusters](https://docs.oracle.com/en-us/iaas/Content/ContEng/Tasks/contengdownloadkubeconfigfile.htm#localdownload).
5. Install and patch fpart and fpsync. The fpsync patch is required to run Rclone or rsync in parallel to scale out the data transfer. The fpsync that comes with the fpart package does not support Rclone or Kubernetes pods, a patch is needed to support these tools.
	Run the following command to install on Ubuntu.
	```md
	# apt-get install fpart
	# git clone https://github.com/aboovv1976/fpsync-k8s-rclone.git
	# cd fpsync-k8s-rclone/
	# cp -p /usr/bin/fpsync /usr/bin/k-fpsync
	# patch /usr/bin/k-fpsync fpsync.patch
	```
6. Build the container image.
	The docker image build specification available in `rclone-rsync-image` can be used to build the container image. Once the image is built, it should be uploaded to a registry that can be accessed from the OKE cluster.
	```md
	# rclone-rsync-image
	# docker build -t rclone-rsync . 
	# docker login
	# docker tag rclone-rsync:latest <registry url/rclone-rsync:latest>
	# docker push <registry url/rclone-rsync:latest>
	```
	A copy of the image is maintained in `fra.ocir.io/fsssolutions/rclone-rsync:latest.` The sample directory contains some examples output files.
7. Run k-fpsync. The patched fpsync (k-fpsync) can partition the source file system and scale out the transfer using multiple Kubernetes pods. The Kubernetes pod anti-affinity rule is configured to prefer nodes that do not have any running transfer worker pods. This helps to utilize the bandwidth on the nodes effectively to optimize performance. For more information, see [Assigning Pods to Nodes](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/).
	Mount the source file system on the fpart operator host and create a shared directory that will be accessed by all the pods. This is the directory where all the log files and partition files are kept.
	The following command transfers data from the filesystem `/data/src` to the OCI Object Storage bucket rclone-2. It will start 2 pods at a time to transfer the file system partition created by fpart.
	```md
	# mkdir /data/fpsync
	# PART_SIZE=512 && ./k-fpsync -v -k fra.ocir.io/fsssolutions/rclone-rsync:latest,lustre-pvc  -m rclone -d /data/fpsync  -f $PART_SIZE -n 2 -o "--oos-no-check-bucket --oos-upload-cutoff 10Mi --multi-thread-cutoff 10Mi --no-check-dest --multi-thread-streams 64 --transfers $PART_SIZE  --oos-upload-concurrency 8 --oos-disable-checksum  --oos-leave-parts-on-error" /data/src/ rclone:rclone-2
	```
	> **Note:** The logs for the run are kept in the `run-ID` directory, in the following example they are in `/data/fpsync/{Run-Id}/log directory`. The sample outputs are provided in the sample directory.

## (Optional) Test Environments

Recommendations are made based on testing and customer interactions.

> **Note:** Runs from the bulk copy script, `os sync` and `s5cmd` results are included to give more information on performance. Learn about using the bulk copy script from here: [Use Oracle Cloud Infrastructure Object Storage Python Utilities for Bulk Operations](https://docs.oracle.com/en/learn/oci-objstorage-bulk-operation/index.html#introduction). For more information about using `os sync` and the `s5cmd`, see [Tutorial 3: Move Data into OCI Cloud Storage Services using OCI Object Storage Sync and S5cmd](https://docs.oracle.com/en/learn/migr-ocistorage-ossync-s5cmd-p3).

### Test Environment 1:

1 VM instance `VM.Standard.E4.Flex`, 1 OCPU, 1Gbps network bandwidth, 16GB of memory. To simulate on-premises to OCI migration copied data from PHX NFS to IAD.

**Data Sets**

- **Data Set 1:**
	| Total Size | File Count | File Size Range |
	| --- | --- | --- |
	| 3TB | 3 | 1TB |
	| Method | To-From | Time | Command | Flags |
	| --- | --- | --- | --- | --- |
	| os sync | NFS/File PHX to Object IAD | 123m17.102s | NA | `--parallel-operations-count 100` |
	| s5cmd | NFS/File PHX to Object IAD | 239m20.625s | copy | `run commands.txt`, default run `--numworkers 256` |
	| rclone | NFS/File PHX to Object IAD | 178m27.101s | copy | `--transfers=100 --oos-no-check-bucket --fast-list --checkers 64 --retries 2 --no-check-dest` |
	> **Note:** Our tests showed `os sync` running the fastest for this data set.
- **Data set 2:**
	| Total Size | File Count | File Size Range |
	| --- | --- | --- |
	| 9.787GB | 20,000 | 1MB |
	| Method | To-From | Time | Command | Flags |
	| --- | --- | --- | --- | --- |
	| s5cmd | NFS/File PHX to Object IAD | 1m12.746s | copy | default run `--numworkers 256` |
	| os sync | NFS/File PHX to Object IAD | 2m48.742s | NA | `--parallel-operations-count 1000` |
	| rclone | NFS/File PHX to Object IAD | 1m52.886s | copy | `--transfers=500 --oos-no-check-bucket --no-check-dest` |
	> **Note:** Our tests showed `s5cmd` performing the best for this data set.

### Test Environment 2:

**VM Instances:** 2 VM instances were used for each test, we used a `VM.Standard.E4.Flex` with 24 OCPU, 24Gbps network bandwidth, 384GB of memory. Oracle Linux 8 was used for Linux testing.

**Data sets used in testing:** 14 main directories with the following file count and sizes.

| Data Set Directory | Size | File count | Size of Each File |
| --- | --- | --- | --- |
| Directory 1 | 107.658 GiB | 110,242 | 1 MiB |
| Directory 2 | 1.687 GiB | 110,569 | 15 MiB |
| Directory 3 | 222 GiB | 111 | 2 GiB |
| Directory 4 | 1.265 TiB | 1,295 | 1 GiB |
| Directory 5 | 26.359 GiB | 1,687 | 16 MiB |
| Directory 6 | 105.281 MiB | 26,952 | 4 KiB |
| Directory 7 | 29.697 MiB | 30,410 | 1 KiB |
| Directory 8 | 83.124 GiB | 340,488 | 256 KiB |
| Directory 9 | 21.662 GiB | 354,909 | 64 KiB |
| Directory 10 | 142.629 GiB | 36,514 | 4 MiB |
| Directory 11 | 452.328 MiB | 57,898 | 8 MiB |
| Directory 12 | 144 GiB | 72 | 2GiB |
| Directory 13 | 208.500 GiB | 834 | 256 MiB |
| Directory 14 | 54.688 GiB | 875 | 64 MiB |

> **Note:**
> 
> - The 14 directories were split between the 2 VM instances.
> - Each VM ran 7 commands/processes, 1 for each directory unless otherwise noted.

| Method | To-From | Time | Command | Flags/ Notes |
| --- | --- | --- | --- | --- |
| s5cmd | NFS/File PHX to Object IAD | 54m41.814s | copy | `--numworkers 74` |
| os sync | NFS/File PHX to Object IAD | 65m43.200s | NA | `--parallel-operations-count 50` |
| rclone | NFS/File PHX to Object IAD | 111m59.704s | copy | ` --oos-no-check-bucket --no-check-dest --ignore-checksum --oos-disable-checksum --transfers 50` |
| rclone | Object PHX to Object IAD | 28m55.663s | copy | ` --oos-no-check-bucket --no-check-dest --ignore-checksum --oos-disable-checksum --transfers 400`, same command run across 2 VM’s for a concurrency of 800 transfers |
| python bulk copy script | Object PHX to Object IAD | 25m43.715s | Default | 1 VM, 50 workers, 100,000 files queued at a time |

The `s5cmd` and `os sync` commands do well over filesystem/NFS to object storage. The bulk copy script only does bucket-to-bucket transfers and was not tested for NFS migration.

Only `rclone` and the python bulk copy script are capable of doing bucket-to-bucket transfers across regions so the other tools were not tested for it. The python bulk copy script perfoms better on the cross region bucket-to-bucket data, however is only compatible with OCI Object Storage while `rclone` supports many backends and cloud providers.

Small test runs were conducted using `rclone` to transfer data from Microsoft Azure Blob Storage, Amazon Simple Storage Service (Amazon S3), and Google Cloud Platform Cloud Storage to OCI Object Storage to verify the tools works for these types of transfers. For more information, see [Move data to object storage in the cloud by using Rclone](https://docs.oracle.com/en/solutions/move-data-to-cloud-storage-using-rclone/configure-your-source1.html#GUID-C3869127-3570-43C8-9750-77417A0A58CF).

### Test Environment 3:

**VM Instances:** 1-2 VM instances were used for each test, we used a `VM.Standard.E4.Flex` with 24 OCPU, 24Gbps network bandwidth, 384GB of memory. Oracle Linux 8 was used for Linux testing. All tests were bucket-to-bucket.

| Total Size | File Count | File Size Range |
| --- | --- | --- |
| 7.74 TiB | 1,000,000 | 30 MiB |

| Method | To-From | Time | Command | Flags | Notes |  |
| --- | --- | --- | --- | --- | --- | --- |
| rclone | Object-to-Object IAD -> IAD | 18h39m11.4s | copy | `--oos-no-check-bucket --fast-list --no-traverse --transfers 500 --oos-chunk-size 10Mi` | 1 VM, very slow due to the high file count and listing calls to source |  |
| rclone | Object-to-Object IAD -> IAD | 55m8.431s | copy | `--oos-no-check-bucket --no-traverse --transfers 500 --oos-chunk-size 10Mi --files-from <file>` | 2 VM’s, 500 transfers per VM, object/file list fed 1,000 files at a time, prevents listing on source and destination and improves performance |  |
| python bulk copy script | Object-to-Object IAD -> IAD | 28m21.013s | NA | Default | 1 VM, 50 workers, 100,000 files queued at a time |  |
| python bulk copy script | Object-to-Object IAD -> IAD | NA | NA | Default | 2 VMs, 50 workers per VM, 100,000 files queued at a time. Received 429 errors, script hung and could not complete |  |
| s5cmd | Object-to-Object IAD -> IAD | 14m10.864s | copy | Defaults (256 workers) | 1 VM | NA |
| s5cmd | Object-to-Object IAD -> IAD | 7m50.013s | copy | Defaults | 2 VM’s, 256 workers each VM | Ran in abuot half the time as 1 VM |
| s5cmd | Object-to-Object IAD -> IAD | 3m23.382s | copy | `--numworkers 1000` | 1 VM, 1000 workers | Across multiple tests we found this was the optimal run for this data set with the s5cmd |
| rclone | Object-to-Object IAD -> PHX | 184m36.536s | copy | `--oos-no-check-bucket --no-traverse --transfers 500 --oos-chunk-size 10Mi --files-from <file>` | 2 VM’s, 500 transfers per VM, object/file list fed 1,000 files at a time |  |
| python bulk copy script | Object-to-Object IAD -> PHX | 35m31.633s | NA | Default | 1VM, 50 workers, 100,000 files queued at a time |  |

The `s5cmd` command ran consistently best for the large file count and small files. The `s5cmd` is limited because it can only do bucket-to-bucket copies within the same tenancy and same region.

Notice high improvements to `rclone` once files are fed to the command and from scaling out to another VM. Rclone may run slower than other tools, it is the most versatile in the various platforms it supports and types of migrations it can perform.

The OCI Object Storage Bulk Copy Python API can only use the OCI Native CopyObject API and can only get up to a concurrency of 50 workers before being throttled.

Tests for IAD to PHX were only done on what worked best in IAD to IAD and problematic tests were not re-run. The `s5cmd` was not run for IAD to PHX because it can only do bucket-to-buckets copies within the same region.

## Next Steps

Proceed to the related tutorial(s) relevant to your migration needs. To move data into OCI cloud storage services:

- Using OCI Object Storage Sync and S5cmd, see [Tutorial 3: Move Data into OCI Cloud Storage Services using OCI Object Storage Sync and S5cmd](https://docs.oracle.com/en/learn/migr-ocistorage-ossync-s5cmd-p3).
- Using Fpsync and Rsync for file system data migrations, see [Tutorial 4: Move Data into OCI Cloud Storage Services using Fpsync and Rsync for File System Data Migrations](https://docs.oracle.com/en/learn/migr-ocistorage-fpsync-rsync-p4).

## Related Links

- [Rclone](https://rclone.org/docs/)
- [Rclone Forum](https://forum.rclone.org/)
- [Migration Essentials for Moving Data into OCI Cloud Storage](https://oracle-livelabs.github.io/oci/oci-migration-essentials/workshops/freetier/)
- [Move data to object storage in the cloud by using Rclone](https://docs.oracle.com/en/solutions/move-data-to-cloud-storage-using-rclone/configure-rclone-object-storage.html)
- [Data Transfer between file systems and OCI Object Storage using OKE](https://github.com/aboovv1976/fpsync-k8s-rclone)
- [Tutorial 1: Use Migration Tools to Move Data into OCI Cloud Storage Services](https://docs.oracle.com/en/learn/migr-ocistorage-p1)
- [Tutorial 3: Move Data into OCI Cloud Storage Services using OCI Object Storage Sync and S5cmd](https://docs.oracle.com/en/learn/migr-ocistorage-ossync-s5cmd-p3)
- [Tutorial 4: Move Data into OCI Cloud Storage Services using Fpsync and Rsync for File System Data Migrations](https://docs.oracle.com/en/learn/migr-ocistorage-fpsync-rsync-p4)
- [Announcing native OCI Object Storage provider backend support in rclone](https://blogs.oracle.com/cloud-infrastructure/post/native-oci-object-storage-support-rclone)
- [Rclone OCI Object Storage Documentation](https://rclone.org/oracleobjectstorage/)
- [fpsync](https://www.fpart.org/fpsync/), [fpart](https://www.fpart.org/) and [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/#install-kubectl-binary-with-curl-on-linux)

## Acknowledgments

- **Authors** - Melinda Centeno (Senior Principal Product Manager, OCI Object Storage), Vinoth Krishnamurthy (Principal Member of Technical Staff, OCI File Storage), Aboo Valappil (Consulting Member of Technical Staff, OCI File and Block Storage)

## More Learning Resources

Explore other labs on [docs.oracle.com/learn](https://docs.oracle.com/learn) or access more free learning content on the [Oracle Learning YouTube channel](https://www.youtube.com/user/OracleLearning). Additionally, visit [education.oracle.com/learning-explorer](https://education.oracle.com/learning-explorer) to become an Oracle Learning Explorer.

For product documentation, visit [Oracle Help Center](https://docs.oracle.com/).

---
