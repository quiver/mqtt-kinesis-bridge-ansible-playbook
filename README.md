Deploying MQTT mosquitto, paho-mqtt and MQTT-Amazon Kinesis bridge with Ansible Playbook
----------------------------------------------------------------------------------------

This playbooks is intended to ease installation of awslabs/mqtt-kinesis-bridge.

This playbook deploys

- MQTT mosquitto http://mosquitto.org/
- paho-mqtt python binding https://www.eclipse.org/paho/clients/python/
- MQTT-Kinesis bridge https://github.com/awslabs/mqtt-kinesis-bridge

Getting Started
----------------------------------------------------------------------------------------


Launch EC2 instance
========================================================================================

Launch EC2 instance and once InstanceState becomes running, edit hosts.ini's IP address and 
run the playbook with the following command ::

        $ cat hosts.ini
        ; IP address of the target EC2 instance
        1.2.3.4
        $ ansible-playbook -i hosts.ini site.yml

Play with MQTT/mosquitto
========================================================================================

Log in to provisioned EC2 instance and make sure that MQTT/mosquitto is working

        [mqtt-ec2] $ service mosquitto  status
        mosquitto (pid  7612) is running...

Open two terminals. Let's name one terminal is `pub_term` and another is `sub_term`:
From `pub_term` terminal, subscribe to the topic `censors/temperature`

        [pub_term] $ mosquitto_sub -d -t censors/temperature
        1439621637: New connection from 127.0.0.1 on port 1883.
        Client mosqsub/355-ip-172-31-2 sending CONNECT
        1439621637: New client connected from 127.0.0.1 as mosqsub/355-ip-172-31-2 (c1, k60).
        Client mosqsub/355-ip-172-31-2 received CONNACK
        Client mosqsub/355-ip-172-31-2 sending SUBSCRIBE (Mid: 1, Topic: censors/temperature, QoS: 0)
        Client mosqsub/355-ip-172-31-2 received SUBACK
        Subscribed (mid: 1): 0

From `pub_term` terminal, publish a message to the topic `censors/temperature`

        [sub_term]$ mosquitto_pub -d -t censors/temperature -m "78.9"
        Client mosqpub/408-ip-172-31-2 sending CONNECT
        Client mosqpub/408-ip-172-31-2 received CONNACK
        Client mosqpub/408-ip-172-31-2 sending PUBLISH (d0, q0, r0, m1, 'censors/temperature', ... (4 bytes))
        Client mosqpub/408-ip-172-31-2 sending DISCONNECT

If MQTT works correctly, you should see message in `pub_term` terminal

        [pub_term]  ...
        1439621681: New connection from 127.0.0.1 on port 1883.
        1439621681: New client connected from 127.0.0.1 as mosqpub/373-ip-172-31-2 (c1, k60).
        Client mosqsub/355-ip-172-31-2 received PUBLISH (d0, q0, r0, m0, 'censors/temperature', ... (4 bytes))
        78.9
        1439621681: Client mosqpub/373-ip-172-31-2 disconnected.


Launch Kinesis Stream
========================================================================================

Lanuch Amazon Kinesis stream and wait till the StreamStatus becomes ACTIVE.

        $ aws kinesis describe-stream --stream-name Foo
        {
            "StreamDescription": {
                "StreamStatus": "CREATING",
                "StreamName": "Foo",
                "StreamARN": "arn:aws:kinesis:ap-northeast-1:000000000000:stream/Foo",
                "Shards": []
            }
        }


Integrate MQTT and Amazon Kinesis
========================================================================================

Run mqtt-Kinesis bridge and subscribe to `censors/+` topics:

        $ export STREAM=Foo
        $ cd /home/ec2-user/mqtt-kinesis-bridge
        $ python bridge.py $STREAM --region ap-northeast-1 --topic_name censors/+
        {
          "StreamDescription": {
            "HasMoreShards": false,
            "Shards": [
              {
                "HashKeyRange": {
                  "EndingHashKey": "10",
                  "StartingHashKey": "0"
                },
                "SequenceNumberRange": {
                  "StartingSequenceNumber": "20"
                },
                "ShardId": "shardId-000000000000"
              },
              {
                "HashKeyRange": {
                  "EndingHashKey": "34",
                  "StartingHashKey": "17"
                },
                "SequenceNumberRange": {
                  "StartingSequenceNumber": "49"
                },
                "ShardId": "shardId-000000000001"
              }
            ],
            "StreamARN": "arn:aws:kinesis:ap-northeast-1:000000000000:stream/Foo",
            "StreamName": "Foo",
            "StreamStatus": "ACTIVE"
          }
        }
        Starting MQTT-to-Kinesis bridge
        Bridge Connected, looping...
        Connection Msg:
        Subscribe topic: censors/+ RC: (0, 1)

From another terminal, publish message to the topic "censors/humidity"

        $ mosquitto_pub -d -t censors/humidity -m "78.9"
        Client mosqpub/29001-ip-172-31 sending CONNECT
        Client mosqpub/29001-ip-172-31 received CONNACK
        Client mosqpub/29001-ip-172-31 sending PUBLISH (d0, q0, r0, m1, 'censors/humidity', ... (4 bytes))
        Client mosqpub/29001-ip-172-31 sending DISCONNECT

From the bridge terminal, you'll receive a message like

        on_message topic: "censors/humidity" msg.payload: "78.9"
        -= put seqNum: 49553524959787167203100577550735447012172782095451553810


Retrieve records from Amazon Kinesis using `seqNum`

        $ aws kinesis get-shard-iterator \
          --stream-name Foo \
          --shard-id shardId-000000000001 \
          --shard-iterator-type AT_SEQUENCE_NUMBER \
          --starting-sequence-number 49553524959787167203100577550735447012172782095451553810
        {
            "ShardIterator": "LONG_ITERATOR_ID"
        }
        $ aws kinesis get-records --shard-iterator "LONG_ITERATOR_ID"
        {
            "Records": [
                {
                    "PartitionKey": "censors/humidity",
                    "Data": "NzguOQ==",
                    "SequenceNumber": "49553524959787167203100577550735447012172782095451553810"
                }
            ],
            "NextShardIterator": "...",
            "MillisBehindLatest": 199000
        }

Response data is base64 encoded, so verify that the data is same as the one we just sent:

        $ echo -n "NzguOQ==" | base64 -d
        78.9

Perfect!


Notice
----------------------------------------------------------------------------------------

This playbook is tested with

- Ansible 1.9
- Amazon Linux AMI 2015.03

but should work with CentOS 6.x and older Ansible.

