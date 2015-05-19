# Segment service

*Segment service* allows third parties to add/remove users from segments.
Those segments should be created in advance by its advertisers.

*IMPORTANT* Before you can use this service you need to send an enquire to [support@ligamatic.com](mailto://support@ligamatic.com).

## API

Our api is a simple to use REST JSON one. The examples of this manual use [curl](http://curl.haxx.se/) and a little bit of shell scripting. If you have any trouble or question please open an issue.  
The service address is `https://dsp-suite.ligamatic.com`.  
Calls to its equivalent non secure (http) countertpart will just not work.  
Please use always the `https` protocol.

1. [Authentication](#authentication)
2. [Query segments for a given advertiser](#query-segments-for-a-given-advertiser)
3. [Include/Exclude users from a given segment](#includeexclude-users-from-a-given-segment)

### Authentication

To authenticate your requests to our system you need to add an authentication header to them.
The header has the following morphology:
```
authentication: Token $user_id;$timespan;sha1Hash($timespan$password)
```
Where  
1. `$user_id` is your user id  
2. `$timespan` [unix timespan](https://en.wikipedia.org/wiki/Unix_time)  
3. `$password` is the password we have already provided to you.  

*Important* If there is a difference of +300 seconds between your submited unix timespan and our unix timespan we will return a `403` with a `bad timespan` message.
All our responses include our `timespan` in the the `timespan` header. You can store the difference in a variable and add it to your `timespan` in every request.

A shell example
```
user_id=user
timespan='1433331278'
password='secret'

auth="authentication: Token $user_id;$timespan;`echo -n "$timespan" | openssl sha1 -binary |base64`"
# $auth value is 'authentication: Token user;1433331307;bGiDvpb0NW7ZFtuUcgU9LILvNR8='

# 
# Use the authentication header with curl
curl -H "$auth" "https://dsp-suite.ligamatic.com..."
```

### Query segments for a given advertiser

```
curl -H "$auth" "https://dsp-suite.ligamatic.com/advertisers/a0/segments"
```

### Create a new segment

In order to create a new segment you need to post it to `/advertisers/:advertiser_id/segments`.  
`name` is mandatory. Use alphanumeric characters plus `-_` and spaces.

```
curl -v -H 'content-type: application/json' -H "$auth" -X POST "https://dsp-suite.ligamatic.com/advertisers/a0/segments"  -d '{"name": "my segment"}' 
```

You can also attach meta data to segments. Your meta data is only visible to you. Other parties will not have acces to it.
Meta data fields start with underscore.
```
// example JSON segment with meta data
{
   "name": "my segment",
   "_code": "123",
}
```

*You can submit up to 128 bytes of data.*

### Include/exclude users from a given segment

You can include/exclude users from a given segment by sending a list of them to our service. That request will create an update job in our system. 
This service accepts two files `include` and `exclude`. Both are a newline `\n` delimited lists of d3 media user IDs.  

A sample file:

```
user1
user2
user3
```

You can call this services with both files or just one. Here is curl example.

```
advertiser_id=a0
segment_id=a0g0
curl -v -H "$auth" \
-F "include=@file1" \
-F "exclude=@file2" \ 
"https://dsp-suite.ligamatic.com/advertisers/${advertiser_id}/segments/${segment_id}"
```

If everything went fine you will receive a status 201 and a *status link header*

```

# REQUEST

> POST /advertisers/a0/segments/a0g0 HTTP/1.1
> User-Agent: curl/7.35.0
> Host: dsp-suite.ligamatic.com
> Accept: */*
> Content-Length: 366
> Expect: 100-continue
> Content-Type: multipart/form-data; boundary=------------------------16bc9dbff92e8f0e
> 

# RESPONSE

< HTTP/1.1 100 Continue
< HTTP/1.1 201 Created
< timespan: 1433331657
< ETag: 1-92945be834c037edd12fcdca525f9245
< Link: <https://dsp-suite.ligamatic.com/advertisers/a0/segments/a0g0/jobs/7eb683df-78f2-4de6-b5d1-9d1329523804>; rel="status"
< Date: Tue, 19 May 2015 12:07:11 GMT
< Connection: keep-alive
< Transfer-Encoding: chunked
< 

```

The *status link header* points to a url you can query to check the state of the job. Jobs can take a certain of time to finish so please do not query the status every 100ms.  
Instead of polling us, you can specify a url to we can call you back when the job is done. To do that, you need to inclued in the querystring the `callback` parameter.
This parameter is an url (encoded please). We will post the result of the job to that url. In order to simplify your integration with us you can add `{job_id}` to any part of your url. We will replace it with the job id.

```
# PLEASO NOTE {job_id} has been escaped to \{job_id\} so bash does not expand it.
# In a regular integration you should encode the whole callback parameter value
advertiser_id=a0
segment_id=a0g0
curl -v -H "$auth" \
-F "include=@file1" \
-F "exclude=@file2" \
"https://dsp-suite.ligamatic.com/advertisers/${advertiser_id}/segments/${segment_id}?callback=https://service.io/jobdone?id=\{job_id\}"
```
