## s3-upload-stream

A basic pipeable write stream which uploads to Amazon S3 using the multipart file upload API.

Advantages
----------

- You don't need to know the size of the stream prior to uploading, which works excellently for cases where you are creating files from dynamic content. (I created this module to solve the problem of taking large amounts of incoming text, compressing it on the fly, and uploading it directly to S3, all done via streams without saving any files to disk or keeping any chunk of data in memory for an excessive period of time.)
- Easy to use, clean piping mechanism.
- Uses the official Amazon SDK for Node.js
- You can provide options for the upload call directly to do things like set server side encryption, reduced redundancy storage, or access level on the object, which I found some other similar modules to be lacking.
- Emits "chunk" events which expose the amount of incoming data received by the writable stream versus the amount of data that has been uploaded via the multipart API so far.

Problems
--------

- The multipart upload API does not accept chunks less than 5mb in size. So although this module emits "chunk" events which can be used to show progress, the progress is not very granular, as the chunk events are only emitted every 5 MB, which even on the fastest connections isn't very frequently. (Note: this could be fixed by abandoning the official API, and making direct requests to S3, and piping each 5mb chunk directly into the web request, allowing for much more granular progress reports.)
- There may be some nasty issues with tremendously large, fast incoming streams that can't upload as fast as they fill up the buffers. The stream to S3 pulls about 5mb of data from the incoming stream and then writes it all to S3. Meanwhile while it writes to S3 it pauses the incoming data stream until the upload of that part completes. So if that incoming stream is meanwhile filling up all the buffers with tons of data then bad things might happen.
- I can't think of a good way right now to provide tests without giving away my AWS credentials. Maybe there is a way to make a fake S3 service for the purpose of making tests for the library, but honestly that is way too much work for such a simple module.

Installation
------------

```
npm install s3-upload-stream
```

Usage
-----

``` javascript
	var Uploader = require('s3-upload-stream').Uploader,
		zlib       = require('zlib'),
		fs         = require('fs');

	var read = fs.createReadStream('./path/to/file.ext');
	var compress = zlib.createGzip();

	var UploadStreamObject = new Uploader(
		//Connection details.
		{
			"accessKeyId": "REDACTED",
			"secretAccessKey": "REDACTED",
			"region": "us-east-1"
		},
		//Upload destination details.
		{
			"Bucket": "your-bucket-name",
			"Key": "uploaded-file-name " + new Date()
		},
		function (err, uploadStream)
		{
			if(err)
				console.log(err, uploadStream);
			else
			{
				uploadStream.on('chunk', function (data) {
					console.log(data);
				});
				uploadStream.on('uploaded', function (data) {
					console.log(data);
				});

				//Pipe the file stream through Gzip compression and upload result to S3.
				read.pipe(compress).pipe(uploadStream);
			}
		}
	);
```
