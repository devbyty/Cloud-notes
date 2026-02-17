# aws-cli-patterns.md
Tyler’s AWS CLI Patterns (CloudShell-friendly)

## 0) The only “syntax” you need to remember
AWS CLI is basically:

aws <service> <command> [--flag value]...

Examples:
- aws s3api list-buckets
- aws sts get-caller-identity

If you forget a command:
- aws help
- aws <service> help
- aws <service> <command> help

---

## 1) Placeholders (IMPORTANT)
Never type `<random>` literally.

Bad (bash treats `<random>` like file redirection):
- tyler-cloudops-lab-<random>

Good:
- Use a variable:
  BUCKET="tyler-cloudops-lab-$RANDOM-$RANDOM"

---

## 2) “Am I logged in?” (Auth sanity check)
Run this first when anything feels broken:

aws sts get-caller-identity

In CloudShell, you usually do NOT run `aws configure`.
(CloudShell uses temporary credentials automatically.)

If you ever did run `aws configure` and broke things:
- rm -f ~/.aws/credentials ~/.aws/config

---

## 3) S3 basics (bucket + object) — the 4 commands you’ll reuse forever
### Create a unique bucket name (bash)
BUCKET="tyler-cloudops-lab-$RANDOM-$RANDOM"
echo "$BUCKET"

### Create bucket (us-east-1)
aws s3api create-bucket --bucket "$BUCKET" --region us-east-1

### Upload a file
echo "hello from cloudops lab" > hello.txt
aws s3 cp hello.txt "s3://$BUCKET/hello.txt"

### List objects
aws s3api list-objects-v2 --bucket "$BUCKET"

---

## 4) “Show me less JSON” (filters with --query)
You do NOT need to memorize JMESPath—just steal patterns.

### List object keys only
aws s3api list-objects-v2 --bucket "$BUCKET" \
  --query "Contents[].Key" --output text

### Key + Size as a table
aws s3api list-objects-v2 --bucket "$BUCKET" \
  --query "Contents[].{Key:Key,Size:Size}" --output table

---

## 5) Read object metadata
aws s3api head-object --bucket "$BUCKET" --key "hello.txt"

Just the ETag:
aws s3api head-object --bucket "$BUCKET" --key "hello.txt" \
  --query "ETag" --output text

---

## 6) “Folders” in S3 are just prefixes
Upload to a “folder”:
aws s3 cp hello.txt "s3://$BUCKET/lab/hello.txt"

List only that prefix:
aws s3api list-objects-v2 --bucket "$BUCKET" --prefix "lab/" --output table

---

## 7) Versioning mini-cheat sheet
### Turn on versioning
aws s3api put-bucket-versioning --bucket "$BUCKET" \
  --versioning-configuration Status=Enabled

### List versions of a key
aws s3api list-object-versions --bucket "$BUCKET" --prefix "hello.txt" --output table

### Download a specific version (replace VERSION_ID)
aws s3api get-object --bucket "$BUCKET" --key "hello.txt" \
  --version-id VERSION_ID old.txt

### Delete a specific version (replace VERSION_ID)
aws s3api delete-object --bucket "$BUCKET" --key "hello.txt" \
  --version-id VERSION_ID

---

## 8) “It says END” (pager)
If AWS output opens a scroll view and shows “END”:
- Press `q` to quit.

To disable pager:
export AWS_PAGER=""

---

## 9) Cleanup patterns (so you don’t get stuck)
### Simple cleanup (works when versioning is OFF)
aws s3 rm "s3://$BUCKET" --recursive
aws s3api delete-bucket --bucket "$BUCKET"

### When versioning is ON (you must delete versions)
(Quick nuke pattern)
aws s3api list-object-versions --bucket "$BUCKET" \
  --query '{Objects: Versions[].{Key:Key,VersionId:VersionId}}' \
  --output json > delete.json

aws s3api delete-objects --bucket "$BUCKET" --delete file://delete.json
aws s3api delete-bucket --bucket "$BUCKET"

---

