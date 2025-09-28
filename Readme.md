# steps to create an access logs for ALB & send it to cloudwatch

1. Create a S3 bucket :
- Create a bucket for access log.
```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "AWSLogDeliveryWrite",
            "Effect": "Allow",
            "Principal": {
                "AWS": "arn:aws:iam::127311923021:root"
            },
            "Action": "s3:PutObject",
            "Resource": "arn:aws:s3:::nginx-alb-access-logs-v1/AWSLogs/697172517874/*",
            "Condition": {
                "StringEquals": {
                    "s3:x-amz-acl": "bucket-owner-full-control"
                }
            }
        },
        {
            "Sid": "AWSLogDeliveryAclCheck",
            "Effect": "Allow",
            "Principal": {
                "AWS": "arn:aws:iam::127311923021:root"
            },
            "Action": "s3:GetBucketAcl",
            "Resource": "arn:aws:s3:::nginx-alb-access-logs-v1"
        }
    ]
}
```
- Replace the S3 ARN & 127311923021 (this is value for us-east). More details refer : https://docs.aws.amazon.com/elasticloadbalancing/latest/application/enable-access-logging.html
2. Configure a Access log for ALB :
- Go to ALB Attributes >> Monitoring >> Access Log : Add the created bucket.
3. Create a Lambda Role:
- Assign then permission ```AWSLambdaBasicExecutionRole```.
- Add a policy to to connect to the s3 bucket:
```{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "s3:ListBucket"
            ],
            "Resource": [
                "arn:aws:s3:::nginx-alb-access-logs-v1"
            ]
        },
        {
            "Effect": "Allow",
            "Action": [
                "s3:GetObject",
                "s3:GetObjectVersion"
            ],
            "Resource": [
                "arn:aws:s3:::nginx-alb-access-logs-v1/*"
            ]
        }
    ]
}
```
- Replace the bucket names.
4. Create a Lambda function:
- ```vi lambda_function.py```
```import boto3
import gzip
import io
import json
import time

# Initializing S3 and CloudWatch Logs clients
s3_client = boto3.client('s3')
logs_client = boto3.client('logs')

# CloudWatch log group name
log_group_name = '/alb/live-access-logs'

# Create a log group if it doesn't exist
def create_log_group_if_not_exists(log_group_name):
    try:
        logs_client.create_log_group(logGroupName=log_group_name)
    except logs_client.exceptions.ResourceAlreadyExistsException:
        print(f"Log group '{log_group_name}' already exists.")

# Create a log stream
def create_log_stream(log_group_name, log_stream_name):
    try:
        logs_client.create_log_stream(
            logGroupName=log_group_name,
            logStreamName=log_stream_name
        )
        print(f"Log stream '{log_stream_name}' created.")
    except logs_client.exceptions.ResourceAlreadyExistsException:
        print(f"Log stream '{log_stream_name}' already exists.")

# Process log file from S3
def process_s3_object(bucket_name, key):
    s3_object = s3_client.get_object(Bucket=bucket_name, Key=key)
    log_data = s3_object['Body'].read()
    log_lines = []

    try:
        with gzip.GzipFile(fileobj=io.BytesIO(log_data)) as gzipfile:
            log_lines = gzipfile.read().decode('utf-8').splitlines()
    except OSError:
        log_lines = log_data.decode('utf-8').splitlines()

    return log_lines

# Main Lambda handler function
def lambda_handler(event, context):
    try:
        create_log_group_if_not_exists(log_group_name)
        
        # Retrieve S3 bucket and prefix from the event
        records = event.get('Records', [])
        if not records:
            raise ValueError("No 'Records' found in the event")

        s3_bucket = 'nginx-alb-access-logs-v1'
        prefix = 'AWSLogs/697172517874/elasticloadbalancing/us-east-1/2025/09/28/'
        # Continue listing S3 objects if there are more than 1,000 objects
        continuation_token = None
        while True:
            list_params = {
                'Bucket': s3_bucket,
                'Prefix': prefix
            }
            if continuation_token:
                list_params['ContinuationToken'] = continuation_token
            
            response = s3_client.list_objects_v2(**list_params)
            
            if 'Contents' not in response:
                print(f"No objects found in the bucket '{s3_bucket}' with prefix '{prefix}'")
                break

            for obj in response['Contents']:
                s3_key = obj['Key']
                print(f"Processing bucket: {s3_bucket}, key: {s3_key}")

                try:
                    log_lines = process_s3_object(s3_bucket, s3_key)

                    log_events = []
                    for line in log_lines:
                        log_events.append({
                            'timestamp': int(round(time.time() * 1000)),
                            'message': line
                        })

                    if log_events:
                        log_stream_name = f"log_stream-{int(time.time() // 86400)}"
                        create_log_stream(log_group_name, log_stream_name)
                        
                        # Upload batched log events to CloudWatch Logs
                        batch_size = 0
                        batch_events = []
                        for event in log_events:
                            event_size = len(json.dumps(event))
                            if batch_size + event_size > 1048576:
                                if batch_events:
                                    logs_client.put_log_events(
                                        logGroupName=log_group_name,
                                        logStreamName=log_stream_name,
                                        logEvents=batch_events
                                    )
                                    print(f"Uploaded batch of size {batch_size} bytes.")
                                batch_events = []
                                batch_size = 0
                            batch_events.append(event)
                            batch_size += event_size
                        
                        if batch_events:
                            logs_client.put_log_events(
                                logGroupName=log_group_name,
                                logStreamName=log_stream_name,
                                logEvents=batch_events
                            )
                            print(f"Uploaded final batch of size {batch_size} bytes.")

                except s3_client.exceptions.NoSuchKey:
                    print(f"Error: The key '{s3_key}' does not exist in bucket '{s3_bucket}'")
                    continue

            if response.get('IsTruncated'):
                continuation_token = response.get('NextContinuationToken')
            else:
                break

        return {
            'statusCode': 200,
            'body': json.dumps('Success')
        }

    except Exception as e:
        print(f"Exception: {str(e)}")
        return {
            'statusCode': 500,
            'body': json.dumps(f'Error: {str(e)}')
        }
```
- Replace the 'log_group_name=<log group name> & s3_bucket=<s3 name>'
- ```zip lambda_function.zip lambda_function.py```
- ```aws lambda create-function function-name alb-log-processor --zip-file fileb://lambda_function.zip --handler lambda_function.lambda_handler --runtime python3.11 --role arn:aws:iam::697172517874:role/test-execution-role```
- ``` aws lambda add-permission --function-name alb-log-processor --principal s3.amazonaws.com --statement-id s3invoke --action lambda:InvokeFunction --source-arn arn:aws:s3:::nginx-alb-access-logs-v1```
- ```vi s3_notification_config.json```
```
{
  "LambdaFunctionConfigurations": [
    {
      "LambdaFunctionArn": "arn:aws:lambda:us-east-1:697172517874:function:alb-log-processor",
      "Events": ["s3:ObjectCreated:*"],
      "Filter": {
        "Key": {
          "FilterRules": [
            {
              "Name": "prefix",
              "Value": "AWSLogs/697172517874/elasticloadbalancing/us-east-1/"
            },
            {
              "Name": "suffix",
              "Value": ".log.gz"
            }
          ]
        }
      }
    }
  ]
}
```
- ```aws s3api put-bucket-notification-configuration --bucket nginx-alb-access-logs-v1 --notification-configuration file://s3_notification_config.json```
