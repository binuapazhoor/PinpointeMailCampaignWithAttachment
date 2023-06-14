# PinpointeMailCampaignWithAttachment

Amazon Pinpoint currently doesn't support attachments when sending emails via Campaigns or Journeys. Customers have to use the Amazon SES SendRawMessage API operation to attach files, which lacks features such as customer segmentation and scheduling.

Email attachments are key to many use cases, some of them are:

    Monthly bills (specific to the recipient)
    New terms & conditions (same for all)
    Contracts (specific to the recipient)
    Booking confirmation (specific to the recipient)
    e-Tickets (specific to the recipient)

This solution enables marketers to design and schedule Amazon Pinpoint journeys with attachments or pre-signed Amazon S3 URLs without the support of technical resources.

High Level Architecture

This is the solution utilizes Amazon Pinpoint custom channel (AWS Lambda function) to support email attachments for Pinpoint Journeys. The AWS Lambda function calls Pinpoint & 
SES API operation for sending emails. The attached file is stored in an S3 bucket and can be either attached to the email send or accessed via an Amazon S3 pre-signed URL.

Marketers can specify the email template, friendly sender name, sender address, attachment and attachment type using Pinpoint custom channel's Custom data input field. Data 
inserted in that field will be accessible by the AWS Lambda function, which will process accordingly.

![image](https://github.com/binuapazhoor/PinpointeMailCampaignWithAttachment/assets/58440253/48d5b759-61bb-47f8-9812-737de48e5c3a)

Attahced files mechanism:

Attachment per recipient: Each recipient (endpoint) receives a different file e.g. monthly bill specific to them. The files stored in S3 should have the following naming
convention prefix_endpointid.file e.g. OctoberBill_111.pdf. The file prefix can be specified in the AWS Lambda Custom data when building a journey. The AWS Lambda function 
builds the S3 object key (file name) by concatenating the file prefix, endpoint id and file type. To select this method, specify ONEPER in the Pinpoint journey Custom data field.
Attachment per journey The attachment file name should be the same as the Pinpoint journey Custom data file prefix, the AWS Lambda will concatenate the file prefix 
and file type to form the S3 object key. To select this method, specify ONEALL in the Pinpoint journey Custom data.
    
Pre-signed S3 link: This solution uses the Substitution component of the Amazon Pinpoint SendMessages API operation to dynamically populate the message template with the 
S3 pre-signed URL that is passed as a parameter in the request body. Marketers can choose where to place the link in the email template by simply placing {URL} in the Amazon 
Pinpoint HTML template as part of the href tag as displayed here: </a href="{URL}">Download the file<//a>. The solution uses {URL} but this can be changed in the code depending 
the requirements.

Sending mechanism for file attachments:

Attachment per recipient: To attach and send one file per recipient the solution calls the Amazon S3 GetObject API operation per endpoint id to obtain the S3 object and the 
Amazon SES SendRawEmail API operation to send the email with the attachment. To follow that approach specify ONEPER in the Pinpoint journey Custom data.
Attachment per journey: To attach the same file for all the recipients the solution calls only once per AWS Lambda invokation the Amazon S3 GetObject API operation, 
creates a list of all the endpoints (max 50) and then calls the Amazon SES SendRawEmail API operation once to send the email with the attachment.To follow that approach 
specify ONEALL in the Pinpoint journey Custom data.

Considerations

Events' attribution: Engagement events from the emails sent won't be attributed automatically back to the journey. These events can still be accessed and analysed if streamed 
using Amazon Kinesis Firehose. To reconsile the events back to the journey, the solution includes the journey id as trace_id when sent via the SendMessage Pinpoint API operation 
and as a tag JourneyId when sent via SES SendRawEmail.
No email personalization when attaching a file: Emails with attached files sent via the SES SendRawEmail API operation won't support message helpers for personalisation. 
The message template is expected to not have any user or endpoint attributes.
    
Custom data: The order and spelling of the AWS Lambda Custom data field are key for this solution to function as expected. Each variable is expected to be in the a specific 
place and some of them need to be spelled as per the instructions in this GitHub repository e.g. NO, ONEPER, ONEALL and NA. If a Pinpoint journey gets published with the wrong 
Custom data then the emails might not be send and it will need to get duplicated and published again.

