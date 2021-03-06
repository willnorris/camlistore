# Blob Stat Protocol

This document describes the "batch stat" API end-point, for checking
the size/existence of multiple blobs when the client and/or server do
not support HTTP/2.0.  See [blob-upload](blob-upload.md) for more
background.

Notably: the HTTP method may be GET or POST.  GET is more correct but
be aware that HTTP servers and proxies start to suck around the 2k and
4k URL lengths.  If you're stat'ing more than ~37 blobs, using POST
would be safest.

The HTTP request path is `$blobRoot/camli/stat`.  See
[blob-upload](blob-upload.md) and [discovery](discovery.md) for background.

In either case, the request form values: (either in the URL for GET or
application/x-www-form-urlencoded body for POST)

    camliversion    required   Version of Perkeep and/or stat protocol;
                               reserved for future use.  Must be "1" for now.

    blob<n>         optional/  Must start at 1 and go up, no gaps allowed, not
                    repeated   zero-padded, etc.  Value is a blobref, e.g
                               "sha1-9b03f7aca1ac60d40b5e570c34f79a3e07c918e8"
                               There's no defined limit on how many you include here,
                               but servers may return a 400 Bad Request if you ask
                               for too many.  All servers should support <= 1000
                               though.

    maxwaitsec      optional   The client may send this, an integer max number
                               of seconds the client is willing to wait
                               for the arrival of blobs.  If the server
                               supports long-polling (an optional
                               feature), then the server will return
                               immediately if all the requested blobs
                               or available, or wait up until this
                               amount of time for the blobs to become
                               available.  The server's reply must
                               include "canLongPoll" set to true if the
                               server supports this feature.  Even if
                               the server supports long polling, the
                               server may cap 'maxwaitsec' and wait for
                               less time than requested by the client.

## Examples

    GET /some-blob-root/camli/stat?camliversion=1&blob1=sha1-9b03f7aca1ac60d40b5e570c34f79a3e07c918e8 HTTP/1.1
    Host: example.com

 -or-

    POST /some-blob-root/camli/stat HTTP/1.1
    Content-Type: application/x-www-form-urlencoded
    Host: example.com

    camliversion=1&
    blob1=sha1-9b03f7aca1ac60d40b5e570c34f79a3e07c918e8&
    blob2=sha1-abcdabcdabcdabcdabcdabcdabcdabcdabcdabcd&
    blob3=sha1-deadbeefdeadbeefdeadbeefdeadbeefdeadbeef

Response:

    HTTP/1.1 200 OK
    Content-Length: ...
    Content-Type: text/javascript

    {
       "stat": [
          {"blobRef": "sha1-abcdabcdabcdabcdabcdabcdabcdabcdabcdabcd",
           "size": 12312}
       ],
       "canLongPoll": true
    }

Response keys:

    stat             required   Array of {"blobRef": BLOBREF, "size": INT_bytes}
                                for blobs that the system already has. Empty
                                list if no blobs are already present.

    canLongPoll      optional   Set to true (type boolean) if the server supports
                                long polling.  If not true, the server ignores
                                the client's "maxwaitsec" parameter.

