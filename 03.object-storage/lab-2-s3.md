---
duration: 1h
category:
  - name: LAB
components:
  - name: PYTHON
  - name: UV
platforms:
  - name: LINUX
resources:
  - title: uv (official homepage)
    url: https://docs.astral.sh/uv/
  - title: Installing uv (official documentation)
    url: https://docs.astral.sh/uv/getting-started/installation/
  - title: Faker (official homepage)
    url: https://pypi.org/project/Faker/
  - title: argparse (official homepage)
    url: https://docs.python.org/3/library/argparse.html
revisions:
  - date: 2026-09-13
    comment: Initial page
    author: david@adaltas.com
tags:
  - name: TUTORIAL
---

# Lab: project creation with uv and data generation

## Objectives

- Familiarize the usage and management of Ceph S3 bucket.
- Create Ceph S3 bucket.
- Put files into the bucket.

## CLI utilities

`aws s3` is the official CLI tool recommanded by AWS.

`s5cmd` is a faster drop-in alternative to the AWS CLI for S3 operations. Key differences:

- **Speed**  
  Runs operations in parallel by default, significantly faster for bulk upload, downloads, and deletes.
- **Syntax**  
  Slightly shorter, no `s3` subcommand (eg `aws s3 cp ` is replaced by `s5cmd cp`).
- **Batch operations**
  Accepts a list of commands from a file, enabling massive parallel execution (eg `s5cmd run commands.txt`)
- **Scope**
  Covers only file operations (`cp`, `mv`, `rm`, `ls`, `sync`). For API-level operations (`put-bucket-policy`, `head-object`, `list-object-versions`…) you still need `aws s3api`.
- **No configuration management**
  No `aws configure`, no profiles.

Use `s5cmd` for data transfer at scale, `aws s3api` for everything else.

# s5cmd utility installation

User binaries are commonly installed in `~/.local/bin`. The folder is added to the `$PATH` environmental variable.

```bash
CMD='export PATH="$HOME/.local/bin:$PATH"'
! (cat "$HOME/.bashrc" | grep -xF "$CMD") \
  && echo "$CMD" >> "$HOME/.bashrc"
source "$HOME/.bashrc"
```

