---
title: boto3给HTTP签名
categories: 
   - aws sdk
   - dev
tags: 
    - boto3 
    - http
    - Python
---

## 为什么要签名
通常来说HTTP请求是没有办法携带IAM身份凭证的，日常的请求通过AWS SDK已经足够，但是仍然有些服务暴露出来的是REST API，用API的话只是提供一个接口，无关内部实现和具体语言调用。只要你的程序支持web，就可以使用这个REST API。

下边这个例子我们用到三个库，**boto3**，**requests**，**requests_aws4auth**

```python
iimport boto3
import requests  
from requests_aws4auth import AWS4Auth  

Url = 'requesturl'
region = 'cn-north-1'
service = 's3'
credentials = boto3.Session().get_credentials()
awsauth = AWS4Auth(credentials.access_key, credentials.secret_key, region, service)


request = requests.get(Url,auth=AWS4Auth)
```
官方其实提提供了两种办法，如下：
```python
# 1. 用普通参数传入，这里用位置参数会报错。
import requests
from requests_aws4auth import AWS4Auth
auth = AWS4Auth('<ACCESS ID>', '<ACCESS KEY>', 'eu-west-1', 's3')

# 2.使用STS身份，都是位置参数
from requests_aws4auth import AWS4Auth
from botocore.session import Session
credentials = Session().get_credentials()
auth = AWS4Auth(region='eu-west-1', service='es',
refreshable_credentials=credentials)
```

AWS4Auth这个类的构造器是这样的，我们看一下源码。
入口处两个参数 \*args，\*\*kwargs,  关于这两参数，前者代表传入数组，后者代表传入字典，更多内容在geeksforgeeks有说明[^args]。这里有个坑，就是构造器限制数组的长度只能是0，2，4，5，如果检测到其他直接会报错type error。如果把给数组的参数给字典，也就是把位置参数写成关键字参数会报错，源码没有就这点进行说明。

1.  **if self.refreshable_credentials** 检测是不是使用的STS身份，如果是就按照名字取字典的数值。
2. **else if l not in [2, 4, 5]:** 这里检测数组的长度
    1. 如果是len == 2， 那么参数是access_id，AWS4SigningKey，其中AWS4SigningKey又是根据secret_key, region, service，和时间制作的sign_sha256算法的签名。
    2. 如果不是那么那么需要使用secret_key, region, service，和时间重新签名。

