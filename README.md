# aws-lightweight-client-java
<a href="https://github.com/davidmoten/aws-lightweight-client-java/actions/workflows/ci.yml"><img src="https://github.com/davidmoten/aws-lightweight-client-java/actions/workflows/ci.yml/badge.svg"/></a><br/>
[![codecov](https://codecov.io/gh/amsa-code/aws-lightweight-client-java/branch/master/graph/badge.svg)](https://codecov.io/gh/amsa-code/aws-lightweight-client-java)<br/>
[![Maven Central](https://maven-badges.herokuapp.com/maven-central/com.github.davidmoten/aws-lightweight-client-java/badge.svg?style=flat)](https://maven-badges.herokuapp.com/maven-central/com.github.davidmoten/aws-lightweight-client-java)<br/>

This is a really lightweight standalone artifact (about 56K) that performs authentication (signing requests with AWS Signature Version 4) and helps you build requests against the AWS API. It includes nice concise builders, a lightweight inbuilt xml parser (to parse responses), and useful convenience methods. 

For example with the 56K standalone artifact you can do:

```java
Client s3 = Client.s3()
  .regionName("ap-southeast-2")
  .accessKey(accessKey)
  .secretKey(secretKey)
  .build();

String content = s3
  .path("myBucket/myObject.txt")
  .responseAsUtf8();
```
This is actually a lot more concise than using the AWS SDK for Java but moreover because the artifact is small and the number of classes loaded to perform the action is much less, the *cold start* time for a Java AWS Lambda that uses s3 is **reduced from 10s to 4s**! Sub-second cold start time latency would be great but the catch is that a lot of classes are loaded by the java platform to perform the https call to the S3 API so that's going to be hard to avoid. In fact my testing shows that without any https calls at all a lambda can cold start in <1s (but will have presumably pretty limited functionality)!

Aside from cold-start improvements in AWS Lambda, the small artifact size is presumably attractive also for Android developers. 

## Getting started
Add this dependency to your pom.xml:

```xml
<dependency>
    <groupId>com.github.davidmoten</groupId>
    <artifactId>aws-lightweight-client-java</artifactId>
    <version>VERSION_HERE</version>
</dependency>
```

## Usage

To perform actions against the API you do need to know what methods exist and the parameters for those methods. This library is lightweight because it doesn't include a mass of generated classes from the API so you'll need to check the AWS API documentation to get that information. For example the API docs for S3 is [here](https://docs.aws.amazon.com/AmazonS3/latest/API/Welcome.html).

### Creating a Client
In a Lambda handler environment variables hold the credentials and session token. To pick those values up:

```java
Client s3 = Client.s3().defaultClient().build();
```
Outside of lambda you might specify your credentials explicitly:

```java
Client s3 = Client
  .s3()
  .regionName("ap-southeast-2")
  .accessKey(accessKey)
  .secretKey(secretKey)
  .build()
```
There are a number of other options that can be set when building the Client:

```java
Client iam = Client
    .serviceName("iam") 
    .regionName(regionName) 
    .accessKey(accessKey)
    .secretKey(secretKey)
    .exceptionFactory(myExceptionFactory)
    .exception(
	    x -> !x.isOk() && x.contentUtf8().contains("NonExistentPolicy"),
	    x -> new PolicyDoesNotExistException(x.contentUtf8()))
    .httpClient(myHttpClient) 
    .connectTimeoutMs(30000)
    .readTimeoutMs(120000)
    .build();
```
A client can be copied from another client to pick up same configuration (but with a different service name):

```java
Client sqs = Client.from(iam).build();
```
### Presigned URLs
Presigned URLs are generated as follows (with a specified expiry duration):

```java
String presignedUrl = 
     s3
     .path(bucketName + "/" + objectName) 
     .presignedUrl(1, TimeUnit.DAYS));
```

### S3
The code below demonstrates the following:
* create bucket
* put object with metadata
* read object and metadata
* list objects in bucket
* delete object
* delete bucket

```java
// we'll create a random bucket name
String bucketName = "temp-bucket-" + System.currentTimeMillis();

///////////////////////
// create bucket
///////////////////////

String createXml = "<?xml version=\"1.0\" encoding=\"UTF-8\"?>\n"
        + "<CreateBucketConfiguration xmlns=\"http://s3.amazonaws.com/doc/2006-03-01/\">\n"
        + "   <LocationConstraint>" + regionName + "</LocationConstraint>\n"
        + "</CreateBucketConfiguration>";
s3.path(bucketName)
    .method(HttpMethod.PUT)
    .requestBody(createXml)
    .execute();

////////////////////////////
// put object with metadata
///////////////////////////

String objectName = "ExampleObject.txt";
s3
    .path(bucketName, objectName)
    .method(HttpMethod.PUT)
    .requestBody("hi there")
    .metadata("category", "something")
    .execute();

///////////////////////////////////
// read object including metadata
///////////////////////////////////

String text = s3
    .path(bucketName + "/" + objectName)
    .responseAsUtf8();

///////////////////////////////////
// read object
///////////////////////////////////

Response r = s3
    .path(bucketName, objectName)
    .response();
System.out.println("response ok=" + response.isOk());
System.out.println(r.content().length + " chars read");
System.out.println("category=" + r.metadata("category").orElse(""));

///////////////////////////////////
// list bucket objects 
///////////////////////////////////

List<String> keys = s3
    .url("https://" + bucketName + ".s3." + regionName + ".amazonaws.com")
    .query("list-type", "2")
    .responseAsXml()
    .childrenWithName("Contents")
    .stream()
    .map(x -> x.content("Key"))
    .collect(Collectors.toList());
System.out.println(keys);

///////////////////////////////////
// delete object 
///////////////////////////////////

s3.path(bucketName, objectName) 
    .method(HttpMethod.DELETE) 
    .execute();
        
///////////////////////////////////
// delete bucket 
///////////////////////////////////

s3.path(bucketName) 
	.method(HttpMethod.DELETE) 
	.execute();
```

### SQS
Here are some SQS tasks:

* create an sqs queue
* get the queue url
* place two messages on the queue
* read all messages of the queue and mark them as read
* delete the sqs queue

You'll note that most of the interactions with sqs involve using the url of the queue rather than the base service endpoint (`http://sqs.amazonaws.com`).

```java
String queueName = "MyQueue-" + System.currentTimeMillis();

///////////////////////////////////
// create queue
///////////////////////////////////

sqs.query("Action", "CreateQueue") 
    .query("QueueName", queueName) 
    .execute();

///////////////////////////////////
// get queue url
///////////////////////////////////

String queueUrl = sqs 
    .query("Action", "GetQueueUrl") 
    .query("QueueName", queueName) 
    .responseAsXml() 
    .content("GetQueueUrlResult", "QueueUrl");

///////////////////////////////////
// send a message
///////////////////////////////////

sqs.url(queueUrl) 
    .query("Action", "SendMessage") 
    .query("MessageBody", "hi there") 
    .execute();

///////////////////////////////////
// read all messages
///////////////////////////////////

List<XmlElement> list;
do {
    list = sqs.url(queueUrl)
        .query("Action", "ReceiveMessage")
        .responseAsXml()
        .child("ReceiveMessageResult")
        .children();

    list.forEach(x -> {
	    String msg = x.child("Body").content();
	    System.out.println(msg);
	    // mark message as read
	    sqs.url(queueUrl)
	            .query("Action", "DeleteMessage")
	            .query("ReceiptHandle", x.child("ReceiptHandle").content())
	            .execute();
    });
} while (!list.isEmpty());

///////////////////////////////////
// delete queue
///////////////////////////////////

sqs.url(queueUrl) 
    .query("Action", "DeleteQueue") 
    .execute();
```

### Error handling
Let's look at a simple one, reading an object in an S3 bucket.

```java
String text = s3
    .path(bucketName + "/" + objectName)
    .responseAsUtf8();
```
If the object does not exist an exception will be thrown like this:
```
com.github.davidmoten.aws.lw.client.ServiceException: statusCode=404: <?xml version="1.0" encoding="UTF-8"?>
<Error><Code>NoSuchKey</Code><Message>The specified key does not exist.</Message><Key>not-there</Key><RequestId>1TVAXX4VF5DYHJJH</RequestId><HostId>VrvGCPhExKbjuONSuX/LGw0mYSndjg3t26LNAQCKTL/i5U+cZfYa4ow3KQ1tpJdQuMH9sB4JTUk=</HostId></Error>
	at com.github.davidmoten.aws.lw.client.internal.ExceptionFactoryDefault.create(ExceptionFactoryDefault.java:17)
	at com.github.davidmoten.aws.lw.client.Request.responseAsBytes(Request.java:140)
	at com.github.davidmoten.aws.lw.client.Request.responseAsUtf8(Request.java:153)
	at com.github.davidmoten.aws.lw.client.ClientMain.main(ClientMain.java:48)
```

You can see that the AWS exception message (in xml format) is present in the error message and can be used to check for standard codes. If you were using the full AWS SDK library then it would throw a `NoSuchKeyException`. In our case we check for the presence of `NoSuchKey` in the error message.

The code below does not throw an exception when the object does not exist. However, `response.isOk()` returns false:

```java
Response r = s3
    .path(bucketName + "/" + objectName)
    .response();
System.out.println("ok=" + r.isOk() + ", statusCode=" + r.statusCode() + ", message=" + r.contentUtf8());
```

The output is:
```
ok=false, statusCode=404, message=<?xml version="1.0" encoding="UTF-8"?>
<Error><Code>NoSuchKey</Code><Message>The specified key does not exist.</Message><Key>notThere</Key><RequestId>4AAX24QZ8777FA6B</RequestId><HostId>4N1rsMjjdM7tjKSQDXNQZNH8EOqNckUsO6gRVPfcjMmHZ9APRwYJwufZOr9l1Qlinux5W537bDc=</HostId></Error>
```
### Exception factory
You can define what exceptions get thrown using a builder method for a `Client`:

```java
Client sqs = Client 
    .sqs()
    .defaultClient()
    .exception(
            x -> !x.isOk() && x.contentUtf8().contains("NonExistentQueue"),
            x -> new QueueDoesNotExistException(x.contentUtf8()))
    .build()
```


