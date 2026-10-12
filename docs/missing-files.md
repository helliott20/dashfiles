# Files that show `NoSuchKey`

## What's happening

An upload happens in two steps:

1. `upload.lambda` writes the file's record to DynamoDB and returns a presigned S3 URL.
2. The browser then `PUT`s the file straight to S3.

If step 2 never finishes, the record exists but S3 has no object for it. That happens when the upload times out, the visitor closes the tab, or the network drops. It also happens if the object is deleted later, by hand or by a lifecycle rule. Opening the file then redirects to S3, which answers with:

```xml
<Error><Code>NoSuchKey</Code><Message>The specified key does not exist.</Message>...</Error>
```

The file's data was never stored (or is gone), so these files **can't be recovered**. The fix is to stop creating them, flag the ones that exist, and clean them up.

## What's already fixed in this repo

- `index.html`: the upload timeout used to be a fixed 2 minutes, which cut off large uploads (up to 100 MB) on slower connections. It now scales with file size (minimum 5 minutes). The page also warns before you leave mid-upload.
- `admin.html`: when the admin API marks a file as missing, the dashboard:
  - shows a red **Missing** badge on it
  - shows a banner with **Show only these** and **Select all to delete**
  - shows a toast instead of opening S3's XML error page

  Files less than 15 minutes old are shown as **Uploading** rather than missing.

The admin page only knows a file is missing if the API tells it, so it needs change 1 below.

## Change 1: admin list Lambda (`GET /f/admin/files`)

Add a `missing` flag to each file. This Lambda already returns `bucketSize`, so it probably lists the bucket already. If so, reuse that listing instead of calling `listAllKeys` again.

```js
// List every key in the bucket (paginated, 1000 per call)
async function listAllKeys(bucket) {
    const keys = new Set();
    let ContinuationToken;
    do {
        const res = await s3.listObjectsV2({ Bucket: bucket, ContinuationToken }).promise();
        for (const obj of res.Contents || []) keys.add(obj.Key);
        ContinuationToken = res.IsTruncated ? res.NextContinuationToken : undefined;
    } while (ContinuationToken);
    return keys;
}

// ...after scanning DynamoDB into `files`, before returning:
const keys = await listAllKeys(process.env.BUCKET_NAME);
files = files.map(f => ({ ...f, missing: !keys.has(f.s3Key) }));
```

The Lambda's role needs `s3:ListBucket` on the bucket.

## Change 2: share Lambda (`GET /f/{fileId}`)

So people opening a share link get a friendly page instead of the XML error, check that the object exists before redirecting:

```js
try {
    await s3.headObject({ Bucket: process.env.BUCKET_NAME, Key: item.s3Key }).promise();
} catch (err) {
    // Without s3:ListBucket, S3 reports a missing key as 403 instead of 404
    if (err.code === 'NotFound' || err.statusCode === 404 || err.statusCode === 403) {
        return {
            statusCode: 410,
            headers: { 'Content-Type': 'text/html; charset=utf-8' },
            body: `<!doctype html><meta name="viewport" content="width=device-width, initial-scale=1">
<title>File not available</title>
<body style="font-family:system-ui,sans-serif;max-width:480px;margin:15vh auto;padding:0 16px;text-align:center;color:#0f172a">
<h1 style="font-size:1.4rem">This file isn't available</h1>
<p style="color:#475569">It may not have finished uploading, or it has been removed.</p>
<p><a href="https://dashfiles.co/" style="color:#2563eb">Upload a file on DashFiles</a></p></body>`
        };
    }
    throw err;
}
// ...then redirect to the signed URL as before
```

`headObject` uses the same `s3:GetObject` permission the Lambda already has for signing download links. Add `s3:ListBucket` too, so a missing key comes back as a clean 404.

## Optional: mark uploads as complete

To track this in the data itself, add an S3 `ObjectCreated` event notification on the bucket. Point it at a small Lambda that sets `status = 'ready'` on the matching DynamoDB record; the key is `${fileId}.${ext}`, so the `fileId` is the part before the first `.`. Records still `pending` after an hour are failed uploads, and a scheduled job can delete them.