[s5cmd](https://github.com/peak/s5cmd) is a command line utility designed to manipulate files on S3-compatible object storage services. It is used to upload and download files from the object store in this lab. The s5cmd utility is downloaded and installed on the client node and the host machine.

```bash
S5CMD_VERSION=$(
  curl \
    -s https://api.github.com/repos/peak/s5cmd/releases/latest \
  | jq -r '.tag_name | .[1:]'
)
BIN_DIR=$([[ "$USER" == "root" ]] && echo /usr/local/bin || echo ~/.local/bin)
mkdir -p "$BIN_DIR"
# Architecture discovery
ARCH_CMD=$(uname -m)
if [[ "$ARCH_CMD" == "x86_64" ]]; then
  S5CMD_ARCH="64bit"
elif [[ "$ARCH_CMD" == "aarch64" ]] || [[ "$ARCH_CMD" == "arm64" ]]; then
  S5CMD_ARCH="arm64"
else
  echo "System architecture ${ARCH_CMD} not supported."
  exit 1
fi
# Binary download
S5CMD_FILE="s5cmd_${S5CMD_VERSION}_Linux-${S5CMD_ARCH}.tar.gz"
S5CMD_URL="https://github.com/peak/s5cmd/releases/download/v${S5CMD_VERSION}/${S5CMD_FILE}"
curl -fsSL -o "/tmp/${S5CMD_FILE}" "$S5CMD_URL"
# Signature validation
CHECKSUM_URL="https://github.com/peak/s5cmd/releases/download/v${S5CMD_VERSION}/s5cmd_checksums.txt"
CHECKSUM_FILE="s5cmd_checksums.txt"
curl -fsSL -o "/tmp/${CHECKSUM_FILE}" "$CHECKSUM_URL"
cat /tmp/${CHECKSUM_FILE} | sed "s|${S5CMD_FILE}|/tmp/${S5CMD_FILE}|g" > /tmp/checksum_custom.txt
# Installation
grep "${S5CMD_FILE}" /tmp/checksum_custom.txt | sha256sum -c \
&& tar -xf /tmp/${S5CMD_FILE} -C $BIN_DIR
chmod +x "$BIN_DIR/s5cmd"
# Cleanup
rm -rf /tmp/${S5CMD_FILE}
rm -rf /tmp/${CHECKSUM_FILE}
rm -rf /tmp/checksum_custom.txt
```

The `s5cmd` command is now available.

```bash
command -v s5cmd
#> /home/onyxia/.local/bin/s5cmd
s5cmd version
#> v2.3.0-991c9fb
```

## Bucket name

In Onyxia, the bucket name is `user-<username>` and equals your namespace name.

```bash
export LAB_BUCKET_NAME="$KUBERNETES_NAMESPACE"
```

## Upload (PUT)

Using the script previously developed, the user dataset is generated in the CSV format and uploaded to the bucket. Under the hood this is a single HTTP `PUT` request. The object is written atomically. No other client sees a partial state during the upload.

```bash
uv run src/dataset_users.py -o csv > users.csv
aws s3 --profile 'default' cp users.csv "s3://$LAB_BUCKET_NAME/data/users.csv"
# s5cmd cp users.csv "s3://$LAB_BUCKET_NAME/data/users.csv"
```

## List objects (GET)

Listing returns all keys that start with a given prefix. It looks like browsing a directory, but the bucket contains actually a flat list of keys.

```bash
aws s3 --profile 'default' ls "s3://$LAB_BUCKET_NAME/data/"
# s5cmd ls "s3://$LAB_BUCKET_NAME/data"
# Both commands produce the same output
#> 2026-08-14 00:38:01       5431 users.csv
```

The `--recursive` argument lists every object in the bucket regardless of their prefix.

```bash
aws s3 --profile 'default' ls "s3://$LAB_BUCKET_NAME/" --recursive
```

## Download (GET)

`GET` retrieves an object by its exact key:

```bash
aws s3 --profile 'default' cp "s3://$LAB_BUCKET_NAME/data/users.csv" ./users_downloaded.csv
```

## Moving and renaming

There is no atomic rename. It is achieved with two separate operations, copying the source file to its destination and removing the source file. Pipelines must be defined to tolerate seeing both keys briefly.

```bash
aws s3 --profile 'default' cp "s3://$LAB_BUCKET_NAME/data/users.csv"  "s3://$LAB_BUCKET_NAME/data/users.csv"
```

### Delete

`DELETE` removes a single object by key:

```bash
aws s3 --profile 'default' rm "s3://$LAB_BUCKET_NAME/data/users.csv"
```

Deleting everything under a prefix requires deleting each object individually. The `--recursive` flag loops over all matching keys:

```bash
# See teardown instructions, here only for illustration purpose
# s3 rm s3://$LAB_BUCKET_NAME/data/ --recursive
```

## Metadata

Every object carries two kinds of metadata.

System metadata is set automatically: size, content type, last modified date, and an ETag which is the MD5 of the content. It is used to verify integrity or detect changes.

```bash
s3api head-object --bucket "$LAB_BUCKET_NAME" --key data/users.csv
```

```json
{
  "ContentLength": 4218,
  "ContentType": "text/csv",
  "ETag": "\"d41d8cd98f00b204e9800998ecf8427e\"",
  "LastModified": "2026-08-14T10:23:00Z"
}
```

User-defined metadata lets you attach arbitrary key-value pairs to any object at upload time:

```bash
aws s3 --profile 'default' cp users.csv "s3://$LAB_BUCKET_NAME/data/users.csv" \
  --metadata "source=dataset_users.py,version=1,rows=50"

s3api head-object --bucket "$LAB_BUCKET_NAME" --key data/users.csv \
  | jq '.Metadata'
# { "source": "dataset_users.py", "version": "1", "rows": "50" }
```

Because objects are immutable, metadata is immutable too. To update it you must re-upload the object (or copy it to the same key with new metadata).

Metadata is free and queryable via `HeadObject` — use it to track lineage, schema version, or processing status instead of maintaining a separate index.

## Presigned URLs

A presigned URL lets anyone download an object without needing credentials. The URL embeds the authentication signature and an expiry time:

```bash
aws --endpoint-url $ENDPOINT_URL s3 presign \
  s3://$LAB_BUCKET_NAME/data/users.csv \
  --expires-in 3600
#>  http://ceph-rgw:80/users-abc123/data/users.csv?X-Amz-Algorithm=...&X-Amz-Expires=3600&X-Amz-Signature=...
```

Any HTTP client can use it directly, with no configuration:

```bash
curl "<presigned-url>" -o users.csv
```

After the expiry time, the URL returns a `403 Forbidden`. You can also generate presigned `PUT` URLs to let external clients upload without exposing your credentials.

Presigned URLs can also be `PUT` URLs, useful for letting external clients upload without exposing credentials and without routing data through a client's complementary backend.

## Multipart Upload

For large files, the S3 protocol splits the upload into numbered parts sent in parallel and assembled by the server. The AWS CLI does this automatically above a configurable threshold:

```bash
dd if=/dev/urandom of=large_dataset.bin bs=1M count=200

aws s3 --profile 'default' cp large_dataset.bin s3://$LAB_BUCKET_NAME/large/dataset.bin
```

The benefits are:

- **Resilience**: if a part fails, only that part is retried.
- **Speed**: parts are uploaded in parallel, saturating available bandwidth.
- **No size limit**: a single `PUT` is capped at 5 GB; multipart handles up to 5 TB.

Incomplete multipart uploads are invisible to normal listing but consume storage. Always add a lifecycle rule to abort them automatically:

```bash
s3api list-multipart-uploads --bucket $LAB_BUCKET_NAME
```

## Versioning

When versioning is enabled, uploading to an existing key creates a new version instead of overwriting the previous one. All versions are retained and individually addressable:

```bash
s3api put-bucket-versioning \
  --bucket $LAB_BUCKET_NAME \
  --versioning-configuration Status=Enabled

aws s3 --profile 'default' cp users.csv s3://$LAB_BUCKET_NAME/data/users.csv        # version 1

sed 's/shawn95/shawn96/' users.csv | \
  aws s3 --profile 'default' cp - s3://$LAB_BUCKET_NAME/data/users.csv              # version 2

s3api list-object-versions --bucket $LAB_BUCKET_NAME --prefix data/users.csv
```

```json
{
  "Versions": [
    {
      "VersionId": "abc123",
      "IsLatest": true,
      "LastModified": "2026-08-14T10:30:00Z"
    },
    {
      "VersionId": "def456",
      "IsLatest": false,
      "LastModified": "2026-08-14T10:23:00Z"
    }
  ]
}
```

Retrieve any specific version by its ID:

```bash
s3api get-object \
  --bucket $LAB_BUCKET_NAME \
  --key data/users.csv \
  --version-id def456 \
  users_v1.csv
```

Deleting a versioned object does not remove any data — it creates a **delete marker**. The object disappears from normal `GET` requests but all versions remain recoverable:

```bash
aws s3 --profile 'default' rm s3://$LAB_BUCKET_NAME/data/users.csv
# The object seems gone...
s3api list-object-versions --bucket $LAB_BUCKET_NAME --prefix data/users.csv
# ...but all versions are still there
```

## Lifecycle Policies

Lifecycle rules let the storage backend automatically expire objects or abort stale uploads, without any external cron job or script.

The following policy deletes everything under `data/` after 30 days, and aborts any multipart upload that has not been completed within 7 days:

```bash
cat > lifecycle.json <<'EOF'
{
  "Rules": [
    {
      "ID": "expire-raw-after-30-days",
      "Status": "Enabled",
      "Filter": { "Prefix": "data/" },
      "Expiration": { "Days": 30 }
    },
    {
      "ID": "abort-incomplete-multipart",
      "Status": "Enabled",
      "Filter": { "Prefix": "" },
      "AbortIncompleteMultipartUpload": { "DaysAfterInitiation": 7 }
    }
  ]
}
EOF

s3api put-bucket-lifecycle-configuration \
  --bucket $LAB_BUCKET_NAME \
  --lifecycle-configuration file://lifecycle.json
```

Versioning and lifecycle rules are complementary. Versioning is enabled for recovery, lifecycle rules are enabled to control the cost of retaining old versions.

## Access Control

Access control has two levels## Access Control

Access control has two levels.

**Bucket policies** apply to all objects matching a prefix. They define who can do what on which resources:

```bash
cat > policy.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": { "AWS": "arn:aws:iam::$LAB_BUCKET_NAME:user/analyst" },
      "Action": ["s3:GetObject"],
      "Resource": "arn:aws:s3:::$LAB_BUCKET_NAME/data/*"
    }
  ]
}
EOF

s3api put-bucket-policy --bucket $LAB_BUCKET_NAME --policy file://policy.json
```

**Object ACLs** apply to a single object. The most common use is making an object publicly readable without generating a presigned URL:

```bash
s3api put-object-acl \
  --bucket $LAB_BUCKET_NAME \
  --key data/users.csv \
  --acl public-read

# Anyone can now fetch it with no credentials
curl "http://$BUCKET_HOST:$BUCKET_PORT/$LAB_BUCKET_NAME/data/users.csv"
```

## Consistency

Ceph RGW provides **strong read-after-write consistency**: as soon as a `PUT` returns successfully, any subsequent `GET` on that key returns the new object. There is no delay, no propagation window to wait for:

```bash
aws s3 --profile 'default' cp users.csv "s3://$LAB_BUCKET_NAME/data/users.csv"
aws s3 --profile 'default' cp "s3://$LAB_BUCKET_NAME/data/users.csv" /dev/null && echo "immediately available"
```

This is worth knowing because some object stores — especially multi-region cloud setups — historically provided only eventual consistency. Always verify this for your specific backend before building pipelines that depend on it.
