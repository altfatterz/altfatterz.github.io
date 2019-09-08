---
layout: post
title: Signed URLs with Google Cloud Storage bucket
tags: [mongodb]
---

Let's say you have an object in a [Google Cloud Storage](https://cloud.google.com/storage/) bucket which is set to be private. You want to share it with people who have no Google Cloud account, for example, subscribed visitors to your website.
This can be a video course that only paying users can access, or an E-book that requires subscription.

[Signed URLs](https://cloud.google.com/storage/docs/access-control/signed-urls) are good solution for this problem. They give a time-limited resource access to anyone in possession of the URL, regardless of whether they have a Google account or not.
                                                
The signed URL contains authentication information in its query string allowing users without credentials to perform specific actions on a resource.

In this blog post we show how to provide users a time-limited read access to an object inside a Google Cloud Storage bucket. 

## Google Cloud Storage bucket

We create a `bucket` and upload a file to it.

```bash
$ gsutil mb gs://signed-url-demo
$ echo "Since you are a subscribed user, you get this resource." > demo.txt
$ gsutil cp demo.txt gs://signed-url-demo
```

We make sure the file was uploaded.

```bash
$ gsutil ls -r gs://signed-url-demo

gs://signed-url-demo/demo.txt
```

We verify that the file is private.

```bash
$ http https://storage.googleapis.com/signed-url-demo/demo.txt

HTTP/1.1 403 Forbidden
Alt-Svc: quic=":443"; ma=2592000; v="46,43,39"
Cache-Control: private, max-age=0
Content-Length: 216
Content-Type: application/xml; charset=UTF-8
Date: Sun, 08 Sep 2019 10:48:40 GMT
Expires: Sun, 08 Sep 2019 10:48:40 GMT
Server: UploadServer
X-GUploader-UploadID: AEnB2UpTLJO2792GLnoNGHnyhuEiLOyoal4ibgogvHAX-6dyO0glTU6uF8wmmnliF7repZbKGOQArmz2V52vSppuOFYmTqCjkg

<?xml version='1.0' encoding='UTF-8'?><Error><Code>AccessDenied</Code><Message>Access denied.</Message><Details>Anonymous caller does not have storage.objects.get access to signed-url-demo/demo.txt.</Details></Error>
```

## Create a service account

Next we create a `service account`.

```bash
$ gcloud iam service-accounts create signed-url-demo --display-name "Signed URL demo" 
```

We verify that the `service account` was created.

```bash
$ gcloud iam service-accounts list

NAME                                    EMAIL                                               DISABLED
...
Signed URL demo                         signed-url-demo@altfatterz.iam.gserviceaccount.com  False
... 
```

## Grant the `Storage Object Viewer` role to the service account
 
```bash
$ gcloud projects add-iam-policy-binding altfatterz \
  --member serviceAccount:signed-url-demo@altfatterz.iam.gserviceaccount.com \
  --role roles/storage.objectViewer
```

In the response we see the updated `IAM policy` for the `altfatterz` project

```bash
Updated IAM policy for project [altfatterz].
bindings:
...
- members:
  ...
  - serviceAccount:signed-url-demo@altfatterz.iam.gserviceaccount.com
  role: roles/storage.objectViewer
```     

## Create a service account key

Next, we create a `key` for the `service account` and store it in the `key.json` file

```bash
$ gcloud iam service-accounts keys create key.json \
    --iam-account signed-url-demo@altfatterz.iam.gserviceaccount.com
```

## Use the `signurl` command 

And finally we can use the `signurl` command to generate a signed URL that embeds authentication data so the URL can be used by someone who does not have a Google account. 

```bash
$ gsutil signurl -d 1m key.json gs://signed-url-demo/demo.txt
```

The `signurl` command uses the private key (`key.json`) of the service account to generate the cryptographic signature for the generated URL.

```bash
URL                             HTTP Method	    Expiration	        Signed URL
gs://signed-url-demo/demo.txt	GET	            2019-09-08 12:24:28	https://storage.googleapis.com/signed-url-demo/demo.txt?x-goog-signature=8f4afba14b4ef24c2af1975bee453383c7a290bca3ca3db2e11350889caa4e48cc11cccfd099a1ccf6763185f36a33862802502362aa9fdc0095dfd7075bd274ce0b99c42b7e4a2a30e7c247d55195b83d1bb45ea390ab947ea51e086c0d948fc61ee2dcfe9304f8affdc267bee05dd45daf6e279da259c1c574a00d94908e707acec1641bec3c1f88058c063affd96c3aa4431f961f622e88d964ae6243239c9e4ecdecd0153e39bb5a6e806c5c0ff6c3da26f3377fbae9fd39451553469bca0f0b9281fc8103ff450d36848298bd49605fd76dcba6465a52ecfc125952f9642902ae332ad7c0914b2a56e2c1a54f20ab937429adc340b2c54f6437a5ad4a9a&x-goog-algorithm=GOOG4-RSA-SHA256&x-goog-credential=signed-url-demo%40altfatterz.iam.gserviceaccount.com%2F20190908%2Fus%2Fstorage%2Fgoog4_request&x-goog-date=20190908T102328Z&x-goog-expires=60&x-goog-signedheaders=host
```

The `signurl` command requires the `pyopenssl` library, which you can easily install with `pip install pyopenssl`

And now we access the file using the signed URL.

```bash
$ http https://storage.googleapis.com/signed-url-demo/demo.txt\?x-goog-signature\=8f4afba14b4ef24c2af1975bee453383c7a290bca3ca3db2e11350889caa4e48cc11cccfd099a1ccf6763185f36a33862802502362aa9fdc0095dfd7075bd274ce0b99c42b7e4a2a30e7c247d55195b83d1bb45ea390ab947ea51e086c0d948fc61ee2dcfe9304f8affdc267bee05dd45daf6e279da259c1c574a00d94908e707acec1641bec3c1f88058c063affd96c3aa4431f961f622e88d964ae6243239c9e4ecdecd0153e39bb5a6e806c5c0ff6c3da26f3377fbae9fd39451553469bca0f0b9281fc8103ff450d36848298bd49605fd76dcba6465a52ecfc125952f9642902ae332ad7c0914b2a56e2c1a54f20ab937429adc340b2c54f6437a5ad4a9a\&x-goog-algorithm\=GOOG4-RSA-SHA256\&x-goog-credential\=signed-url-demo%40altfatterz.iam.gserviceaccount.com%2F20190908%2Fus%2Fstorage%2Fgoog4_request\&x-goog-date\=20190908T102328Z\&x-goog-expires\=60\&x-goog-signedheaders\=host

HTTP/1.1 200 OK
Accept-Ranges: bytes
Alt-Svc: quic=":443"; ma=2592000; v="46,43,39"
Cache-Control: private, max-age=0
Content-Language: en
Content-Length: 55
Content-Type: text/plain
Date: Sun, 08 Sep 2019 10:23:44 GMT
ETag: "ef8836fdf24851afb3ea40377e4fed52"
Expires: Sun, 08 Sep 2019 10:23:44 GMT
Last-Modified: Sun, 08 Sep 2019 10:06:51 GMT
Server: UploadServer
X-GUploader-UploadID: AEnB2UqiI6vBXIybnBx9P80LSNlmxARPcXlJjA-XY8ErQyumQ3rSuh_kefOYbvFYpVoRaD0oh12uQYMx9sKGpNJuj9dSVVQWDA
x-goog-generation: 1567937211730069
x-goog-hash: crc32c=eDfjkw==
x-goog-hash: md5=74g2/fJIUa+z6kA3fk/tUg==
x-goog-metageneration: 1
x-goog-storage-class: STANDARD
x-goog-stored-content-encoding: identity
x-goog-stored-content-length: 55

Since you are a subscribed user, you get this resource.
```

With the `-d` parameter we specified that duration until the signed url should be valid (default is 1 hour).

In this case after a minute we get `400 Bad Request`

```bash
$ http https://storage.googleapis.com/signed-url-demo/demo.txt\?x-goog-signature\=8f4afba14b4ef24c2af1975bee453383c7a290bca3ca3db2e11350889caa4e48cc11cccfd099a1ccf6763185f36a33862802502362aa9fdc0095dfd7075bd274ce0b99c42b7e4a2a30e7c247d55195b83d1bb45ea390ab947ea51e086c0d948fc61ee2dcfe9304f8affdc267bee05dd45daf6e279da259c1c574a00d94908e707acec1641bec3c1f88058c063affd96c3aa4431f961f622e88d964ae6243239c9e4ecdecd0153e39bb5a6e806c5c0ff6c3da26f3377fbae9fd39451553469bca0f0b9281fc8103ff450d36848298bd49605fd76dcba6465a52ecfc125952f9642902ae332ad7c0914b2a56e2c1a54f20ab937429adc340b2c54f6437a5ad4a9a\&x-goog-algorithm\=GOOG4-RSA-SHA256\&x-goog-credential\=signed-url-demo%40altfatterz.iam.gserviceaccount.com%2F20190908%2Fus%2Fstorage%2Fgoog4_request\&x-goog-date\=20190908T102328Z\&x-goog-expires\=60\&x-goog-signedheaders\=host

HTTP/1.1 400 Bad Request
Alt-Svc: quic=":443"; ma=2592000; v="46,43,39"
Cache-Control: private, max-age=0
Content-Length: 202
Content-Type: application/xml; charset=UTF-8
Date: Sun, 08 Sep 2019 10:25:23 GMT
Expires: Sun, 08 Sep 2019 10:25:23 GMT
Server: UploadServer
X-GUploader-UploadID: AEnB2UpomRy06euJeNlmt-nMWJAxCIbGJUJCmaz2t08s4KXvy2hXUySRS62J77d2JV07o5hmRmxqeVuJMTytQU3KVyPq-Bd6gQ

<?xml version='1.0' encoding='UTF-8'?><Error><Code>ExpiredToken</Code><Message>The provided token has expired.</Message><Details>Request signature expired at: 2019-09-08T10:24:28+00:00</Details></Error>
``` 
