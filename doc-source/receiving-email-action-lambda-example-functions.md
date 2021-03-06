# Lambda Function Examples<a name="receiving-email-action-lambda-example-functions"></a>

This topic contains examples of Lambda functions that control mail flow\.

## Example 1: Drops Spam<a name="receiving-email-action-lambda-example-functions-1"></a>

This example stops processing messages that have at least one spam indicator\.

```
 1. exports.handler = function(event, context, callback) {
 2.     console.log('Spam filter');
 3.     
 4.     var sesNotification = event.Records[0].ses;
 5.     console.log("SES Notification:\n", JSON.stringify(sesNotification, null, 2));
 6.  
 7.     // Check if any spam check failed
 8.     if (sesNotification.receipt.spfVerdict.status === 'FAIL'
 9.             || sesNotification.receipt.dkimVerdict.status === 'FAIL'
10.             || sesNotification.receipt.spamVerdict.status === 'FAIL'
11.             || sesNotification.receipt.virusVerdict.status === 'FAIL') {
12.         console.log('Dropping spam');
13.         // Stop processing rule set, dropping message
14.         callback(null, {'disposition':'STOP_RULE_SET'});
15.     } else {
16.         callback(null, null);   
17.     }
18. };
```

## Example 2: Continues if Particular Header<a name="receiving-email-action-lambda-example-functions-2"></a>

This example continues processing the current rule only if the email contains a specific header value\.

```
 1. exports.handler = function(event, context, callback) {
 2.     console.log('Header matcher');
 3.  
 4.     var sesNotification = event.Records[0].ses;
 5.     console.log("SES Notification:\n", JSON.stringify(sesNotification, null, 2));
 6.     
 7.     // Iterate over the headers
 8.     for (var index in sesNotification.mail.headers) {
 9.         var header = sesNotification.mail.headers[index];
10.         
11.         // Examine the header values
12.         if (header.name === 'X-Header' && header.value === 'X-Value') {
13.             console.log('Found header with value.');
14.             callback(null, null);
15.             return;
16.         }
17.     }
18.     
19.     // Stop processing the rule if the header value wasn't found
20.     callback(null, {'disposition':'STOP_RULE'});    
21. };
```

## Example 3: Retrieves Email from Amazon S3<a name="receiving-email-action-lambda-example-functions-3"></a>

This example gets the raw email from Amazon S3 and processes it\.

**Note**  
You must first write the email to Amazon S3 using an S3 Action\.

```
 1. var AWS = require('aws-sdk');
 2. var s3 = new AWS.S3();
 3.  
 4. var bucketName = '<YOUR BUCKET GOES HERE>';
 5.  
 6. exports.handler = function(event, context, callback) {
 7.     console.log('Process email');
 8.  
 9.     var sesNotification = event.Records[0].ses;
10.     console.log("SES Notification:\n", JSON.stringify(sesNotification, null, 2));
11.     
12.     // Retrieve the email from your bucket
13.     s3.getObject({
14.             Bucket: bucketName,
15.             Key: sesNotification.mail.messageId
16.         }, function(err, data) {
17.             if (err) {
18.                 console.log(err, err.stack);
19.                 callback(err);
20.             } else {
21.                 console.log("Raw email:\n" + data.Body);
22.                 
23.                 // Custom email processing goes here
24.                 
25.                 callback(null, null);
26.             }
27.         });
28. };
```

## Example 4: Bounces Messages that Fail DMARC Authentication<a name="receiving-email-action-lambda-example-functions-4"></a>

This examples sends a bounce message if an incoming email fails DMARC authentication\.

**Note**  
When using this example, set the value of the `emailDomain` environment variable to your email receiving domain\.

```
 1. 'use strict';
 2. 
 3. const AWS = require('aws-sdk');
 4. 
 5. // Assign the emailDomain environment variable to a constant.
 6. const emailDomain = process.env.emailDomain;
 7. 
 8. exports.handler = (event, context, callback) => {
 9.     console.log('Spam filter starting');
10. 
11.     const sesNotification = event.Records[0].ses;
12.     const messageId = sesNotification.mail.messageId;
13.     const receipt = sesNotification.receipt;
14. 
15.     console.log('Processing message:', messageId);
16. 
17.     // If DMARC verdict is FAIL and the sending domain's policy is REJECT
18.     // (p=reject), bounce the email.
19.     if (receipt.dmarcVerdict.status === 'FAIL' 
20.         && receipt.dmarcPolicy.status === 'REJECT') {
21.         // The values that make up the body of the bounce message.
22.         const sendBounceParams = {
23.             BounceSender: `mailer-daemon@${emailDomain}`,
24.             OriginalMessageId: messageId,
25.             MessageDsn: {
26.                 ReportingMta: `dns; ${emailDomain}`,
27.                 ArrivalDate: new Date(),
28.                 ExtensionFields: [],
29.             },
30.             // Include custom text explaining why the email was bounced.
31.             Explanation: "Unauthenticated email is not accepted due to the sending domain's DMARC policy.",
32.             BouncedRecipientInfoList: receipt.recipients.map((recipient) => ({
33.                 Recipient: recipient,
34.                 // Bounce with 550 5.6.1 Message content rejected
35.                 BounceType: 'ContentRejected',
36.             })),
37.         };
38. 
39.         console.log('Bouncing message with parameters:');
40.         console.log(JSON.stringify(sendBounceParams, null, 2));
41.         // Try to send the bounce. 
42.         new AWS.SES().sendBounce(sendBounceParams, (err, data) => {
43.             // If something goes wrong, log the issue.
44.             if (err) {
45.                 console.log(`An error occurred while sending bounce for message: ${messageId}`, err);
46.                 callback(err);
47.             // Otherwise, log the message ID for the bounce email.
48.             } else {
49.                 console.log(`Bounce for message ${messageId} sent, bounce message ID: ${data.MessageId}`);
50.                 // Stop processing additional receipt rules in the rule set.
51.                 callback(null, {
52.                     disposition: 'stop_rule_set',
53.                 });
54.             }
55.         });
56.     // If the DMARC verdict is anything else (PASS, QUARANTINE or GRAY), accept
57.     // the message and process remaining receipt rules in the rule set.
58.     } else {
59.         console.log('Accepting message:', messageId);
60.         callback();
61.     }
62. };
```