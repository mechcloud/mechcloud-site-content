# Real time sync of AWS cloud assets

## Overall setup
![AWS assets real time sync](https://raw.githubusercontent.com/mechcloud/mechcloud-site-content/master/images/mechcloud/turbine/design/real-time-sync-aws.svg)

## Sync vs discovery
### Sync
* Using sync a cloud provider notifies Turbine+ whenever a cloud asset is created, modified or terminated.
* Sync is triggered by a cloud provider and it's meant for incremental updates ONLY.
* It is unidirectional (cloud provider to Turbine+) at this (v2.0.0) moment.
* In order to setup sync, it should be manually enabled for a cloud account from cloud account context menu item `Enable Sync` after following steps required on cloud provider side (described in a later section below).

![Enable Sync](https://raw.githubusercontent.com/mechcloud/mechcloud-site-content/master/images/mechcloud/turbine/screenshots/enable-sync.png)

### Discovery 
* Discovery is initiated by a user from Turbine+ after a new cloud account is added to Turbine so that metadata of existing cloud services/assets can be loaded into Turbine+ database from cloud provider.
* Discovery will be fetching metadata of all the existing services/assets from cloud provider no matter some or all of these assets are already loaded into Turbine+ database or not.
* In addition to running discovery after adding a cloud account, it can also be run in a situation when sync fails for some reason or it is not enabled on an account.

## Supported resource types
* At this moment, only vpc, subnet and virtual server are supported for sync.

## Steps to be followed for a region
* These steps should be followed for every region.
* Update region code (`us-east-1`) as per target region in the following steps.

### SQS Queue
* Create an sqs queue with following details -
  - Only few details are mentioned here. Other details can be entered/selected as per need.
  - **Details**
    - Name : management-events-us-east-1.fifo 
    - Type : FIFO
  - **Configuration**
    - Make sure `Content-based deduplication` is enabled.


### Rules
#### management-events-tags
* Create an eventbridge rule with following details -
  - Only few details are mentioned here. Other details can be entered/selected as per need.
  - **Name and description**
    - Name : management-events-tags (or any other suitable name)
  - **Define pattern**
    - Type : Event pattern
    - Event matching pattern : Pre-defined pattern by service
    - Service provider : AWS
    - Service Name : Tags
    - Event type : Tag Change on Resource
  - For event bus, select default event bus.
  - Select `SQS queue` as target service, `management-events-us-east-1.fifo` under queue and enter `management-events-tags` in `Message group ID` field.
  - Other values can be left unchanged.
#### management-events-ec2
* Create an eventbridge rule with following details -
  - Only few details are mentioned here. Other details can be entered/selected as per need.
  - **Name and description**
    - Name : management-events-ec2 (or any other suitable name)
  - **Define pattern**
    - Type : Event pattern
    - Event matching pattern : Pre-defined pattern by service
    - Service provider : AWS
    - Service Name : EC2
    - Event type : EC2 Instance State-change Notification
  - For event bus, select default event bus.
  - Select `SQS queue` as target service, `management-events-us-east-1.fifo` under queue and enter `ec2-instances-events` in `Message group ID` field.
  - Other values can be left unchanged.

## Python script
```python
import boto3, json, utils, redis

region_name = 'us-east-1'

def publish_to_redis(msg):
    account_number = boto3.client('sts').get_caller_identity().get('Account')
    topic_name = 'topic-account-management-events-{}'.format(account_number)
    
    redis_client = redis.StrictRedis(host='mongo-redis-host', port=6379, decode_responses=True)
    redis_client.publish(topic_name, json.dumps(msg))

# Create SQS client
sqs_client = boto3.client('sqs', region_name=region_name)

response = sqs_client.get_queue_url(
        QueueName='management-events-{}.fifo'.format(region_name),
)
queue_url = response["QueueUrl"]

while True:
    print('Waiting for message from queue ..')
    
    # Receive message from SQS queue
    response = sqs_client.receive_message(
        QueueUrl=queue_url,
        MaxNumberOfMessages=1,
        MessageAttributeNames=[
            'All'
        ]
    )
    
    if 'Messages' in response:
        msg = response['Messages'][0]
        msg_body = json.loads(msg['Body'])
        print('Message body : ' + utils.pretty_print_dict(msg_body))
        receipt_handle = msg['ReceiptHandle']
        
        # publish to redis
        publish_to_redis(msg_body)
        print('Message published to redis.')
        
        # Delete received message from queue
        sqs_client.delete_message(
            QueueUrl=queue_url,
            ReceiptHandle=receipt_handle
        )
        print('Message deleted.')
```