签名算法如下
```python 
init_key = ('AWS4' + secret_key).encode('utf-8')  
date_key = cls.sign_sha256(init_key, date)  
region_key = cls.sign_sha256(date_key, region)  
service_key = cls.sign_sha256(region_key, service)  
key = cls.sign_sha256(service_key, 'aws4_request')  
if intermediates:  
    return (key, date_key, region_key, service_key)  
else:  
    return key
```
[^args]: [*args and **kwargs in Python - GeeksforGeeks](https://www.geeksforgeeks.org/args-kwargs-python/)

![](asserts/Pasted%20image%2020220925164746.png)

```python
AWS4Auth(*args, **kwargs)


self.signing_key = None  
self.refreshable_credentials = kwargs.get('refreshable_credentials', None)  
if self.refreshable_credentials:  
    # instantiate from refreshable_credentials  
    self.service = kwargs.get('service', None)  
    if not self.service:  
        raise TypeError('service must be provided as keyword argument when using refreshable_credentials')  
    self.region = kwargs.get('region', None)  
    if not self.region:  
        raise TypeError('region must be provided as keyword argument when using refreshable_credentials')  
    self.date = kwargs.get('date', None)  
    self.default_include_headers.add('x-amz-security-token')  
else:  
    l = len(args)  
    if l not in [2, 4, 5]:  
        msg = 'AWS4Auth() takes 2, 4 or 5 arguments, {} given'.format(l)  
        raise TypeError(msg)  
    self.access_id = args[0]  
    if isinstance(args[1], AWS4SigningKey) and l == 2:  
        # instantiate from signing key  
        self.signing_key = args[1]  
        self.region = self.signing_key.region  
        self.service = self.signing_key.service  
        self.date = self.signing_key.date  
    elif l in [4, 5]:  
        # instantiate from args  
        secret_key = args[1]  
        self.region = args[2]  
        self.service = args[3]  
        self.date = args[4] if l == 5 else None  
        self.regenerate_signing_key(secret_key=secret_key)  
    else:  
        raise TypeError()  
  
    self.session_token = kwargs.get('session_token')  
    if self.session_token:  
        self.default_include_headers.add('x-amz-security-token')  
  
raise_invalid_date = kwargs.get('raise_invalid_date', False)  
if raise_invalid_date in [True, False]:  
    self.raise_invalid_date = raise_invalid_date  
else:  
    raise ValueError('raise_invalid_date must be True or False in AWS4Auth.__init__()')  
  
self.include_hdrs = set(self.default_include_headers)  
  
# if the key exists and it's some sort of listable object, use it.  
if 'include_hdrs' in kwargs and isinstance(kwargs['include_hdrs'], abc.Iterable):  
    self.include_hdrs = set(kwargs['include_hdrs'])  
  
AuthBase.__init__(self)
```

**help(AWS4Auth)**  之后：


```python
Help on class AWS4Auth in module requests_aws4auth.aws4auth:

class AWS4Auth(requests.auth.AuthBase)
 |  AWS4Auth(*args, **kwargs)
 |  
 |  Requests authentication class providing AWS version 4 authentication for
 |  HTTP requests. Implements header-based authentication only, GET URL
 |  parameter and POST parameter authentication are not supported.
 |  
 |  Provides authentication for regions and services listed at:
 |  http://docs.aws.amazon.com/general/latest/gr/rande.html
 |  
 |  The following services do not support AWS auth version 4 and are not usable
 |  with this package:
 |      * Simple Email Service (SES)' - AWS auth v3 only
 |      * Simple Workflow Service - AWS auth v3 only
 |      * Import/Export - AWS auth v2 only
 |      * SimpleDB - AWS auth V2 only
 |      * DevPay - AWS auth v1 only
 |      * Mechanical Turk - has own signing mechanism
 |  
 |  You can reuse AWS4Auth instances to sign as many requests as you need.
 |  
 |  Basic usage
 |  -----------
 |  >>> import requests
 |  >>> from requests_aws4auth import AWS4Auth
 |  >>> auth = AWS4Auth('<ACCESS ID>', '<ACCESS KEY>', 'eu-west-1', 's3')
 |  >>> endpoint = 'http://s3-eu-west-1.amazonaws.com'
 |  >>> response = requests.get(endpoint, auth=auth)
 |  >>> response.text
 |  <?xml version="1.0" encoding="UTF-8"?>
 |      <ListAllMyBucketsResult xmlns="http://s3.amazonaws.com/doc/2006-03-01">
 |          <Owner>
 |          <ID>bcaf1ffd86f461ca5fb16fd081034f</ID>
 |          <DisplayName>webfile</DisplayName>
 |          ...
 |  
 |  This example lists your buckets in the eu-west-1 region of the Amazon S3
 |  service.
 |  
 |  STS Temporary Credentials
 |  -------------------------
 |  >>> from requests_aws4auth import AWS4Auth
 |  >>> auth = AWS4Auth('<ACCESS ID>', '<ACCESS KEY>', 'eu-west-1', 's3',
 |                      session_token='<SESSION TOKEN>')
 |  ...
 |  
 |  This example shows how to construct an AWS4Auth object for use with STS
 |  temporary credentials. The ``x-amz-security-token`` header is added with
 |  the session token. Temporary credential timeouts are not managed -- in
 |  case the temporary credentials expire, they need to be re-generated and
 |  the AWS4Auth object re-constructed with the new credentials.
 |  
 |  Dynamic STS Credentials using botocore RefreshableCredentials
 |  -------------------------------------------------------------
 |  >>> from requests_aws4auth import AWS4Auth
 |  >>> from botocore.session import Session
 |  >>> credentials = Session().get_credentials()
 |  >>> auth = AWS4Auth(region='eu-west-1', service='es',
 |                      refreshable_credentials=credentials)
 |  ...
 |  
 |  This example shows how to construct an AWS4Auth instance with
 |  automatically refreshing credentials, suitable for long-running
 |  applications using AWS IAM assume-role.
 |  The RefreshableCredentials instance is used to generate valid static
 |  credentials per-request, eliminating the need to recreate the AWS4Auth
 |  instance when temporary credentials expire.
 |  
 |  Date handling
 |  -------------
 |  If an HTTP request to be authenticated contains a Date or X-Amz-Date
 |  header, AWS will only accept authorisation if the date in the header
 |  matches the scope date of the signing key (see
 |  http://docs.aws.amazon.com/general/latest/gr/sigv4-date-handling.html).
 |  
 |  From version 0.8 of requests-aws4auth, if the header date does not match
 |  the scope date, the AWS4Auth class will automatically regenerate its
 |  signing key, using the same scope parameters as the previous key except for
 |  the date, which will be changed to match the request date. (If a request
 |  does not include a date, the current date is added to the request in an
 |  X-Amz-Date header).
 |  
 |  The new behaviour from version 0.8 has implications for thread safety and
 |  secret key security, see the "Automatic key regeneration", "Secret key
 |  storage" and "Multithreading" sections below.
 |  
 |  This also means that AWS4Auth is now attempting to parse and extract dates
 |  from the values in X-Amz-Date and Date headers. Supported date formats are:
 |  
 |      * RFC 7231 (e.g. Mon, 09 Sep 2011 23:36:00 GMT)
 |      * RFC 850 (e.g. Sunday, 06-Nov-94 08:49:37 GMT)
 |      * C time (e.g. Wed Dec 4 00:00:00 2002)
 |      * Amz-Date format (e.g. 20090325T010101Z)
 |      * ISO 8601 / RFC 3339 (e.g. 2009-03-25T10:11:12.13-01:00)
 |  
 |  If either header is present but AWS4Auth cannot extract a date because all
 |  present date headers are in an unrecognisable format, AWS4Auth will delete
 |  any X-Amz-Date and Date headers present and replace with a single
 |  X-Amz-Date header containing the current date. This behaviour can be
 |  modified using the 'raise_invalid_date' keyword argument of the AWS4Auth
 |  constructor.
 |  
 |  Automatic key regeneration
 |  --------------------------
 |  If you do not want the signing key to be automatically regenerated when a
 |  mismatch between the request date and the scope date is encountered, use
 |  the alternative StrictAWS4Auth class, which is identical to AWS4Auth except
 |  that upon encountering a date mismatch it just raises a DateMismatchError.
 |  You can also use the PassiveAWS4Auth class, which mimics the AWS4Auth
 |  behaviour prior to version 0.8 and just signs and sends the request,
 |  whether the date matches or not. In this case it is up to the calling code
 |  to handle an authentication failure response from AWS caused by a date
 |  mismatch.
 |  
 |  Secret key storage
 |  ------------------
 |  To allow automatic key regeneration, the secret key is stored in the
 |  AWS4Auth instance, in the signing key object. If you do not want this to
 |  occur, instantiate the instance using an AWS4Signing key which was created
 |  with the store_secret_key parameter set to False:
 |  
 |  >>> sig_key = AWS4SigningKey(secret_key, region, service, date, False)
 |  >>> auth = StrictAWS4Auth(access_id, sig_key)
 |  
 |  The AWS4Auth class will then raise a NoSecretKeyError when it attempts to
 |  regenerate its key. A slightly more conceptually elegant way to handle this
 |  is to use the alternative StrictAWS4Auth class, again instantiating it with
 |  an AWS4SigningKey instance created with store_secret_key = False.
 |  
 |  Multithreading
 |  --------------
 |  If you share AWS4Auth (or even StrictAWS4Auth) instances between threads
 |  you are likely to encounter problems. Because AWS4Auth instances may
 |  unpredictably regenerate their signing key as part of signing a request,
 |  threads using the same instance may find the key changed by another thread
 |  halfway through the signing process, which may result in undefined
 |  behaviour.
 |  
 |  It may be possible to rig up a workable instance sharing mechanism using
 |  locking primitives and the StrictAWS4Auth class, however this poor author
 |  can't think of a scenario which works safely yet doesn't suffer from at
 |  some point blocking all threads for at least the duration of an HTTP
 |  request, which could be several seconds. If several requests come in in
 |  close succession which all require key regenerations then the system could
 |  be forced into serial operation for quite a length of time.
 |  
 |  In short, it's best to create a thread-local instance of AWS4Auth for each
 |  thread that needs to do authentication.
 |  
 |  Class attributes
 |  ----------------
 |  AWS4Auth.access_id   -- the access ID supplied to the instance
 |  AWS4Auth.region      -- the AWS region for the instance
 |  AWS4Auth.service     -- the endpoint code for the service for this instance
 |  AWS4Auth.date        -- the date the instance is valid for
 |  AWS4Auth.signing_key -- instance of AWS4SigningKey used for this instance,
 |                          either generated from the supplied parameters or
 |                          supplied directly on the command line
 |  
 |  Method resolution order:
 |      AWS4Auth
 |      requests.auth.AuthBase
 |      builtins.object
 |  
 |  Methods defined here:
 |  
 |  __call__(self, req)
 |      Interface used by Requests module to apply authentication to HTTP
 |      requests.
 |      
 |      Add x-amz-content-sha256 and Authorization headers to the request. Add
 |      x-amz-date header to request if not already present and req does not
 |      contain a Date header.
 |      
 |      Check request date matches date in the current signing key. If not,
 |      regenerate signing key to match request date.
 |      
 |      If request body is not already encoded to bytes, encode to charset
 |      specified in Content-Type header, or UTF-8 if not specified.
 |      
 |      req -- Requests PreparedRequest object
 |  
 |  __init__(self, *args, **kwargs)
 |      AWS4Auth instances can be created by supplying key scope parameters
 |      directly or by using an AWS4SigningKey instance:
 |      
 |      >>> auth = AWS4Auth(access_id, secret_key, region, service
 |      ...                 [, date][, raise_invalid_date=False][, session_token=None])
 |      
 |        or
 |      
 |      >>> auth = AWS4Auth(access_id, signing_key[, raise_invalid_date=False])
 |      
 |        or using auto-refreshed STS temporary creds via botocore RefreshableCredentials
 |        (useful for long-running processes):
 |      
 |      >>> auth = AWS4Auth(refreshable_credentials=botocore.session.Session().get_credentials(),
 |      ...                 region='eu-west-1', service='es')
 |      
 |      access_id   -- This is your AWS access ID
 |      secret_key  -- This is your AWS secret access key
 |      region      -- The region you're connecting to, as per the list at
 |                     http://docs.aws.amazon.com/general/latest/gr/rande.html#s3_region
 |                     e.g. us-east-1. For services which don't require a region
 |                     (e.g. IAM), use us-east-1.
 |                     Must be supplied as a keyword argument iff refreshable_credentials
 |                     is set.
 |      service     -- The name of the service you're connecting to, as per
 |                     endpoints at:
 |                     http://docs.aws.amazon.com/general/latest/gr/rande.html
 |                     e.g. elasticbeanstalk.
 |                     Must be supplied as a keyword argument iff refreshable_credentials
 |                     is set.
 |      date        -- Date this instance is valid for. 8-digit date as str of the
 |                     form YYYYMMDD. Key is only valid for requests with a
 |                     Date or X-Amz-Date header matching this date. If date is
 |                     not supplied the current date is used.
 |      signing_key -- An AWS4SigningKey instance.
 |      raise_invalid_date
 |                  -- Must be supplied as keyword argument. AWS4Auth tries to
 |                     parse a date from the X-Amz-Date and Date headers of the
 |                     request, first trying X-Amz-Date, and then Date if
 |                     X-Amz-Date is not present or is in an unrecognised
 |                     format. If one or both of the two headers are present
 |                     yet neither are in a format which AWS4Auth recognises
 |                     then it will remove both headers and replace with a new
 |                     X-Amz-Date header using the current date.
 |      
 |                     If this behaviour is not wanted, set the
 |                     raise_invalid_date keyword argument to True, and
 |                     instead an InvalidDateError will be raised when neither
 |                     date is recognised. If neither header is present at all
 |                     then an X-Amz-Date header will still be added containing
 |                     the current date.
 |      
 |                     See the AWS4Auth class docstring for supported date
 |                     formats.
 |      session_token
 |                  -- Must be supplied as keyword argument. If session_token
 |                     is set, then it is used for the x-amz-security-token
 |                     header, for use with STS temporary credentials.
 |      refreshable_credentials
 |                  -- A botocore.credentials.RefreshableCredentials instance.
 |                     Must be supplied as keyword argument. This instance is
 |                     used to generate valid per-request static credentials,
 |                     without needing to re-generate the AWS4Auth instance.                       
 |                     If refreshable_credentials is set, the following arguments
 |                     are ignored: access_id, secret_key, signing_key,
 |                     session_token.
 |  
 |  amz_cano_path(self, path)
 |      Generate the canonical path as per AWS4 auth requirements.
 |      
 |      Not documented anywhere, determined from aws4_testsuite examples,
 |      problem reports and testing against the live services.
 |      
 |      path -- request path
 |  
 |  get_canonical_request(self, req, cano_headers, signed_headers)
 |      Create the AWS authentication Canonical Request string.
 |      
 |      req            -- Requests/Httpx PreparedRequest object. Should already
 |                        include an x-amz-content-sha256 header
 |      cano_headers   -- Canonical Headers section of Canonical Request, as
 |                        returned by get_canonical_headers()
 |      signed_headers -- Signed Headers, as returned by
 |                        get_canonical_headers()
 |  
 |  handle_date_mismatch(self, req)
 |      Handle a request whose date doesn't match the signing key scope date.
 |      
 |      This AWS4Auth class implementation regenerates the signing key. See
 |      StrictAWS4Auth class if you would prefer an exception to be raised.
 |      
 |      req -- a requests prepared request object
 |  
 |  refresh_credentials(self)
 |  
 |  regenerate_signing_key(self, secret_key=None, region=None, service=None, date=None)
 |      Regenerate the signing key for this instance. Store the new key in
 |      signing_key property.
 |      
 |      Take scope elements of the new key from the equivalent properties
 |      (region, service, date) of the current AWS4Auth instance. Scope
 |      elements can be overridden for the new key by supplying arguments to
 |      this function. If overrides are supplied update the current AWS4Auth
 |      instance's equivalent properties to match the new values.
 |      
 |      If secret_key is not specified use the value of the secret_key property
 |      of the current AWS4Auth instance's signing key. If the existing signing
 |      key is not storing its secret key (i.e. store_secret_key was set to
 |      False at instantiation) then raise a NoSecretKeyError and do not
 |      regenerate the key. In order to regenerate a key which is not storing
 |      its secret key, secret_key must be supplied to this function.
 |      
 |      Use the value of the existing key's store_secret_key property when
 |      generating the new key. If there is no existing key, then default
 |      to setting store_secret_key to True for new key.
 |  
 |  ----------------------------------------------------------------------
 |  Class methods defined here:
 |  
 |  get_canonical_headers(req, include=None) from builtins.type
 |      Generate the Canonical Headers section of the Canonical Request.
 |      
 |      Return the Canonical Headers and the Signed Headers strs as a tuple
 |      (canonical_headers, signed_headers).
 |      
 |      req     -- Requests PreparedRequest object
 |      include -- List of headers to include in the canonical and signed
 |                 headers. It's primarily included to allow testing against
 |                 specific examples from Amazon. If omitted or None it
 |                 includes host, content-type and any header starting 'x-amz-'
 |                 except for x-amz-client context, which appears to break
 |                 mobile analytics auth if included. Except for the
 |                 x-amz-client-context exclusion these defaults are per the
 |                 AWS documentation.
 |  
 |  get_request_date(req) from builtins.type
 |      Try to pull a date from the request by looking first at the
 |      x-amz-date header, and if that's not present then the Date header.
 |      
 |      Return a datetime.date object, or None if neither date header
 |      is found or is in a recognisable format.
 |      
 |      req -- a requests PreparedRequest object
 |  
 |  ----------------------------------------------------------------------
 |  Static methods defined here:
 |  
 |  amz_cano_querystring(qs)
 |      Parse and format querystring as per AWS4 auth requirements.
 |      
 |      Perform percent quoting as needed.
 |      
 |      qs -- querystring
 |  
 |  amz_norm_whitespace(text)
 |      Replace runs of whitespace with a single space.
 |      
 |      Ignore text enclosed in quotes.
 |  
 |  encode_body(req)
 |      Encode body of request to bytes and update content-type if required.
 |      
 |      If the body of req is Unicode then encode to the charset found in
 |      content-type header if present, otherwise UTF-8, or ASCII if
 |      content-type is application/x-www-form-urlencoded. If encoding to UTF-8
 |      then add charset to content-type. Modifies req directly, does not
 |      return a modified copy.
 |      
 |      req -- Requests PreparedRequest object
 |  
 |  get_sig_string(req, cano_req, scope)
 |      Generate the AWS4 auth string to sign for the request.
 |      
 |      req      -- Requests PreparedRequest object. This should already
 |                  include an x-amz-date header.
 |      cano_req -- The Canonical Request, as returned by
 |                  get_canonical_request()
 |  
 |  parse_date(date_str)
 |      Check if date_str is in a recognised format and return an ISO
 |      yyyy-mm-dd format version if so. Raise DateFormatError if not.
 |      
 |      Recognised formats are:
 |      * RFC 7231 (e.g. Mon, 09 Sep 2011 23:36:00 GMT)
 |      * RFC 850 (e.g. Sunday, 06-Nov-94 08:49:37 GMT)
 |      * C time (e.g. Wed Dec 4 00:00:00 2002)
 |      * Amz-Date format (e.g. 20090325T010101Z)
 |      * ISO 8601 / RFC 3339 (e.g. 2009-03-25T10:11:12.13-01:00)
 |      
 |      date_str -- Str containing a date and optional time
 |  
 |  ----------------------------------------------------------------------
 |  Data and other attributes defined here:
 |  
 |  default_include_headers = {'content-type', 'date', 'host', 'x-amz-*'}
 |  
 |  ----------------------------------------------------------------------
 |  Data descriptors inherited from requests.auth.AuthBase:
 |  
 |  __dict__
 |      dictionary for instance variables (if defined)
 |  
 |  __weakref__
 |      list of weak references to the object (if defined)
```