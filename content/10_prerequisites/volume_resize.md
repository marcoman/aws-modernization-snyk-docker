---
title: "Resize workspace's EBS volume"
chapter: false
weight: 14 
---

{{% notice warning %}}
The Cloud9 workspace volume should be resized by an IAM user with Administrator privileges,
not the root account user. Please ensure you are logged in as an IAM user, not the root
account user.
{{% /notice %}}

{{% notice info %}}
This workshop will require more volume on the EBS storage attached to the Cloud9 workspace instance. Thie set of instructions will show how to resize the volume.
{{% /notice %}}

### Set up resize script:

- Create a file called `resize.sh` to resize the root EBS volume of the Cloud9 instance.
```bash
$ touch resize.sh && chmod +x resize.sh
```
- Double click `resize.sh` on the file bar on the left and copy in the script below. Make sure to save it. 
```bash
#!/bin/bash

# Specify the desired volume size in GiB as a command line argument. If not specified, default to 20 GiB.
SIZE=${1:-20}

# Get the ID of the environment host Amazon EC2 instance.
INSTANCEID=$(curl http://169.254.169.254/latest/meta-data/instance-id)

# Get the ID of the Amazon EBS volume associated with the instance.
VOLUMEID=$(aws ec2 describe-instances \
    --instance-id $INSTANCEID \
    --query "Reservations[0].Instances[0].BlockDeviceMappings[0].Ebs.VolumeId" \
    --output text)

echo Modify volume ${VOLUMEID}

# Resize the EBS volume.
aws ec2 modify-volume --volume-id $VOLUMEID --size $SIZE

# Wait for the resize to finish.
seconds=1

while [ \
        "$(aws ec2 describe-volumes-modifications \
            --volume-id $VOLUMEID \
            --filters Name=modification-state,Values="optimizing","completed" \
            --query "length(VolumesModifications)"\
            --output text)" != "1" ]; do
        sleep 1
        let seconds++
done

echo ${seconds} seconds to resize disk

#Check if we're on an NVMe filesystem
if [ $(readlink -f /dev/xvda) = "/dev/xvda" ]
then
        # Rewrite the partition table so that the partition takes up all the space that it can.
        sudo growpart /dev/xvda 1

        # Expand the size of the file system.
        # Check if we are on AL2
        STR=$(cat /etc/os-release)
        SUB="VERSION_ID=\"2\""
        if [[ "$STR" == *"$SUB"* ]]
        then
            sudo xfs_growfs -d /
        else
            sudo resize2fs /dev/xvda1
        fi

else
        # Rewrite the partition table so that the partition takes up all the space that it can.
        sudo growpart /dev/nvme0n1 1

        # Expand the size of the file system.
        # Check if we're on AL2
        STR=$(cat /etc/os-release)
        SUB="VERSION_ID=\"2\""
        if [[ "$STR" == *"$SUB"* ]]
        then
            sudo xfs_growfs -d /
        else
            sudo resize2fs /dev/nvme0n1p1
        fi
fi
```

### Increase the volume size

- Going back to the terminal, run `resize.sh` using the command bellow. This will resize the volume to 20 GiB
```bash
$ ./resize.sh 20
```
- The script may take some time to complete. 

### Verify the volume was resized succesffully. 
- Run `lsblk` to ensure that the disk is partitioned properly. The size of the partition with the`/` mountpoint and the overarching drive should be 20G.
```bash
$ lsblk
NAME          MAJ:MIN RM SIZE RO TYPE MOUNTPOINT
nvme0n1       259:0    0  20G  0 disk 
├─nvme0n1p1   259:1    0  20G  0 part /
└─nvme0n1p128 259:2    0   1M  0 part 
```