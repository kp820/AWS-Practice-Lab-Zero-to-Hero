# Secure Static Website Using Amazon S3, CloudFront, and OAC

This guide documents how to host a static website in a **private Amazon S3 bucket** and securely serve it through **Amazon CloudFront** using **Origin Access Control (OAC)**.

## Architecture

```text
User -> HTTPS -> Amazon CloudFront -> OAC-signed request -> Private S3 bucket
```

## Environment Used

| Resource | Value |
|---|---|
| S3 bucket | `aws-private-static-site-demo` |
| AWS Region | `ap-south-1` |
| AWS account ID | `112233445566` |
| CloudFront distribution ID | `E36FSU2A259TL2` |
| CloudFront domain | `dtkpyn8xaanla.cloudfront.net` |
| Default root object | `index.html` |

> The CloudFront domain may change if the distribution is recreated.

## Prerequisites

- An AWS account
- Permission to manage Amazon S3 and Amazon CloudFront
- A static website file named `index.html`

## Step 1: Create the private S3 bucket

1. Sign in to the AWS Management Console.
2. Open **Amazon S3**.
3. Select **Create bucket**.
4. Enter your own bucket name like: `aws-private-static-site-demo`.
5. Select your AWS Region: `Asia Pacific (Mumbai) ap-south-1`.
6. Keep **Object Ownership** set to **Bucket owner enforced**.
7. Keep **Block all public access** enabled.
8. Select **Create bucket**.

Do not make the bucket public.

## Step 2: Upload the website

1. Open the S3 bucket `aws-private-static-site-demo`.
2. Select **Upload**.
3. Upload `index.html` to the bucket root.
4. Confirm the final object key is exactly:

```text
index.html
```

Do not enable S3 static website hosting. CloudFront OAC uses the standard S3 REST endpoint, not the S3 website endpoint.

## Step 3: Create the CloudFront distribution

1. Open **Amazon CloudFront**.
2. Select **Create distribution**.
3. For **Origin domain**, select:

```text
aws-private-static-site-demo.s3.ap-south-1.amazonaws.com
```

4. Under **Origin access**, select **Origin access control settings (recommended)**.
5. Create a new Origin Access Control.
6. Configure the OAC as follows:
   - **Origin type:** S3
   - **Signing behavior:** Sign requests (recommended)
   - **Signing protocol:** SigV4/SigV4a
7. Under the default cache behavior, configure:
   - **Viewer protocol policy:** Redirect HTTP to HTTPS
   - **Allowed HTTP methods:** GET, HEAD
   - **Cache policy:** CachingOptimized
   - **Compress objects automatically:** Enabled
8. Set **Default root object** to:

```text
index.html
```

9. Create the distribution and wait until the status becomes **Enabled/Deployed**.

## Step 4: Configure the S3 bucket policy

Open:

**Amazon S3 → Bucket → Permissions → edit Bucket policy**

Add the following policy:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowCloudFrontOACReadAccess",
      "Effect": "Allow",
      "Principal": {
        "Service": "cloudfront.amazonaws.com"
      },
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::**YOUR-BUCKET-NAME**/*",
      "Condition": {
        "StringEquals": {
          "AWS:SourceArn": "arn:aws:cloudfront::**YOUR-ACCOUNT-ID**:distribution/**YOUR-DISTRIBUTION-ID**"
        }
      }
    }
  ]
}
```

This policy allows only CloudFront distribution `**Your distribution id**` to read the website objects.

## Step 5: Verify the OAC attachment

1. Open **CloudFront**.
2. Select distribution `**Your distribution id**`.
3. Open the **Origins** tab.
4. Select the S3 origin and choose **Edit**.
5. Confirm:
   - The origin is the standard S3 REST endpoint.
   - **Origin access control settings** is selected.
   - The correct OAC is attached.
   - Signing requests is enabled.

## Step 6: Test the website

Open the CloudFront URL:

```text
https://**Your distribution**.cloudfront.net/
like: https://dtkpyn8xccnla.cloudfront.net/
```

You can also test the file directly through CloudFront:

```text
https://**Your distribution**.cloudfront.net/index.html
like: https://dtkpyn8xccnla.cloudfront.net/index.html
```

The website should load successfully.

## Step 7: Confirm that S3 is private

Try to access the object directly through S3:

```text
https://aws-private-static-site-demo.s3.ap-south-1.amazonaws.com/index.html
```

The expected response is:

```text
AccessDenied
```

This is correct. It confirms that the website can be accessed through CloudFront but not directly through S3.

## Step 8: Update website content

1. Upload the updated files to the same S3 object paths.
2. Open the CloudFront distribution.
3. Select **Invalidations**.
4. Select **Create invalidation**.
5. Enter:

```text
/*
```

6. Create the invalidation and wait until its status becomes **Completed**.

For routine deployments, version CSS and JavaScript filenames to reduce full-cache invalidations, for example:

```text
style.v2.css
app.v2.js
```

## Troubleshooting

### CloudFront returns AccessDenied

Check the following:

- The bucket policy contains the correct AWS account ID and CloudFront distribution ID.
- The `index.html` file exists at the bucket root.
- The CloudFront origin uses the standard S3 endpoint, not an `s3-website` endpoint.
- The OAC is attached to the CloudFront origin.
- OAC request signing is enabled.
- The distribution has finished deploying.
- A cached 403 response has been cleared using a CloudFront invalidation.

### `/index.html` works, but `/` fails

Set the CloudFront **Default root object** to:

```text
index.html
```

Do not use `/index.html`.

### Direct S3 URL returns AccessDenied

This is expected because the S3 bucket is private. Use the CloudFront domain to access the website.

## Security Summary

- S3 Block Public Access: **Enabled**
- S3 static website hosting: **Disabled**
- S3 Object Ownership: **Bucket owner enforced**
- CloudFront OAC: **Enabled**
- CloudFront request signing: **Enabled**
- Viewer access: **HTTPS**
- Direct public S3 access: **Denied**
- Bucket policy access: **Restricted to one CloudFront distribution**
