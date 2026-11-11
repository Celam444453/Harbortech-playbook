# Week 10: Storage and Recovery Investigation

**Course:** IT 4560 | **Ticket:** TKT-2026-0010 | **Client:** Bright Path Community Services | **Consultant:** HarborTech Solutions
**Author:** Camden Elam | **Student ID:** 1522505 | **Date:** November 10, 2026

> **Evidence note:** Every command output below comes from my own AWS Learner Lab CloudShell session. Account numbers, credentials, and private endpoints are redacted. Items in `[brackets]` must be replaced with your real results before publishing. No results or resource IDs were invented.

---

## 1. Ticket Summary

| Requirement | Detail |
|---|---|
| Application disk recovery | Newest documented EC2 snapshot is **19 days old**; maximum acceptable data loss (RPO) is **24 hours** |
| Shared working files | Three Linux servers need **concurrent** access |
| Uploaded documents | Require reliable **object storage** |
| Completed records | Rarely accessed after 90 days; must be **retained seven years** |

**Evidence separation:** The 19-day-old snapshot is *client ticket evidence*. It is not a resource in my student account, and my practice resources do not remediate the client environment. No production changes are authorized.

---

## 2. AWS Environment (sanitized)

- Account: `[redacted]`
- Identity type: `[assumed-role / role name]`
- Region used for all commands: `[region]`
- AWS CLI version: `[version]`
- Tool: AWS CloudShell via AWS Academy Learner Lab

---

## 3. Investigation Findings

### Section 1: AWS Environment Verification

**Commands executed**
```bash
aws sts get-caller-identity
aws --version
aws configure get region
# if region was blank:
aws configure list
echo "$AWS_DEFAULT_REGION"
```

**Actual output (sanitized)**
```
[paste real output, redact Account]
```

**Q1. What do my identity and Region results establish?**
My `get-caller-identity` returned `[assumed-role ARN / role name]`, showing I work through temporary Learner Lab credentials in a sandbox account (`[redacted]`). The Region `[region]` is where all later commands ran. This establishes that my resources are disposable lab resources, separate from Bright Path's environment.

**Q2. Why verify the account and Region first?**
Storage resources are account- and Region-scoped. A volume, snapshot, or bucket created or deleted in the wrong place would not appear where an administrator expects, and a destructive command could hit the wrong resource. Verifying first confirms commands act on the intended environment.

---

### Section 2: EBS Volume Investigation

**Commands executed**
```bash
aws ec2 describe-availability-zones --region [region]
aws ec2 create-volume --availability-zone [az] --size 1 --volume-type gp3 \
  --tag-specifications "ResourceType=volume,Tags=[{Key=Purpose,Value=IT4560Week10}]" --region [region]
aws ec2 describe-volumes --volume-ids [vol-id] --region [region]
```

**Actual output / error (sanitized)**
```
[paste real output or exact error]
```

**Q1. What do state, size, type, and AZ reveal?**
Volume `[vol-id]` is `[State]` with `[Attachments]`, `[1]` GiB, type `[gp3]`, in `[AZ]`. EBS is block storage tied to a single Availability Zone and attached to one instance, so this configuration suits an instance disk, not shared storage. I did not attach it.

**Q2. What risks come from selecting the wrong volume?**
Snapshotting the wrong volume creates a recovery point for the wrong data and gives false confidence that the application is protected. Restoring from the wrong snapshot can replace good data with stale or unrelated data. The AZ and size matter because a restore must land in the right AZ. Verifying volume ID, attachment, tags, and size first prevents recovering the wrong system.

---

### Section 3: EBS Snapshot and Recovery

**Commands executed**
```bash
aws ec2 create-snapshot --volume-id [vol-id] --description "IT4560 Week10 practice" \
  --tag-specifications "ResourceType=snapshot,Tags=[{Key=Purpose,Value=IT4560Week10}]" --region [region]
aws ec2 describe-snapshots --snapshot-ids [snap-id] --region [region]
```

**Actual output / error (sanitized)**
```
[paste real output, include pending and completed states if captured]
```

