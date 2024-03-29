# How-to-Build-and-USe-glue-crawler-incremental-faashion-using-SQS-and-S3-events-
How to Build and USe glue crawler incremental faashion using SQS and S3 events 
![Capture](https://github.com/soumilshah1995/How-to-Build-and-USe-glue-crawler-incremental-faashion-using-SQS-and-S3-events-/assets/39345855/0b59894f-de45-4b3e-90c1-7bcd6075f88a)


# Code
```
try:
    import datetime
    import json
    import random
    import boto3
    import os
    import uuid
    import time
    from datetime import datetime
    from faker import Faker
    from dotenv import load_dotenv

    load_dotenv("../dev.env")
except Exception as e:
    pass



class AWSS3(object):
    """Helper class to which add functionality on top of boto3 """

    def __init__(self, bucket, aws_access_key_id, aws_secret_access_key, region_name):

        self.BucketName = bucket
        self.client = boto3.client(
            "s3",
            aws_access_key_id=aws_access_key_id,
            aws_secret_access_key=aws_secret_access_key,
            region_name=region_name,
        )

    def put_files(self, Response=None, Key=None):
        """
        Put the File on S3
        :return: Bool
        """
        try:

            response = self.client.put_object(
                ACL="private", Body=Response, Bucket=self.BucketName, Key=Key
            )
            return "ok"
        except Exception as e:
            print("Error : {} ".format(e))
            return "error"

    def item_exists(self, Key):
        """Given key check if the items exists on AWS S3 """
        try:
            response_new = self.client.get_object(Bucket=self.BucketName, Key=str(Key))
            return True
        except Exception as e:
            return False

    def get_item(self, Key):

        """Gets the Bytes Data from AWS S3 """

        try:
            response_new = self.client.get_object(Bucket=self.BucketName, Key=str(Key))
            return response_new["Body"].read()

        except Exception as e:
            print("Error :{}".format(e))
            return False

    def find_one_update(self, data=None, key=None):

        """
        This checks if Key is on S3 if it is return the data from s3
        else store on s3 and return it
        """

        flag = self.item_exists(Key=key)

        if flag:
            data = self.get_item(Key=key)
            return data

        else:
            self.put_files(Key=key, Response=data)
            return data

    def delete_object(self, Key):

        response = self.client.delete_object(Bucket=self.BucketName, Key=Key, )
        return response

    def get_all_keys(self, Prefix=""):

        """
        :param Prefix: Prefix string
        :return: Keys List
        """
        try:
            paginator = self.client.get_paginator("list_objects_v2")
            pages = paginator.paginate(Bucket=self.BucketName, Prefix=Prefix)

            tmp = []

            for page in pages:
                for obj in page["Contents"]:
                    tmp.append(obj["Key"])

            return tmp
        except Exception as e:
            return []

    def print_tree(self):
        keys = self.get_all_keys()
        for key in keys:
            print(key)
        return None

    def __repr__(self):
        return "AWS S3 Helper class "


global faker
global helper

faker = Faker()
helper = AWSS3(
    aws_access_key_id=os.getenv("DEV_ACCESS_KEY"),
    aws_secret_access_key=os.getenv("DEV_SECRET_KEY"),
    region_name=os.getenv("DEV_REGION"),
    bucket=os.getenv("BUCKET")
)


def run():
    for i in range(1, 5):
        order_id = uuid.uuid4().__str__()
        customer_id = uuid.uuid4().__str__()

        orders = {
            "orderid": order_id,
            "customer_id": customer_id,
            "ts": datetime.now().isoformat().__str__(),
            "order_value": random.randint(10, 1000).__str__(),
            "priority": random.choice(["LOW", "MEDIUM", "URGENT"])

        }
        print(orders)

        # Convert dictionary to JSON string and then to bytes
        order_data = json.dumps(orders).encode("utf-8")

        # # Pass the bytes data to the put_files method
        # helper.put_files(Response=order_data, Key=f'raw/orders/{uuid.uuid4().__str__()}.json')

        customers = {
            "customer_id": customer_id,
            "name": faker.name(),
            "state": faker.state(),
            "city": faker.city(),
            "email": faker.email(),
            "ts": datetime.now().isoformat().__str__()
            # , "new_col":"test"
        }
        print(customers)

        # Convert dictionary to JSON string and then to bytes
        customer_data = json.dumps(customers).encode("utf-8")

        #Pass the bytes data to the put_files method
        helper.put_files(Response=customer_data, Key=f'raw/customers/{uuid.uuid4().__str__()}.json')


if __name__ == "__main__":
    run()

"""

SQS QUEUE POLICY
{
    "Version": "2012-10-17",
    "Id": "example-ID",
    "Statement": [
        {
            "Sid": "example-statement-ID",
            "Effect": "Allow",
            "Principal": {
                "Service": "s3.amazonaws.com"
            },
            "Action": [
                "SQS:SendMessage"
            ],
            "Resource": "arn:aws:sqs:<REGION><ACCOUNT ID>:<QUEUE NAME>",
            "Condition": {
                "ArnLike": {
                    "aws:SourceArn": "arn:aws:s3:*:*:<S3 BUCKET>"
                },
                "StringEquals": {
                    "aws:SourceAccount": "<ACCOUNT ID>"
                }
            }
        }
    ]
}

"""
```
