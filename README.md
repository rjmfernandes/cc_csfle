# Confluent Cloud CSFLE (Client Side Field Level Encryption)

- [Confluent Cloud CSFLE (Client Side Field Level Encryption)](#confluent-cloud-csfle-client-side-field-level-encryption)
  - [AWS Setup](#aws-setup)
  - [CC Setup](#cc-setup)
  - [Clients](#clients)

## AWS Setup

We will be running this for AWS so follow instructions as per https://github.com/confluentinc/csfle-examples/tree/master/aws_shared_kek

## CC Setup

- Create environment in CC with Advanced Streams Governance package Check https://docs.confluent.io/cloud/current/security/encrypt/csfle/quick-start.html#requirements
- Add PII tag to your environment
- Follow instructions on https://github.com/confluentinc/csfle-examples/tree/master/aws_shared_kek 

For fetching permission statement use:

```shell
curl --url "${SR_URL}/dek-registry/v1/policy" \
  --header "Authorization: Basic ${SR_BASE64_CRED}" \
  | jq -r '.policy' \
  | sed 's/\\"/"/g' \
  | sed 's/\\n/\n/g' \
  | jq .
```

Use the Principal AWS to update KMS key policy by adding an entry when switching to policy view similar to this (replace Principal AWS accordingly):

```json

        {
            "Sid": "Allow Confluent Schema Registry Access",
            "Effect": "Allow",
            "Principal": {
                "AWS": "arn:aws:iam::01234501234501:role/confluent-schema-registry-kms-access"
            },
            "Action": [
                "kms:Encrypt",
                "kms:Decrypt",
                "kms:ReEncrypt*",
                "kms:GenerateDataKey*",
                "kms:DescribeKey"
            ],
            "Resource": "*"
        }
```

## Clients

You should from steps before have a .env file with:

```
export TARGET_TOPIC=csfle-kek-shared-demo
export TARGET_CONSUMER_GROUP=csfle-cg
export CC_API_KEY=<CONFLUENT_API_KEY>
export CC_API_SECRET=<CONFLUENT_API_SECRET>
export CC_BOOTSRAP_SERVER=<CONFLUENT_BOOTSTRAP_URL>
export SR_URL=https://<SR_BASE_URL>
export SR_CRED=<SR_API_KEY:SR_API_SECRET>
export SR_BASE64_CRED=<CREATE THIS BY RUNNING echo -en "<SR_API_KEY:SR_API_SECRET>" | base64 -w0>
export AWS_KEK_NAME=pventurini-csfle-shared-kek-demo-key
export AWK_KMS_KEY_ID=<KMS_KEY_ARN>
```

Clone the examples project:

```shell
git clone https://github.com/confluentinc/csfle-examples
```

Let's run the producer:

```shell
cd csfle-examples/aws_shared_kek/KafkaProducer
```

Check jvm target on `csfle-examples/aws_shared_kek/KafkaProducer/build.gradle.kts`. In case it is different than your java change accordingly.

Create topic `csfle-kek-shared-demo` in CC and execute:

```shell
./gradlew run
```

You can check on Confluent Cloud messages being produced with birthday encrypted.

For consuming :

```shell
kafka-avro-console-consumer --topic ${TARGET_TOPIC}  --bootstrap-server   ${CC_BOOTSRAP_SERVER}   --property schema.registry.url=${SR_URL} --property basic.auth.user.info="${SR_CRED}" --property basic.auth.credentials.source=USER_INFO --from-beginning --consumer-property security.protocol=SASL_SSL --consumer-property sasl.mechanism=PLAIN --consumer-property sasl.jaas.config="org.apache.kafka.common.security.plain.PlainLoginModule required username=\"${CC_API_KEY}\" password=\"${CC_API_SECRET}\";"
```

It is also possible not to share KEK with Confluent. Check `csfle-examples/aws_shared_kek/KafkaProducer/src/main/kotlin/ProducerProperties.kt`. In this scenario the clients requires KMS access.