**Q1. What does my snapshot status demonstrate?**
Snapshot `[snap-id]` showed `[State / Progress]`. A `completed` state shows a point-in-time copy exists independently of the source volume and is usable as a recovery point. A `pending` snapshot is not yet a usable recovery point. `[I did / did not perform a restore test, so I make no claim about restore success.]`

**Q2. Why doesn't a completed snapshot prove the 24-hour RPO?**
A completed snapshot proves only that a copy exists at one moment. A 24-hour RPO requires recovery points at least every 24 hours, which means automation, monitoring of snapshot age, and a tested restore. Bright Path's newest snapshot is 19 days (about 456 hours) old against a 24-hour limit, so the requirement fails regardless of whether that snapshot completed successfully.

---

### Section 4: S3 Object Storage

**Commands executed**
```bash
aws s3api create-bucket --bucket [unique-name] --region [region]
# non-us-east-1 regions also need: --create-bucket-configuration LocationConstraint=[region]
printf "IT4560 Week10 practice\n" > week10-sample.txt
aws s3 cp week10-sample.txt s3://[bucket]/week10-sample.txt --region [region]
aws s3api head-object --bucket [bucket] --key week10-sample.txt --region [region]
```

**Actual output / error (sanitized)**
```
[paste real output or exact error]
```

**Q1. What do the bucket and object results confirm?**
`head-object` for `week10-sample.txt` in `[bucket]` returned `ContentLength [value]`, ETag `[value]`, `LastModified [timestamp]`, and `[encryption setting]`. `StorageClass` is `[absent = Standard]`. The data is stored as a whole object identified by bucket and key and accessed through the S3 API with no mount or file system.

**Q2. How does object storage differ from block storage?**
EBS presents a raw volume that an OS formats and mounts, with low-latency random reads and writes, attached to one instance in one AZ. S3 stores whole objects by key, accessed over an API or HTTP, with Region-wide scale. My evidence shows the contrast: the volume had an AZ and attachment state, while the object had a key, ETag, and metadata and needed neither.

---

### Section 5: S3 Versioning and Lifecycle

**Commands executed**
```bash
aws s3api get-bucket-versioning --bucket [bucket] --region [region]
aws s3api put-bucket-versioning --bucket [bucket] --versioning-configuration Status=Enabled --region [region]
aws s3api get-bucket-versioning --bucket [bucket] --region [region]
aws s3api get-bucket-lifecycle-configuration --bucket [bucket] --region [region]
python3 -m json.tool lifecycle.json
```

**Proposed lifecycle rule (practice prefix only, no expiration)**
```json
{
  "Rules": [
    {
      "ID": "practice-archive-rule",
      "Status": "Enabled",
      "Filter": { "Prefix": "practice/" },
      "Transitions": [
        { "Days": 90, "StorageClass": "STANDARD_IA" },
        { "Days": 365, "StorageClass": "GLACIER" }
      ]
    }
  ]
}
```

**Actual output / error (sanitized)**
```
[paste real output, including json.tool result]
```

**Q1. What do versioning and lifecycle results demonstrate?**
Before enabling, versioning returned `[empty]`; afterward it returned `[Status: Enabled]`, so overwrites and deletes preserve prior versions. The lifecycle check returned `[NoSuchLifecycleConfiguration]`, so no automated transition or expiration was active. My JSON `[passed]` local validation. Versioning protects against accidental change; lifecycle governs cost and retention over time. The rule was not applied to any client data.

**Q2. What if an expiration rule were set without confirming seven-year retention?**
Expiration permanently removes objects after the stated period. If records must be kept seven years and a rule expires them sooner, they would be deleted early, causing compliance exposure and unrecoverable loss. Transitions change storage class; expiration deletes. HarborTech must confirm the retention requirement, and consider S3 Object Lock, before any expiration is configured.

---

### Section 6: HarborTech Storage Recommendation

| Data category | Recommended model | Why | Weaker alternatives |
|---|---|---|---|
| Application disk | **Amazon EBS** + scheduled snapshots (AWS Backup or Data Lifecycle Manager) | Low-latency random read/write block storage the OS mounts; snapshots are stored separately and provide recovery points | Instance store loses data when the instance stops or fails; S3 and EFS do not present a boot or application disk |
| Shared Linux working files | **Amazon EFS** | Three servers mount it concurrently with file-system semantics (paths, permissions, locking) | EBS attaches to one instance; S3 is not a mountable locking file system |
| Uploaded documents | **Amazon S3** | Key-based, durable, scalable API/HTTP access; versioning protects against overwrite | EBS is single-instance; EFS is costly for large object volumes |
| Seven-year records | **S3 with lifecycle to Glacier tiers** (+ **Object Lock** if retention is mandated) | Rarely accessed after 90 days, so low-cost archival fits; Object Lock prevents early deletion | EBS and EFS cost far more for rarely accessed data |