## 10) The “mental model” (this is what you’re actually learning)
- AWS services expose APIs
- CLI calls those APIs
- Output is JSON
- --query filters JSON
- You don’t memorize commands; you memorize:
  1) the service name (s3api, ec2, iam, lambda)
  2) the action name (list, create, delete)
  3) where to look for syntax (help)

---

## 11) Your default lab flow (repeatable)
1) aws sts get-caller-identity
2) Create unique resource name with $RANDOM
3) Create resource
4) Do one action (upload/list/describe)
5) Filter output with --query
6) Cleanup

---
### Pro Habit

Once a week:

aws ec2 describe-instances --query "Reservations[].Instances[].State.Name"
aws ec2 describe-volumes --query "Volumes[].State"
aws ec2 describe-addresses


30-second audit.

### When closing instances, checks
Final Safety Checks (2 quick ones)

Just to be disciplined:

1. Check for orphaned volumes
aws ec2 describe-volumes \
  --query "Volumes[].{ID:VolumeId,State:State,Size:Size}" \
  --output table


If it returns nothing, you’re fully clean.
If anything shows available, delete it.

2. Check for Elastic IPs
aws ec2 describe-addresses \
  --query "Addresses[].PublicIp" \
  --output table


If it’s empty → good.
If not, release them.
---
### Why it feels discouraging

Because EC2 CLI looks like this:

aws ec2 run-instances \
  --image-id ami-xxxx \
  --instance-type t3.micro \
  --key-name tyler-ec2-key \
  --security-groups tyler-ec2-sg \
  --count 1


And your brain goes:

“There’s no way I’ll ever remember this.”

You’re not supposed to.

No one memorizes this.

### What real engineers actually know

They know:

- EC2 needs an AMI

- EC2 needs an instance type

- EC2 needs a security group

- EC2 needs a key or IAM role

- Everything has an ID

- describe-* shows you stuff

- run-* creates stuff

- terminate-* deletes stuff

That’s it.

Syntax is lookup.
---
### AWS Lambda – Cold Start vs Warm Start (Study Guide)

-What is AWS Lambda?

AWS Lambda is serverless compute.

You provide:

- Code (a function)

- A trigger (event)

- Permissions (IAM role)

AWS handles:

- Servers

- Scaling

- Patching

- Runtime lifecycle

- Availability

### Lambda Execution Lifecycle

When a Lambda function runs, AWS uses an execution environment (container).

There are two states you observed:

### Cold Start

Occurs when AWS must:

1. Create a new container

2. Load the runtime (Python, Node, etc.)

3. Initialize your function code

In logs, this appears as:

yaml: Init Duration: 138 ms


Cold start = extra startup latency.

When Cold Starts Happen:

- First invocation after deployment

- After a period of inactivity

- When scaling up to handle new concurrent requests

### Warm Start

Occurs when AWS reuses an existing execution environment.

No container creation.
No runtime load.
Just runs your code.

In logs, this looks like:

yaml: Duration: 1.65 ms
Billed Duration: 2 ms


No Init Duration shown.

Warm start = fast execution.

### Lambda Billing Model

Lambda charges for:

(Execution Time + Init Time if cold) × Memory Allocation


From your logs:

Cold start:

Duration: 2 ms
Init Duration: 138 ms
Billed Duration: 141 ms


Warm start:

Duration: 1.65 ms
Billed Duration: 2 ms


You are billed for:

- Total runtime

- Memory configured (e.g., 128MB)

Important Concepts
1. Lambda is Stateless

- Each invocation should not rely on previous state

- Containers may disappear at any time

2. Memory Controls CPU

- Increasing memory increases available CPU

- More memory = faster execution (usually)

3. Scaling

Lambda scales by:

- Creating additional containers

- One container per concurrent execution

Example:
10 simultaneous requests = up to 10 containers
---
### What AWS SAM Actually Is

SAM = Serverless Application Model

Think of it as:

“Infrastructure as Code for Lambda apps.”

Instead of clicking:

- Create Lambda

- Create API Gateway

- Create IAM role

- Wire permissions

You define it in one YAML file.

Then deploy with one command.