---
title: "Regression in aws-sdk-v3 PutObject when using node.js streams"
date:  2025-02-08 11:29:00
description: "Regression in aws-sdk-v3 PutObject when using streams"
keywords: aws-sdk-v3, javascript, node.js, PutObject, PutObjectCommand, streams, express, regression, middleware-flexible-checksums
---

### Update since posting

There is a simple fix for this. We can disable the new checksum features by using `requestChecksumCalculation: 'WHEN_REQUIRED'` when constructing the `S3Client`. e.g.
```
const client = new S3Client({
  apiVersion: '2006-03-01',
  region: 'eu-west-2',
  requestChecksumCalculation: 'WHEN_REQUIRED'
})
```

This appears to skip the manipulation of the headers and allows the stream to work correctly as it did before. More details on [github](https://github.com/aws/aws-sdk-js-v3/issues/6810).

### The original post

When storing objects in S3 from a node.js application server, streams allow a method to store the body of a user's request without buffering to disk, or keeping a full in-memory footprint. Consider the following example using express:

```javascript
app.put('/example', (req, res, next) => {
  /* validation */
  client.send(new PutObjectCommand({
    Bucket: targetS3Bucket,
    Key: targetS3Key,
    ContentLength: Number(req.headers['content-length']),
    ContentMD5: req.headers['content-md5'],
    Body: req, // represents a stream of the request body
  }))
    .then(() => res.sendStatus(200))
    .catch(next);
});
```

This worked until `@aws-sdk/client-s3` version 3.726.1. However, in 3.729.0 the command fails over a WAN[^1] and the following is written to console:
```plaintext
Are you using a Stream of unknown length as the Body of a PutObject request? Consider using Upload instead from @aws-sdk/lib-storage.
```
the `PutObjectCommand` itself fails with error:
```
InvalidChunkSizeError: Only the last chunk is allowed to have a size less than 8192 bytes
```

### Isolating the problem

Are we using a stream of unknown length? No - we know the length. We provide the length as the `ContentLength` field. Tracing that error message back to the [aws-sdk-v3 middleware-sdk-s3 source](https://github.com/aws/aws-sdk-js-v3/blob/376b453aecbc3309def497d4eeddd983a1a4f44a/packages/middleware-sdk-s3/src/check-content-length-header.ts#L32) suggests that the `Content-Length` header is being checked on the request object itself. But again, that should be there, given it's where we source `ContentLength` from.

Adding some diagnostic logging:
* In 3.726.1 the `content-length` header is present
* In 3.729.0 the `content-length` header is absent, but a newly added `x-amz-decoded-content-length` header is present instead.

Searching the aws-sdk-v3 source code, the headers are being manipulated in the [`middleware-flexible-checksums` module](https://github.com/aws/aws-sdk-js-v3/blob/376b453aecbc3309def497d4eeddd983a1a4f44a/packages/middleware-flexible-checksums/src/flexibleChecksumsMiddleware.ts#L129) which is newly added to the middleware applied to requests in version 3.729.0.

As an experiment, I altered the source code to retain the `content-length` header. This changes the error seen to one related to checksum validation.

### Can we bypass the problematic code?

Seemingly not.

Manipulation of the arguments to `PutObjectCommand` does not seem to allow us to avoid going through the flexible checksums middleware, or to avoid the code path which changes the headers. As such we seem stuck - streaming HTTP requests into `PutObjectCommand` seems impossible from 3.729.0 onwards.

### How to resolve then?

```
Consider using Upload instead from @aws-sdk/lib-storage.
```

Ok, let's do that. The reason we opted against the `Upload` helper historically was due to consistency checking. When automatically split into parts the user-provided `Content-MD5` [^2] digest would not match the digest calculated AWS-side. We'll mitigate that by setting part size artificially high to avoid splitting.

```javascript
  const upload = new Upload({
    client,
    partSize: 100 * 1024 * 1024,
    params: {
      Bucket: targetS3Bucket,
      Key: targetS3Key,
      ContentLength: Number(req.headers['content-length']),
      ContentMD5: req.headers['content-md5'],
      Body: req, // represents a stream of the request body
    },
  });
  upload.done()
    .then(() => res.sendStatus(200))
    .catch(next);
```

This resolves our problem while keeping the feature-set of the original.

### See also
* [Announcement: S3 default integrity change](https://github.com/aws/aws-sdk-js-v3/issues/6810)

[^1]: When running a local server the error does not occurr. To reproduce, consider using `curl` with the `--limit-rate` option to simulate a bandwidth-restricted environment like a WAN.
[^2]: Historically `ContentMD5` was the only option you had to ensure a consistency check took place.
