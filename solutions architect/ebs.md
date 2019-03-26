# Elastic Block Store (EBS)

Encryption/copying/sharing related tasks

## Overview

* Basically a NAS
* Resides on an AZ
* Persistant (lasts beyond starts/stops)
* Alternative: Instance Stores
  * Temporary / Ephemeral
  * 10GB Limit per volume
  * You can reboot an instance and the data will persist (but "stop"/terminating will destroy it -- you can't actually stop it)
  * IOPS is faster on the instance store (no network transfers to reach EBS)
* Behaves like raw, unformatted drives
* Suitable for database style data that requires frequent reads/writes
* EBS volume can only be used by one EC2 instance at a time (but it can be moved to another instance)
* EBS volume and EC2 instance must in the same AZ
* EBS data is replicated across multiple servers in a single AZ
* 5k EBS volumes can be created

## Root Volumes

* Can be Instance store-backed or EBS-backed
* If an EBS backed instance is terminatd, then it will be deleted
  * You can change this behavior (set the DeteleOnTermination flag to false)
  * and it only applies to the root volume

## EBS Snapshots

* Point-in-time images/copies of EBS volumes
* Future snapshots are incremental
* 10k snapshots can be created per account
* Saved in S3, but not accessible directly (only via APIs)
* Instance Store AMIs are stored on your own s3 bucket
* Snapshots are _Region_ specific
* You can restore a snapshot to the same size, or larger EBS volume
* Sanpshot is created immediately, but data takes time to copy over
* Data is available while being snapshotted, but performance will be impacted
  * Writes that happen at this time are not reflected in the snapshot (reads too, obviously)
* Snapshot starts in [pending] state and when complete, moves to the [complete] state
* To ensure a complete snapshot, you need to stop EBS i/o or unmount the volume if possible 
* To snapshot the root/boot volume, you must stop the volume
  * Note that if you do this, any instance store volumes will be lost (obviously)
* You can detach the ebs volume, start a snapshot (to ensure that you get everything) and then immediately reattach, rather than waiting for the snapshot to compete
* snapshots are async
* When creating a snapshot, you will be charged for
  * Data in to S3
  * S3 storage
* You can copy snapshots across regions, to enable restoring snapshots on a new region

### EBS Snapshot Sharing

* By default, only the account owner can use snapshots to create volumes
* You can share unencrypted snapshots with the AWS community (making them public)
  * This can be achieved by setting the snapshot permissions
* You can share with specific accounts by leaving the permissions private
* You can still share encrypted snapshots with specific accounts (but you can't make it public)
  * So long as the key used to encrypt is not the default CMK
  * You will need to configure the cross account permissionn
  * You also need to share access to the key
* When sharing a snapshot:
  * Target AWS account needs to create their own copy of the snapshot
  * They can then use their version to create ebs volumes
  * If the snapshot is encrypted, they can also opt to re-encrypt the data with a different key to have sole access to that snapshot (this is a recommended practice)
    * If you don't do this, it's possible to lose access to your data if the person that shared the snapshot with you revokes access to their encryption keys

### EBS Snapshot Copying

* You can copy an unencrypted snapshot
  * During a copy, you can request encryption for the copy
* You can copy an encrypted snapshot
  * During a copy, you can request encryption for the copy with a different key
* Snapshot copies recieve a new snapshot id
* You can make a copy of a snapshot when it has been fully saved to s3 (when state is [completed])
  * Goes from s3 to s3 (uses server side encryption)
* Tags are not copied (user tags specifically)
* You can run up 5 snapshot copies concurrently
* Why Copy?
  * Geographic Expansion
  * Disaster Recovery (store on another region)
  * Migrating to another region
  * Encrypting unencrypted volumes
  * Data region/Auditing requirements

## EBS Encryption

* All EBS volumes types support encrypted
* Snapshots of encrypted volumes are also encrypted
* You can encrypt while the ebs volume is at rest (not sure entirely what this means)
  * Use 3rd party encryption tools
  * Use an encrypted ebs volume
  * Use encryption at the OS level
  * Encrypt at the app level
  * Use an encrypted FS
* Encryption at the EBS level is end-to-end
* Restoring an encypted snapshot creates an encrypted volume
* No easy way to swap encryption flag
* An indirect way
  * Create a new, encrypted ebs volume
  * Mount the volume to an ec2 instance
  * Copy data to the encrypted volume (rsync?)
* Another indirect way
  * Create a snapshot of the un-encrypted volume
  * You can encrypt the data on the snapshot copy
    * (You can also change encryption keys here)
  * You can then restore the encrypted snapshot to a new ebs volume
* You can mix and match encrypted/unencrypted volumes on the same ec2 instance (e.g. unencrypted root / encrypted data volumes)

### Encryping Root Volumes

* You cannot _create_ an encrypted root volume
* But you can do it via this convoluted process
  * Launch the desired instance (un-encrypted)
  * Patch/Install, etc to get to the desired state, sans encryption
  * Create an AMI (amazon machine image) from that instance
  * Copy the AMI, and choose to encrypt the image during the copy

### Encryption keys

* You need to create an encryption key (called Customer Master Keys (CMKs))
* These keys are managed by AWS Key Management Service (KMS)
* AWS Will create a default CMK key when creating the first EBS volume
  * This key is used for the first volume, also for encrypting snapshots of this volumes, and subsequent volumes from the snapshot
* New volumes will use unique keys
* Encryption is AES256
  * Each key is used to encrypt
    * The volume
    * sanpshots of the volume
    * restored snapshots of the volume
* Changing Keys
  * You cannot change keys for any existing encrypted volume
  * But you can change it by creating a copy of the snapshot, and during the copy process, you can choose to encrypt with a different key
  * If you create your own keys, you can then share the keys with other accounts

## Creating and Registering AMIs

AMIs are base images that are used to launch new EC2 instances

* Creating an AMI can be prepared for your use-cases, saving time on initial configuration
* To change the AMI, you need to de-register and then re-register to have changes reflected
* When an instance is created via an instance store AMI, the image is copied from s3 and applied to the root
* Remember to de-register the AMI when no longer needed
* Once deregistered, the AMI will no longer be available to create new instances
  * But instances currently using the AMI remain in state

### Instance Store backed AMIs

* To create an instance store backed AMI:
  * Launch EC2 Instance from an Instance-store AMI
  * Update root volume as needed (e.g. patching, updating)
  * Create the AMI, which will store the AMI on S3 (stored on YOUR S3 bucket)
  * Register the AMI (manually) -- this makes it available on the marketplace (either public for everyone, or private for just you)
  * S3 charges will apply until you de-register the AMI, and delete the S3 objects

### EBS Backed AMIs

* Prior to creating an AMI from an EBS image, you must stop the instance (to ensure data consistency)
  * Not a strict requirement, but a good idea
* AMIs created from EBS backed instances are automatically registered
* During the AMI-creation process, EC2 creates snapshots of your root volume and any EBS volume attached to the instance
* EBS image is stored in Amazon's S3 bucket, not yours
* Once you no longer need the image, delete the volume
  * You need to deregister first
* An AMI that uses encrupted volumes will only luanch on instances that support encryption
* To delete an EBS AMI
  * Deregister the image
  * delete the snapshot

## Raid

* You can raid your EBS discs for higher performance
* Raid is handled at the OS limitation
* You'll need to match or exceed the network IOPS with the raid IOPS to optimally configure these
* Supports Raid 0 (stripping -- write to multiple discs in parallel -- effective single large disc)
  * Faster I/O
  * More brittle solution
* Supports Raid 1 (Mirroroing -- store the same bytes on multiple discs)
  * Same speed
  * More robust solution
* Supports Raid 10 (mirrored and stripped)
* Recommended not to use raid on the boot volume

