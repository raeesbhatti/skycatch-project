# Skycatch project
This is a evaluatory project for a job interview at Skycatch. 

## Architecture
The system is based on AWS S3, AWS DynamoDB and AWS Lambda. 

### Data extraction
The S3 bucket is configured as a trigger for `ImageProcessor` Lambda function. When
you upload a file to the bucket it triggers an S3 event. The Lambda function uses the
data in event to fetch the related files from S3. Then it does some basic validations
to make sure that it is a valid file. And then it extracts EXIF and XMP data and puts
the data into DynamoDB table using `etag` as primary key.

### Exporting as CSV
A Lambda function `CSVExporter` has the stream of DynamoDB table mentioned above
configure as a trigger. So, when the `ImageProcessor` function puts the data into
DynamoDB, `CSVExporter` scans the table, creates a CSV from that data and puts it
into the S3 bucket under the name `image-data.csv`. 

## Flaws
Following are some known flaws in `ImageProcessor` function:
* Library for parsing EXIF data is not as widely used compared to other libs in JS
landscape.
* Library for parsing XMP data is not as widely used compared to other libs in JS
landscape.
* We're throwing an error if the image has more than one instance XMP document
(opening and closing tags). But it is technically possible to have more than documents
in the latest specs.
* We're trying to get `string` value for most of the properties. Which results in
sub-optimal representations some times.
* There is some problem with XP (XP Comment and XP Author) tags representation in
EXIF data.
* There are no CloudWatch alarms configured for errors yet.

Following are some known flaws in `CSVExporter` function:
* It does a `Scan` of DynamoDB, which is resource intensive. I think it can be
configured to work off of Stream event data, and existing data from S3.
* A single `Scan` request can return 1MB of data at most, so, we're re-issuing them
with new parameters until there is no more data. That means that one error in any of
them could shut the function down, discarding all of the fetched data.
* There are no CloudWatch alarms configured for errors yet.
