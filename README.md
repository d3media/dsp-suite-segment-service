# Segment service

*Segment service* allows third parties to add/remove users from segments.
Those segments should be created in advance by its advertisers.

*IMPORTANT* This is not a public API.  In order to get access you must have an account.  Please contact [support@ligamatic.com](mailto://support@ligamatic.com).

## API

This API is REST/JSON.  The following examples use [curl](http://curl.haxx.se/) and a little bit of shell scripting. If you have any issues, feel free to contact support at the email above.

1. [Service Address](#service-address)
2. [Authentication](#authentication)
3. [Query segments for a given advertiser](#query-segments-for-a-given-advertiser)
4. [Include/Exclude users from a given segment](#includeexclude-users-from-a-given-segment)
5. [Checking Update Status](#checking-update-status)
6. [Implementations](#implementations)

### Service Address

The API service can be accessed at `https://dsp-suite.ligamatic.com`.  
Calls to the non-secure (http) url will not work.

### Authentication

All requests must be authenticated.
Requests are authenticated via an authentication header that you provide on each request.
The header has the following form:
```
authentication: Token $user_id;$timestamp;sha1Hash($timestamp$password)
```
Where  
1. `$user_id` is your user id  
2. `$timestamp` [unix timestamp](https://en.wikipedia.org/wiki/Unix_time)  
3. `$password` is the password we have already provided to you.  

*Important* If there is a difference of +/- 300 seconds between your submitted unix timestamp and our unix timestamp we will return a `403` with a `bad timestamp` message.  This is to prevent replay attacks on your account.
All of our responses include our `timestamp` in the the `timestamp` header. You can store the difference from your server clock and add/subtract it to your `timestamp` in every request.  If you are keeping your server clocks updated with ntpd this shouldn't be an issue.

A shell example
```
user_id='user'
timestamp='1433331278'
password='secret'

# build the authentication header
auth="authentication: Token $user_id;$timestamp;`echo -n "$timestamp" | openssl sha1 -binary | base64`"
# example: $auth value is 'authentication: Token user;1433331307;bGiDvpb0NW7ZFtuUcgU9LILvNR8='

# Use the authentication header with curl
curl -H "$auth" "https://dsp-suite.ligamatic.com..."
```

### Query segments for a given advertiser

In your DSP account you likely have multiple advertisers.  
The following method returns a list of DSP segments for a given advertiser.

```
# a0 is an advertiser id under your account
curl -H "$auth" "https://dsp-suite.ligamatic.com/advertisers/a0/segments"
```

### Create a new segment

In order to create a new segment you need to post it to `/advertisers/:advertiser_id/segments`.  
`name` is mandatory. Use alphanumeric characters plus `-_` and spaces.

This will create a new segment in your DSP account under the specified advertiser and store the mapping in the DSP-Suite system.

```
curl -v -H 'content-type: application/json' -H "$auth" -X POST "https://dsp-suite.ligamatic.com/advertisers/a0/segments"  -d '{"name": "my segment"}' 
```


You can also attach meta-data to segments.  Meta-data is useful for simplifying your technical integration with us as you can leave data for your application associated with the segments in our system.
Meta data fields start with underscore.
```
// example JSON segment with meta data
{
   "name": "my segment",
   "_code": "123",
}
```

*You can add up to 128 bytes of data per segment.*

### Include/exclude users from a given segment

You can include/exclude users from a given segment by sending a `segment` update file. That request will create an update job in our system which will be processed in near real-time.  We will use your update file and our internal mapping in order to update the users of the specified segment in your connected DSP.

A sample segment update file:

```
user1:-1
user2:120
user3:0
```
The format is: `d3 user id:lifetime` where the lifetime (specified in minutes) is:
* 0 means that the user will be never removed from this segment
* -1 means that the user will be removed from this segment.
* Time to live (TTL) expressed in seconds, ie: 30 means 30 seconds

*If you want to specify segments in terms of hashed emails (etc) or your own cookie IDs instead of d3 cookie IDs, let us know.  We also support this.*

Here is a curl example.

```
advertiser_id=a0
segment_id=a0g0
curl -v -H "$auth" \
-F "segment=@segment" \
"https://dsp-suite.ligamatic.com/advertisers/${advertiser_id}/segments/${segment_id}"
```
*IMPORTANT* in the above example, the segment file name must be `segment`, the delimiter within the segment file must be a colon (`:`).

If everything worked you will receive a status 201 and a *Link* header containing a url to check the status of the update.

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

### Checking Update Status

The segment in your connected DSP may take a while to update depending on the size of your segment and the speed of the DSP API.  It happens in real-time from our system but the update doesn't complete instantly.

The *status link header* contains a url you can query to check the state of the update. 
Please limit your update checks to once every 10 seconds or so as DSP APIs take a while to update with large segment lists.

Instead of polling/waiting, you can specify a url ("webhook") that we will call when the job is done. 
You can specify your call-back/webhook URL as a querystring variable with name `callback`.
Please URL encode your URL before inserting into querystring to avoid problems.
We will POST the results of the segment update to your URL. 
Your call-back URL can optionally include the string `{job_id}` and we will replace this with the job id prior to callback.


```
# PLEASE NOTE {job_id} has been escaped to \{job_id\} so bash does not expand it.
# In a regular integration you should encode the whole callback parameter value
advertiser_id=a0
segment_id=a0g0
curl -v -H "$auth" \
-F "segment=@segment" \
"https://dsp-suite.ligamatic.com/advertisers/${advertiser_id}/segments/${segment_id}?callback=https://service.io/jobdone?id=\{job_id\}"
```

### Implementations
* [node.js](https://github.com/d3media/ligamatic-api-node)
