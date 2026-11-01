# rclone + Amazon S3 Glacier Manual

Cost-first cold storage with rclone: **local (client-side) encryption**,
**Deep Archive**, and **Bulk** retrieval.

Rclone talks to the **S3 API only**. It cannot use the old Glacier Vault API.

Official references:

- [rclone S3 backend](https://rclone.org/s3/)
- [Glacier section](https://rclone.org/s3/#glacier-and-glacier-deep-archive)
- [backend restore](https://rclone.org/s3/#restore)
- [rclone settier](https://rclone.org/commands/rclone_settier/)
- [rclone crypt](https://rclone.org/crypt/)
- [AWS restoring archived objects](https://docs.aws.amazon.com/AmazonS3/latest/userguide/restoring-objects.html)

---

## Recommended defaults (this manual)

| Choice | Default | Why |
|--------|---------|-----|
| Encryption | rclone **`crypt`** (local / client-side) | Keys never leave your machine; S3 only stores ciphertext |
| Storage class | **`DEEP_ARCHIVE`** | Lowest $/GB for rarely accessed data |
| Restore priority | **`Bulk`** | Cheapest Deep Archive retrieval (~48 hours) |
| Day-to-day remote | **`secret:`** (crypt) | Encrypt on upload, decrypt on download |
| Restore remote | **`mys3:`** (underlying S3) | `crypt` has no `backend restore` command |

Trade-off: restores are slow (about two days with Bulk). Use `Standard`
(~12 hours on Deep Archive) only when you need it sooner. `Expedited` is
**not available** for Deep Archive.

**Always pass both restore options explicitly:**

- `-o priority=Bulk` — if omitted, rclone sends no `GlacierJobParameters`
  and **AWS defaults to `Standard`** (not Bulk)
- `-o lifetime=DAYS` — **required** for `DEEP_ARCHIVE` / `GLACIER` restores
  (`Days` in `RestoreObject`; minimum 1)

### Verification status (AWS docs + rclone SDK)

Cross-checked against AWS S3/IAM docs and rclone `backend/s3` (AWS SDK for
Go v2). The default stack **works** if you follow the required flags and
restore-before-reclass rule below.

| Area | Result |
|------|--------|
| Upload with `storage_class = DEEP_ARCHIVE` | OK — rclone sets `StorageClass` on Put/multipart |
| Client-side `crypt` over S3 | OK — encrypt then upload; class applies to ciphertext |
| `backend restore` → `RestoreObject` + `GlacierJobParameters.Tier` | OK |
| Bulk / Standard timings; no Expedited on Deep Archive | OK — matches RestoreObject docs |
| Temporary restore copy; class stays `DEEP_ARCHIVE` | OK — archival-storage / restoring-objects |
| Permanent hot object via `settier` / `CopyObject` after restore | OK — AWS requires restore before copy from Deep Archive |
| `crypt` has no `restore`; use underlying S3 + encrypted keys | OK |
| Single-object restore = directory Fs + `--include` | OK — rclone NewFs file → “is a file not a directory” |
| IAM actions in §2 | OK — see permission map; includes `RestoreObject` + `PutObjectAcl` |

Known cost footguns (AWS, not rclone bugs): **180-day** Deep Archive minimum
duration; **40 KB** metadata overhead per archived object; while restored you
pay for **archive + temporary Standard copy** (temp copy is **automatic** for
`lifetime` days — see §1); Deep Archive **Bulk is lowest-cost, not free**;
optional **CloudFront** can cut egress (§7) but not retrieval/temp-copy fees.

---

## 1. Concepts

### Storage classes (AWS)

| Class | Readable immediately? | Role in this guide |
|-------|------------------------|--------------------|
| `DEEP_ARCHIVE` | **No** | **Default** cold class |
| `GLACIER` | **No** | Flexible Retrieval (faster/costlier than Deep Archive) |
| `GLACIER_IR` | Yes | Instant Retrieval — not the cost default |
| `STANDARD` | Yes | Hot storage after a permanent reclass |
| `STANDARD_IA` / `ONEZONE_IA` | Yes | Infrequent access |
| `INTELLIGENT_TIERING` | Frequent / Infrequent / Archive Instant: yes; Archive Access / Deep Archive Access: **no** | Optional archive tiers need restore |

Listing and metadata work on archived objects. **Reading content** of
`DEEP_ARCHIVE` / `GLACIER` fails until restore finishes:

```text
Object in DEEP_ARCHIVE, restore first: path/to/file
```

### What rclone can and cannot do

| Task | How |
|------|-----|
| Upload encrypted into Deep Archive | `crypt` → S3 with `storage_class = DEEP_ARCHIVE` |
| Change class of existing objects | `rclone settier` (via `secret:` or `mys3:`) |
| Temporarily restore archives | `rclone backend restore` on **`mys3:`** |
| Check restore progress | `rclone backend restore-status` on **`mys3:`** |
| Delete objects | `rclone deletefile` / `delete` / `purge` on **`secret:`** |
| Inspect class / tier | `rclone lsf --format pT` |
| Lifecycle transition rules | **AWS Console / CLI** (not rclone) |
| Glacier Vaults | **Not supported** |

### Temporary restored copy (automatic — you only pick how many days)

For `DEEP_ARCHIVE` / `GLACIER`, restore is **not** a permanent class change.

| Question | Answer (AWS) |
|----------|----------------|
| Who creates the readable copy? | **Amazon S3, automatically**, after your `RestoreObject` finishes |
| Who deletes it? | **Amazon S3, automatically**, at expiry — you do not delete it by hand |
| How long does it last? | The **`lifetime` / `Days`** you pass (rclone: `-o lifetime=N`, minimum **1**) |
| How is expiry computed? | Completion time + `Days`, then **rounded up to next midnight UTC** |
| Does storage class change? | **No** — `HeadObject` / listings still show `DEEP_ARCHIVE` |
| What do you pay while restored? | **Deep Archive storage + temporary Standard-rate copy** for that object |
| How to keep it longer? | Re-run restore with a new `lifetime` (extends from **now**; request fee only) |
| How to keep it forever as hot? | After restore completes: `settier STANDARD` or copy to a hot class (**manual**) |

AWS example: restore completes 15 Oct 2012 10:30 UTC with `Days=3` → temporary
copy expires **19 Oct 2012 00:00 UTC**, then S3 deletes only the temporary copy;
the archived object remains.

A bucket **lifecycle expiration** rule can delete the object earlier than your
`lifetime`. Intelligent-Tiering archive restores are different: no `Days`; the
object moves to Frequent Access until idle tiering moves it again.

### Retrieval priority (Deep Archive)

| Priority | Typical wait | Cost | Use when |
|----------|--------------|------|----------|
| **`Bulk`** (default here) | ~48 hours | Lowest | Planned / non-urgent restores |
| `Standard` | ~12 hours | Higher | Need it sooner |
| `Expedited` | — | — | **Not available** for Deep Archive |

---

## 2. Create an IAM access key for rclone

Verified against current AWS IAM/S3 docs and against the APIs rclone’s AWS SDK
v2 client actually calls in `backend/s3/s3.go`.

**There is no “bucket access key”.** Access keys belong to an **IAM user** (or
you use a role’s temporary credentials). You attach a **bucket-scoped identity
policy**, then create an access key pair for rclone’s `access_key_id` /
`secret_access_key`.

Sources:

- [Manage access keys for IAM users](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_credentials_access-keys.html)
- [CreateAccessKey API](https://docs.aws.amazon.com/IAM/latest/APIReference/API_CreateAccessKey.html)
- [Identity-based policy examples for Amazon S3](https://docs.aws.amazon.com/AmazonS3/latest/userguide/example-policies-s3.html)
- [Required permissions for Amazon S3 API operations](https://docs.aws.amazon.com/AmazonS3/latest/userguide/using-with-s3-policy-actions.html)
- [RestoreObject API](https://docs.aws.amazon.com/AmazonS3/latest/API/API_RestoreObject.html)
- [Working with archived objects](https://docs.aws.amazon.com/AmazonS3/latest/userguide/archived-objects.html)
- [Actions, resources, and condition keys for Amazon S3](https://docs.aws.amazon.com/service-authorization/latest/reference/list_amazonS3.html)

### 2.1 Create the IAM user

AWS Console → **IAM** → **Users** → **Create user** (e.g. `rclone-cold-backup`).
Console login is optional for rclone (programmatic access only).

CLI:

```bash
aws iam create-user --user-name rclone-cold-backup
```

### 2.2 Attach a bucket-only policy

Prefer an **IAM identity policy on the user** (same pattern as AWS’s
“allowing an IAM user access to one of your buckets” example). Replace
`YOUR-BUCKET` and save as e.g. `rclone-s3-policy.json`:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "BucketLevel",
      "Effect": "Allow",
      "Action": [
        "s3:ListBucket",
        "s3:GetBucketLocation",
        "s3:ListBucketMultipartUploads"
      ],
      "Resource": "arn:aws:s3:::YOUR-BUCKET"
    },
    {
      "Sid": "ObjectLevel",
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject",
        "s3:PutObjectAcl",
        "s3:DeleteObject",
        "s3:AbortMultipartUpload",
        "s3:ListMultipartUploadParts",
        "s3:RestoreObject"
      ],
      "Resource": "arn:aws:s3:::YOUR-BUCKET/*"
    }
  ]
}
```

#### Why each action (AWS permission map ↔ rclone SDK calls)

| IAM action | AWS API(s) it authorizes | rclone use |
|------------|--------------------------|------------|
| `s3:ListBucket` | `ListObjects` / `ListObjectsV2`; also `HeadBucket` | Listing, sync, `no_check_bucket=false` existence check |
| `s3:GetBucketLocation` | `GetBucketLocation` | Region discovery (also in AWS’s single-bucket example) |
| `s3:ListBucketMultipartUploads` | `ListMultipartUploads` | Cleanup / list incomplete uploads |
| `s3:GetObject` | `GetObject`, **and** `HeadObject` | Download, metadata, restore-status listing details |
| `s3:PutObject` | `PutObject`, `CreateMultipartUpload`, `UploadPart`, `CompleteMultipartUpload`; destination side of `CopyObject` / `UploadPartCopy` | Upload, large files, **`settier`** (same-key copy with new `StorageClass`) |
| `s3:PutObjectAcl` | Conditionally required when the request sets an ACL | rclone config `acl = private` sends a canned ACL on put/copy |
| `s3:DeleteObject` | `DeleteObject` | Sync deletes / `deletefile` |
| `s3:AbortMultipartUpload` | `AbortMultipartUpload` | Failed/cancelled multipart cleanup |
| `s3:ListMultipartUploadParts` | `ListParts` | Multipart bookkeeping |
| `s3:RestoreObject` | `RestoreObject` | Deep Archive / Glacier / IT archive restore |

Resource split is required by AWS: bucket ARN for list/location APIs; object ARN
(`bucket/*`) for object APIs. Both appear in AWS’s own single-bucket examples.

#### Confirmed Glacier / Deep Archive behaviour (AWS)

| Claim in this manual | AWS confirmation |
|----------------------|------------------|
| Deep Archive needs restore before read | Archived classes are not real-time accessible; initiate restore first |
| Restore is a **temporary copy**; class stays `DEEP_ARCHIVE` | `HeadObject`/`GetObject` still report Deep Archive after restore |
| Permanent hot object = copy / change class after restore | AWS: create a copy in Standard (rclone: `settier` / `CopyObject`) |
| `Bulk` ≈ 48h on Deep Archive; cheapest Deep Archive tier | RestoreObject docs: Bulk “typically finish within 48 hours” for Deep Archive |
| `Standard` ≈ 12h on Deep Archive | RestoreObject docs |
| `Expedited` **not available** for Deep Archive | RestoreObject docs + archived-objects guide |
| Intelligent-Tiering restore ignores `Days` | Archived-objects guide; rclone clears `Days` for `INTELLIGENT_TIERING` |

#### Notes / least privilege

- Omit `s3:CreateBucket` and `s3:ListAllMyBuckets` when the bucket already
  exists and rclone has `no_check_bucket = true` (avoids `HeadBucket` /
  `ListBuckets`). Test against `mys3:YOUR-BUCKET`, not bare `mys3:`.
- Local **crypt** needs **no** KMS permissions. If the bucket also uses
  **SSE-KMS**, AWS requires `kms:Encrypt`, `kms:Decrypt`, and
  `kms:GenerateDataKey` on that CMK for put/get/copy/multipart.
- If you enable Object Lock / versioning workflows, you may also need
  versioned variants (`GetObjectVersion`, `DeleteObjectVersion`, …) and
  lock actions — not required for this manual’s default path.

Attach the policy:

```bash
aws iam put-user-policy \
  --user-name rclone-cold-backup \
  --policy-name rclone-s3-YOUR-BUCKET \
  --policy-document file://rclone-s3-policy.json
```

### 2.3 Create the access key

Per IAM docs, access keys are **long-term credentials** for an IAM user.
AWS recommends temporary credentials (roles / IAM Identity Center) when
possible; access keys are the usual choice for rclone on a laptop or
non-AWS host.

Console (admin-managed flow):

1. **IAM** → **Users** → select the user → **Security credentials**
2. **Create access key**
3. Choose use case **Command Line Interface (CLI)** or **Local code** /
   **Other** (wording varies; all create the same key type)
4. Download the `.csv` / copy the secret **now** — the secret is shown
   **only once** (`CreateAccessKey` API behaviour)

CLI / SDK:

```bash
aws iam create-access-key --user-name rclone-cold-backup
```

Equivalent SDK call: `iam:CreateAccessKey` (e.g. AWS SDK for Go v2
`CreateAccessKey`, JS `@aws-sdk/client-iam` `CreateAccessKeyCommand`).

Map into rclone:

| AWS field | rclone config |
|-----------|---------------|
| `AccessKeyId` | `access_key_id` |
| `SecretAccessKey` | `secret_access_key` |

Limits / warnings from IAM:

- Max **two** access keys per user
- Do **not** create or use **root** account access keys
- Do not commit keys into source trees; store in rclone config or a secrets
  manager
- If the secret is lost, you cannot retrieve it — delete and create a new key

### 2.4 Security habits

- One IAM user per machine/purpose; never use root keys.
- Scope the policy to **one bucket** (identity policy above).
- Prefer an IAM **role** (instance profile / task role / IRSA) when rclone
  runs on EC2/ECS/EKS — no long-lived access key.
- Rotate on a schedule or immediately if leaked: create new key → update
  rclone → deactivate/delete the old key.
- Optionally monitor with CloudTrail (IAM recommendation after creating keys).

### 2.5 Quick connectivity test

After the remotes in §3 are configured (use the **bucket** path — this policy
does not include `s3:ListAllMyBuckets`):

```bash
rclone lsf mys3:YOUR-BUCKET
# Force STANDARD for the smoke test — avoids Deep Archive 180-day early-delete fees
rclone touch secret:ping.txt --s3-storage-class STANDARD
rclone ls secret:
rclone deletefile secret:ping.txt
```

Do **not** upload-and-quickly-delete with the default `DEEP_ARCHIVE` class:
AWS bills a **180-day minimum** if you delete/overwrite/transition early.

---

## 3. Configure remotes (crypt + Deep Archive)

### Underlying S3 remote

```ini
[mys3]
type = s3
provider = AWS
access_key_id = AKIA...          # from §2
secret_access_key = ...          # from §2
region = eu-west-1
location_constraint = eu-west-1
acl = private
storage_class = DEEP_ARCHIVE
no_check_bucket = true           # bucket already exists; matches least-priv IAM
```

Optional: add AWS SSE as a second layer (`server_side_encryption = AES256` or
`aws:kms`). This guide’s **default confidentiality** is still **local crypt**;
SSE alone does not encrypt filenames or protect against a compromised provider
the way client-side crypt does.

### Crypt remote (default path for all data)

```ini
[secret]
type = crypt
remote = mys3:YOUR-BUCKET
password = <obscured>
password2 = <obscured>
# Prefer encrypting filenames (rclone config default / “standard”)
```

Create passwords with `rclone config` (or `rclone obscure`). Keep the crypt
config and passwords safe — without them the ciphertext is unusable.

### Day-to-day usage

```bash
# Always read/write through crypt
rclone sync /backups secret:archives -P
rclone copy ./data secret:archives/2024/ -P
rclone lsf secret:archives/ --format "pe"    # plaintext + encrypted names
```

Do **not** upload plaintext through `mys3:` if you intend everything to be
locally encrypted.

### Inspect storage class

```bash
rclone lsf --format "pT" -R secret:archives
# or on the raw S3 keys:
rclone lsf --format "pT" -R mys3:YOUR-BUCKET/
```

### Change class of existing objects

```bash
# Prefer operating via crypt (names stay clear to you)
rclone settier DEEP_ARCHIVE secret:archives
rclone settier STANDARD secret:archives/2024/db.dump.gz
```

`settier` uses S3 `CopyObject` with a new `StorageClass`. For Deep Archive /
Glacier Flexible Retrieval, AWS **requires** a completed restore before the
object can be used as a copy source (`CopyObject` docs). Unrestored
`settier` will fail at AWS even though rclone issues the copy.

---

## 4. Restore workflows (Bulk by default)

Restore always runs against the **S3** remote (`mys3:`), not `secret:`.
`crypt` does not implement `backend restore`.

### Restore a directory (default: Bulk)

```bash
# 1) Map clear path → encrypted S3 path (see §6)
rclone lsf secret:archives/2024/ --format "pe"

# 2) Preview
rclone --interactive backend restore mys3:YOUR-BUCKET/ENC_DIR/ \
  -o priority=Bulk -o lifetime=7

# 3) Run (both -o flags required for this manual’s cost defaults)
rclone backend restore mys3:YOUR-BUCKET/ENC_DIR/ \
  -o priority=Bulk -o lifetime=7
```

Options:

- `priority` — pass **`Bulk`** explicitly for cost; `Standard` if you need ~12h.
  If omitted, AWS uses **Standard**.
- `lifetime` — **required** for Deep Archive / Glacier (`Days` ≥ 1). Ignored /
  cleared by rclone for `INTELLIGENT_TIERING`.
- `description` — optional job description

Only objects whose listed class is `GLACIER`, `DEEP_ARCHIVE`, or
`INTELLIGENT_TIERING` are restored by rclone (others are skipped with a status
message). `GLACIER_IR` is not restored — it is already readable.

### Check restore status

```bash
rclone backend restore-status mys3:YOUR-BUCKET/ENC_DIR/
rclone backend restore-status -o all mys3:YOUR-BUCKET/ENC_DIR/
```

rclone lists with `OptionalObjectAttributes=RestoreStatus` (ListObjectsV2).
AWS’s CLI equivalent for one key is `head-object` (shows `Restore:
ongoing-request=...`).

Useful fields:

- `IsRestoreInProgress` — `true` while AWS is still retrieving
- `RestoreExpiryDate` — when the temporary copy expires
- `StorageClass` — still `DEEP_ARCHIVE` after restore

When `IsRestoreInProgress` is `false` and an expiry date is set, read via
**crypt**:

```bash
rclone copy secret:archives/2024/ ./local/restored/
```

### After restore: temporary vs permanent

**Temporary access (default):** you only needed `-o lifetime=…` on restore.
AWS already created the readable copy; download before `RestoreExpiryDate`.
When that date hits, AWS removes the temp copy automatically — no rclone
cleanup step. The object is archived again until another restore.

```bash
# Choose lifetime to cover download + buffer (e.g. Bulk wait + copy day)
rclone backend restore mys3:YOUR-BUCKET/ENC_DIR/ \
  --include "ENC_NAME" -o priority=Bulk -o lifetime=7

# After IsRestoreInProgress=false — decrypt via crypt
rclone copy secret:archives/2024/db.dump.gz ./restored/
# Optional: extend the window — re-issue restore with a new lifetime.
# AWS updates expiry from *now*. If already restored, AWS bills that
# follow-up as a GET-style request (not a full new retrieval).
```

**Permanent hot object (manual):** restore does **not** do this. After the
temporary copy is ready:

```bash
rclone settier STANDARD secret:archives/2024/db.dump.gz
```

That is a `CopyObject` reclass. Do it only if you want ongoing Standard storage
(and remember Deep Archive’s **180-day minimum** if you leave Deep Archive
early).

---

## 5. Single object only

`rclone backend restore` operates on a **directory**. A bare file path fails:

```text
Failed to backend: is a file not a directory
```

### Restore one object (crypt + Bulk)

```bash
# Clear name you care about
FILE=archives/2024/db.dump.gz

# Encrypted S3 key / basename (from crypt)
rclone cryptdecode --reverse secret: "$FILE"
# or: rclone backend encode secret: "$FILE"
# or: rclone lsf secret:archives/2024/ --format "pe"

# Restore: parent encrypted dir + --include encrypted basename
rclone --interactive backend restore mys3:YOUR-BUCKET/ENC_PARENT/ \
  --include "ENC_BASENAME" \
  -o priority=Bulk -o lifetime=7

rclone backend restore mys3:YOUR-BUCKET/ENC_PARENT/ \
  --include "ENC_BASENAME" \
  -o priority=Bulk -o lifetime=7

# Wait (~48h for Bulk on Deep Archive)
rclone backend restore-status mys3:YOUR-BUCKET/ENC_PARENT/

# Download decrypted
rclone copy secret:archives/2024/db.dump.gz ./restored/
```

### Filters and encrypted names

`backend restore` runs on **`mys3:`**, so `--include` / `--exclude` match
**S3 object keys** (ciphertext names when `filename_encryption` is
`standard` / `obfuscate`).

```bash
# WRONG with filename encryption on — "*.sql.gz" will not match encrypted keys
rclone backend restore mys3:YOUR-BUCKET/ENC_DIR/ --include "*.sql.gz" ...

# RIGHT — map first, then include the encrypted basename (or restore whole dir)
rclone lsf secret:archives/2024/ --format "pe"
rclone backend restore mys3:YOUR-BUCKET/ENC_PARENT/ \
  --include "ENC_BASENAME" -o priority=Bulk -o lifetime=3
```

Only use plaintext globs on `mys3:` if crypt `filename_encryption = off`.

---

## 6. Encryption details

### Why local crypt is the default

| Layer | Role in this guide |
|-------|--------------------|
| **rclone `crypt`** | **Default** — encrypt content (+ names) before upload |
| HTTPS | Always — in transit |
| S3 SSE | Optional extra; not a substitute for crypt |

With crypt, S3 (and anyone with bucket access) only sees ciphertext and
encrypted names. Glacier/Deep Archive still apply to those encrypted objects
unchanged.

### Name mapping helpers

```bash
rclone cryptdecode --reverse secret: path/to/file.bin   # clear → encrypted
rclone backend encode secret: path/to/file.bin
rclone backend decode secret: ENC_NAME                  # encrypted → clear
rclone lsf secret:path/ --format "pe"                   # both columns
```

### Crypt + restore checklist

| Do | Don't |
|----|-------|
| Upload/download via `secret:` | Upload plaintext via `mys3:` |
| Restore via `mys3:` + encrypted names | Expect `backend restore` on `secret:` |
| Filter restore with **encrypted** `--include` | `--include "*.txt"` on `mys3:` with name encryption |
| Default restore to `priority=Bulk` | Use `Expedited` on Deep Archive |
| Keep crypt passwords/salt backed up | Lose the crypt config |

### Optional: S3 SSE on top

```ini
[mys3]
# ...
storage_class = DEEP_ARCHIVE
server_side_encryption = AES256
# or: server_side_encryption = aws:kms
#     sse_kms_key_id = arn:aws:kms:...
```

With KMS you **must** set `server_side_encryption = aws:kms`, or small uploads
can fail checksum checks. SSE does not change restore syntax.

---

## 7. CloudFront downloads and egress cost

CloudFront does **not** replace Glacier restore. It can only cheapen **egress
after** an object is readable (restored Deep Archive, or always-hot classes).

### rclone support

Set `download_url` on the S3 remote. Listing/upload/restore still use the S3
API; **downloads** use a plain HTTP GET to the CDN URL:

```ini
[mys3]
type = s3
provider = AWS
# ...
storage_class = DEEP_ARCHIVE
download_url = https://d111111abcdef8.cloudfront.net/
```

```bash
# Still restore on S3 first for Deep Archive
rclone backend restore mys3:YOUR-BUCKET/ENC_DIR/ \
  --include "ENC_NAME" -o priority=Bulk -o lifetime=7

# After restore: copy via crypt (Open uses download_url under the hood)
rclone copy secret:archives/2024/db.dump.gz ./restored/
```

Caveats:

- rclone does **not** sign CloudFront URLs; the distribution must allow that GET
  (public/OAC patterns that accept unsigned viewer requests).
- `download_url` is concatenated with the **object key** only (no bucket name).
  Point CloudFront’s origin at that bucket (and Origin Path, if any, must match).
- Unrestored Deep Archive still fails through CloudFront (`InvalidObjectState`).
- With `crypt`, CloudFront carries **ciphertext**; decrypt happens locally.

### How much cheaper is egress?

Typical **US/EU pay-as-you-go** (verify on current AWS price pages):

| Path | Free / month | Then (approx.) |
|------|----------------|----------------|
| S3 → internet (normal rclone) | **100 GB** (shared AWS DTO) | **~$0.09/GB** first 10 TB |
| S3 → CloudFront | always **$0** | — |
| CloudFront → internet | **1 TB** + 10M requests | **~$0.085/GB** first 10 TB |

Beyond free tier, list price is only ~**5–6%** lower. The big win for rare
restores is CloudFront’s **1 TB free** vs **100 GB** direct.

| Download | Direct S3 egress | Via CloudFront | Rough save |
|----------|------------------|----------------|------------|
| 500 GB | ~$36 | **$0** (under 1 TB free) | ~$36 |
| 1 TB | ~$83 | **$0** | ~$83 |
| 2 TB | ~$175 | ~$87 | ~50% |

### What CloudFront does *not* reduce

| Charge | Still owed? |
|--------|-------------|
| Deep Archive Bulk/Standard **retrieval** | Yes |
| **Temporary Standard storage** for `lifetime` days | Yes (automatic while restored) |
| Deep Archive **storage** / 180-day minimum | Yes |
| S3 GET on CDN cache miss | Small request fee |

Sources: [S3 pricing](https://aws.amazon.com/s3/pricing/),
[CloudFront pricing](https://aws.amazon.com/cloudfront/pricing/),
[AWS DTO free-tier expansion](https://aws.amazon.com/blogs/aws/aws-free-tier-data-transfer-expansion-100-gb-from-regions-and-1-tb-from-amazon-cloudfront-per-month/).

---

## 8. Delete objects

Prefer **`secret:`** so you use clear names. Deleting does **not** require a
prior restore — S3 can delete archived objects directly (`s3:DeleteObject` is
enough; already in the §2 policy).

### One file

```bash
rclone deletefile secret:archives/2024/db.dump.gz --dry-run
rclone deletefile secret:archives/2024/db.dump.gz
```

### Many files / a path

```bash
# Delete matching files under a path (not the directory marker itself)
rclone delete secret:archives/2024/ --include "*.sql.gz" --dry-run
rclone delete secret:archives/2024/ --include "*.sql.gz"

# Remove a whole tree
rclone purge secret:archives/old/ --dry-run
rclone purge secret:archives/old/
```

With **filename encryption**, filters on `secret:` use **plaintext** names
(rclone encrypts them for you). Filters on `mys3:` must use encrypted keys.

### Deep Archive cost warning

| Situation | Effect |
|-----------|--------|
| Object in `DEEP_ARCHIVE` for under 180 days | AWS charges **pro-rated remainder** of the 180-day minimum |
| Object older than 180 days | No early-delete surcharge for the minimum duration |
| Object currently restored | Delete removes the object (archive + any temp readable copy) |
| Bucket **versioning** on | `deletefile` may only add a delete marker; purging versions needs extra steps/IAM |

Always `--dry-run` first on bulk deletes.

### Raw S3 (usually avoid)

```bash
rclone deletefile mys3:YOUR-BUCKET/ENC_KEY
```

Use only if you already mapped the encrypted key.

---

## 9. End-to-end recipes (defaults applied)

### A. Encrypted Deep Archive backup

```bash
rclone sync /backups secret:archives -P
# Objects land in DEEP_ARCHIVE via mys3 storage_class
```

### B. Retrieve one file later (Bulk, ~48h)

```bash
rclone lsf secret:archives/2024/ --format "pe"

rclone backend restore mys3:YOUR-BUCKET/ENC_DIR/ \
  --include "ENC_NAME" \
  -o priority=Bulk -o lifetime=3

# Poll over the next ~2 days
rclone backend restore-status mys3:YOUR-BUCKET/ENC_DIR/

rclone copy secret:archives/2024/db.dump.gz ./restored/
```

### C. Faster (costlier) single-file restore

```bash
rclone backend restore mys3:YOUR-BUCKET/ENC_DIR/ \
  --include "ENC_NAME" \
  -o priority=Standard -o lifetime=3
# ~12 hours on Deep Archive
```

### D. Leave Deep Archive permanently for one object

```bash
# After Bulk/Standard restore completed
rclone settier STANDARD secret:archives/2024/db.dump.gz
```

### E. Delete one archived object

```bash
rclone deletefile secret:archives/2024/obsolete.dump.gz --dry-run
rclone deletefile secret:archives/2024/obsolete.dump.gz
# If younger than 180 days in DEEP_ARCHIVE, expect early-delete charges
```

---

## 10. Limitations and gotchas

1. **Single-file restore** needs `dir/ --include name`, not a file remote path.
2. **Encrypted `--include`**: with filename encryption, filters on `mys3:` must
   use encrypted keys (or restore a whole directory). Plain `*.sql.gz` will miss.
3. **Omit `priority` → AWS Standard**, not Bulk. Always pass `-o priority=Bulk`
   for this manual’s cost default.
4. **Omit `lifetime` → AWS rejects** Deep Archive / Glacier archive restores
   (`Days` required). Always pass `-o lifetime=DAYS` (≥ 1).
5. **Deep Archive + Bulk** is cheapest Deep Archive retrieval and slowest
   (~48h). Bulk is **not free** on Deep Archive (unlike Flexible Retrieval Bulk).
6. **Deep Archive cannot use `Expedited`.**
7. Restore is **temporary** and **automatic**: AWS creates/deletes the
   Standard-rate temp copy for your `lifetime` days (expiry rounded to next
   UTC midnight). Class stays Deep Archive unless you **manually** `settier` /
   copy after restore. While restored you pay archive + temp Standard storage.
8. **180-day minimum** storage duration for Deep Archive: delete / overwrite /
   transition early → pro-rated remainder charge. (`settier STANDARD` early
   counts as a transition.)
9. **40 KB metadata overhead** per archived object (32 KB at Deep Archive rate +
   8 KB at Standard). Prefer fewer large archives over many tiny crypt objects.
10. **`crypt` has no `restore` command** — use `mys3:` with encrypted keys.
11. Modtime updates via server-side copy do not work on unrestored Deep Archive.
12. Lifecycle policies and Glacier **Vaults** are outside rclone (S3 API only).
13. rclone may attempt restore on **all** `INTELLIGENT_TIERING` objects; AWS
    only needs/accepts restore when they are in Archive / Deep Archive Access.
14. Test restores with `--interactive` first; watch AWS retrieval pricing.
15. Losing crypt passwords/salt means permanent loss of the ciphertext.
16. CloudFront `download_url` needs a distribution that accepts rclone’s
    unsigned GET; it does not bypass restore or crypt.

---

## 11. Quick command reference

```bash
# Backup (crypt → Deep Archive)
rclone sync SRC secret:PATH -P

# Change class
rclone settier DEEP_ARCHIVE|STANDARD secret:PATH

# Name mapping
rclone lsf secret:PATH --format "pe"
rclone cryptdecode --reverse secret: clear/path/file

# Restore directory (cost default — both -o flags required)
rclone backend restore mys3:YOUR-BUCKET/ENC_PATH/ -o priority=Bulk -o lifetime=DAYS

# Restore one object (cost default)
rclone backend restore mys3:YOUR-BUCKET/ENC_PATH/ --include "ENC_FILE" \
  -o priority=Bulk -o lifetime=DAYS

# Status / download
rclone backend restore-status mys3:YOUR-BUCKET/ENC_PATH/
rclone copy secret:clear/path/file ./local/

# Delete (prefer secret:; watch 180-day early-delete fees)
rclone deletefile secret:path/file --dry-run
rclone deletefile secret:path/file
rclone delete secret:path/ --include "pattern"
rclone purge secret:old-tree/
```

---

## 12. Decision guide

```text
Default stack
  └─ secret: (local crypt) → mys3: with DEEP_ARCHIVE → restore with Bulk

Need to read an archived object?
  └─ Map name → restore on mys3: with Bulk → wait ~48h → copy via secret:

Need only one object?
  └─ Parent encrypted dir + --include encrypted basename

Need it in ~12h instead of ~48h?
  └─ Same flow with -o priority=Standard (costs more)

Want it permanently hot?
  └─ Restore, then settier STANDARD on secret:path

Need cheaper egress on download?
  └─ After restore, use S3 download_url = CloudFront (see §7)

Need to delete an object?
  └─ rclone deletefile secret:path (no restore needed; 180-day fee if early)

Need lifecycle automation?
  └─ Configure in AWS; rclone sets class on upload / settier only
```