**Observed vs. proposed**
- *Observed (my lab evidence):* [cite Sections 1 to 5 results]
- *Ticket evidence:* 19-day-old snapshot, 24-hour RPO, three-server concurrency, seven-year retention
- *Proposed (not implemented):* everything in the table above

**Q1. Which models best satisfy the four requirements, and what evidence supports them?**
EBS with scheduled snapshots, EFS, S3, and S3 lifecycle with Glacier (plus Object Lock if required), matched to the access patterns above. Supporting evidence: my volume showed AZ-bound, single-instance block characteristics (Section 2); my object showed key-based API access with no mount (Section 4); versioning and lifecycle behavior showed S3 retention controls (Section 5).

**Q2. Which decisions need verification or approval?**
- Snapshot schedule and a tested restore to prove the 24-hour RPO
- The exact retention period and whether it is legally mandated, which determines Object Lock and any expiration
- Migrating shared files to EFS, which needs a compatibility and cutover plan
- Glacier retrieval delays (minutes to hours) and retrieval fees, which the client must accept

No production changes are authorized, so all of these require client approval.

---

### Section 7: Resource Cleanup

**Commands executed**
```bash
aws s3api list-object-versions --bucket [bucket] --region [region]
aws s3api delete-object --bucket [bucket] --key week10-sample.txt --version-id [version-id] --region [region]
aws s3api delete-bucket --bucket [bucket] --region [region]
aws ec2 delete-snapshot --snapshot-id [snap-id] --region [region]
aws ec2 delete-volume --volume-id [vol-id] --region [region]
# verification
aws ec2 describe-snapshots --snapshot-ids [snap-id] --region [region]
aws ec2 describe-volumes --volume-ids [vol-id] --region [region]
aws s3api list-buckets
```

**Actual output / error (sanitized)**
```
[paste real output]
```

**Q1. What confirms removal, or what remains?**
`describe-snapshots` returned `[InvalidSnapshot.NotFound]`, `describe-volumes` returned `[InvalidVolume.NotFound]`, and the bucket `[no longer appears in list-buckets]`. I deleted all object versions and delete markers before the bucket. Remaining: `[nothing / describe any failure]`. I deleted only resources I created.

**Q2. Why does cleanup verification matter?**
Leftover volumes, snapshots, and buckets keep incurring charges and can exhaust a limited lab budget, and they leave data in place that could be exposed. A delete command that appears to succeed may leave versions, delete markers, or resources behind. Confirming with NotFound errors turns an assumption into evidence.

---

## 4. Recovery and Retention Verification

| Requirement | Verification needed | Status |
|---|---|---|
| 24-hour RPO | Automated snapshot schedule; check newest snapshot age; perform a restore test | Not verified in client environment (ticket evidence only) |
| Seven-year retention | Confirm legal period; lifecycle transitions; Object Lock or retention policy | Proposed, not applied |
| Restore capability | Create a volume from a snapshot and confirm data integrity | `[Performed / Not performed in lab]` |

---

## 5. Escalation Notes

- Escalate to the client for approval before any snapshot schedule, EFS migration, lifecycle rule, or Object Lock change.
- Escalate to the client's compliance or legal contact to confirm the seven-year requirement and whether records are under a legal hold.
- `[Document any Learner Lab permission errors you hit: command, error message, and operational implication.]`

---

## 6. Lessons Learned

- A snapshot's existence is not the same as meeting a recovery objective. The schedule, monitoring, and a tested restore are what prove it.
- Match storage to access pattern first (block, file, object), then optimize cost.
- Retention rules need explicit confirmation before any expiration is configured.
- Verify account and Region before every change, and verify cleanup afterward.
- `[Add one or two personal lessons from your own session.]`
