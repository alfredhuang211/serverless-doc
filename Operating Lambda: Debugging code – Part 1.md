# Operating Lambda: Debugging code – Part 1

by [James Beswick](https://aws.amazon.com/blogs/compute/author/jbeswick/) | on 08 MAR 2021 | in [AWS Lambda](https://aws.amazon.com/blogs/compute/category/compute/aws-lambda/), [Best Practices](https://aws.amazon.com/blogs/compute/category/post-types/best-practices/), [Serverless](https://aws.amazon.com/blogs/compute/category/serverless/) | [Permalink](https://aws.amazon.com/blogs/compute/operating-lambda-debugging-code-part-1/) | [ Comments](https://commenting.awsblogs.com/embed.html?disqus_shortname=aws-compute&disqus_identifier=13440&disqus_title=Operating+Lambda%3A+Debugging+code+–+Part+1&disqus_url=https://aws.amazon.com/blogs/compute/operating-lambda-debugging-code-part-1/) | [ Share](https://aws.amazon.com/cn/blogs/compute/operating-lambda-debugging-code-part-1/#)

In the *Operating Lambda* series, I cover important topics for developers, architects, and systems administrators who are managing [AWS Lambda](https://aws.amazon.com/lambda/)-based applications. This three-part series discusses core debugging concepts for Lambda-based applications.

Serverless applications are distributed applications. Debugging distributed applications is different to debugging single-server or monolithic applications. Specifically, you must consider:

- **Debugging across multiple services**, since most production serverless applications use a combination of Lambda functions and other services.
- **Debugging concurrent invocations** of functions, as many workloads use parallelization to process increases in traffic.
- **Understanding the state of a workload** when the error occurred so you can reliably reproduce the issue.

This post explores standardizing a debugging approach from an error occurring, general types of error, and troubleshooting errors arising from Lambda payloads.

## Standardizing a debugging approach

Once a Lambda-based application is deployed, monitor and capture errors as they occur and then start the debugging process. This section introduces a general approach to debugging Lambda-based applications. You can use this to identify the causes of errors and then use the learnings to make your workloads more resilient.

[![Debugging process](https://d2908q01vomqb2.cloudfront.net/1b6453892473a467d07372d45eb05abc2031647a/2021/03/02/debugging1.png)](https://d2908q01vomqb2.cloudfront.net/1b6453892473a467d07372d45eb05abc2031647a/2021/03/02/debugging1.png)

The first step is to **observe an error**. By monitoring a workload effectively, you can identify abnormal behavior in the system. These anomalies may be seen as:

- Lambda function errors showing a problem in completing an invocation.
- CloudWatch alarms and other monitoring tools that are either triggered due to exceptional values or other operational anomalies.
- Changes in expected activity, like unexpected drops in traffic, or large increases in load.
- Issue reports from end users that have not been otherwise identified in the infrastructure.
- Issues in third-party services, such as payment systems, that are impacting your application.

Next, **identify the point of failure** in the application architecture. By using an effective approach to monitoring a workload, you can quickly isolate the cause of the problem. If it is related to a Lambda function, it’s important to find the event source, the event, and the Lambda function processing the event.

**Debug the failure** by first classifying the type of error, such as lack of permissions, lack of capacity, a downstream outage, or a business logic error. The course of action taken here depends upon the type of failure identified. In the event of a business logic error in the Lambda function code, you can debug using the event related to the error. This post explores some typical causes of errors in Lambda functions and how you can identify and resolve these.

Finally, **remediate the problem**. This may include identifying causes of change in the infrastructure and taking action to prevent such changes in the future. Alternatively, it may be a code change requiring a new version of a Lambda function. In any case, the remediation should be documented, and you should ensure that monitoring processes are updated if necessary. If an error is reported by end users but otherwise not identified by your infrastructure, you may be able to improve error-handling logic in the front-end and backend APIs to alert you when these errors occur. You should aim to collect as much detail about the error as possible, including stack traces.

## General types of error

**Syntax or type errors** are generally caught by compilers or surface in log files when errors occur at runtime. Most errors in this class can be caught when writing or testing the code locally or during tests in the build process. Depending upon the runtime, an error of this type usually results in a clear error message identifying the line number of code where the problem occurred.

**Typos** may pass undetected by compilers and may run without errors during some invocations, depending upon the input. The cause of the error may be as simple as missing parentheses in equations or passing parameters in an incorrect order. Using human-readable variable names, visually inspecting code, and reviewing code with another developer can help identify these issues.

**Implementation errors** can occur when the overall logic of the application is correct but a specific task or data structure manipulation is handled incorrectly. For example, a Lambda function may process an event payload and incorrectly assume that there is always only one event in an array. As a result, the implementation fails to process part of the payload correctly.

**Logic errors** are a broad category of bug that can cause a Lambda function to operate incorrectly but not throw an error. The most common symptom of this type of problem is the Lambda function produces the wrong result. For example, an [off-by-one error](https://en.wikipedia.org/wiki/Off-by-one_error) may result in a loop iterating too few or too many times. This type of error can be mitigated by writing tests for your code that use a sufficient range of likely input parameters.

## Troubleshooting payloads

Once an error has occurred and you have narrowed down the problem to a specific Lambda function, there are a number of likely causes. This section explains the most common causes, their symptoms, and corrective actions that you can take.

### Unexpected event payloads

All Lambda functions receive an event payload in the first parameter of the handler. In many functions, this is the single greatest variable in the operation of the code. The event payload is a JSON structure that may contain arrays and nested elements.

Malformed JSON can occur when provided by upstream services that do not use a robust process for checking their JSON structures. This occurs when services concatenate text strings or embed user input that has not been sanitized. JSON is also frequently serialized for passing between services. Always parse JSON structures both as the producer and consumer of JSON to ensure that the structure is valid.

Similarly, failing to check for ranges of values in the event payload can result in errors. This example shows a function that calculates a tax withholding:

```js
exports.handler = async (event) => {    
    let pct = event.taxPct
    let salary = event.salary

    // Calculate % of paycheck for taxes
    return (salary * pct)
}
```

JavaScript

This function uses a salary and tax rate from the *event* payload to perform the calculation. However, the code fails to check if the attributes are present. It also fails to check data types, or ensure boundaries, such as ensuring that the tax percentage is between 0 and 1. As a result, values outside of these bounds produce nonsensical results. An incorrect type or missing attribute causes a runtime error.

You should create tests to ensure that your function handles larger payload sizes. The maximum size for a Lambda event payload is 256 KB. Depending upon the content, larger payloads may mean more items passed to the function or more binary data embedded in a JSON attribute. In both cases, this can result in more processing for a Lambda function.

Larger payloads can also cause timeouts. For example, a Lambda function processes one record per 100 ms and has a timeout of 3 seconds. Processing is successful for 0-29 items in the payload. However, once the payload contains more than 30 items, the function times out and throws an error. To avoid this, ensure that timeouts are set to handle the additional processing time for the maximum number of items expected.

### Unexpectedly large payload sizes

Many event payloads contain pointers to other resources. With [Amazon S3](https://aws.amazon.com/s3/) events, while the event payload is a JSON object, the S3 object that caused the event may be [up to 5 terabytes](https://aws.amazon.com/s3/faqs/#:~:text=Individual Amazon S3 objects can,using the Multipart Upload capability.) in size. If a function performs processing on a referenced data item such as an S3 object, it should first check the size of the data.

For example, a Lambda function with 128 MB of memory may convert a JPG file stored as an object in S3, by using an image-processing library. The function works as expected with smaller image files. However, when a larger JPG file is provided as input, the Lambda function throws an error due to running out of memory. To avoid this, the test cases should include examples from the upper bounds of expected data sizes and the code should also validate payload sizes.

### Incorrectly processing payload parameters

While JSON is a flexible notation, services that generate JSON may use fields in specific ways. For AWS services that generate events, the message structure is documented to show the format of data attributes and whether the attribute is required. Check the documentation from the producing service to ensure that you are processing the JSON attributes correctly. If you are producing JSON, applying a [style guide](https://google.github.io/styleguide/jsoncstyleguide.xml) and creating documentation can help developers who are consuming your events.

For example, for [events generated by S3](https://docs.aws.amazon.com/AmazonS3/latest/dev/notification-content-structure.html), the *s3.object.key* attribute contains a URL encoded object key name. Many functions process this attribute as text to load the referenced S3 object:

```js
const originalText = await s3.getObject({
  Bucket: event.Records[0].s3.bucket.name,
  Key: event.Records[0].s3.object.key
}).promise()
```

JavaScript

This code works with the key name “james.jpg” but throws a *NoSuchKey* error when the name is “james beswick.jpg”. Since [URL encoding](https://www.w3schools.com/tags/ref_urlencode.ASP) converts spaces and other characters in a key name, you must ensure that functions decode keys before using this data:

```js
const originalText = await s3.getObject({
  Bucket: event.Records[0].s3.bucket.name,
  Key: decodeURIComponent(event.Records[0].s3.object.key.replace(/\+/g, " "))
}).promise()
```

JavaScript

## Conclusion

Debugging serverless applications is different to debugging single-server or monolithic applications. You must consider debugging across multiple invocations and services, and understand the state of a distributed workload.

This post explains a general approach to standardizing debugging in Lambda functions and discusses general types of error that can occur. It also looks at troubleshooting Lambda payloads, from unexpected events to unexpected event sizes causing runtime issues.

[Part 2](https://aws.amazon.com/blogs/compute/operating-lambda-debugging-configurations-part-2/) explores errors that can arise from Lambda function configurations.

For more serverless learning resources, visit [Serverless Land](https://serverlessland.com/).

TAGS: [serverless](https://aws.amazon.com/blogs/compute/tag/serverless/)