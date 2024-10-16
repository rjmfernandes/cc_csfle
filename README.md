# Confluent Cloud CSFLE (Client Side Field Level Encryption)

- [Confluent Cloud CSFLE (Client Side Field Level Encryption)](#confluent-cloud-csfle-client-side-field-level-encryption)
  - [Disclaimer](#disclaimer)
  - [References](#references)
  - [AWS Setup](#aws-setup)
  - [CC Setup](#cc-setup)
    - [Register Schema](#register-schema)
    - [Register KEK](#register-kek)
    - [Register Encryption Rules](#register-encryption-rules)
    - [Allow DEK Registry to access KMS](#allow-dek-registry-to-access-kms)
  - [Clients](#clients)
  - [Cleanup](#cleanup)

## Disclaimer

The code and/or instructions here available are **NOT** intended for production usage. 
It's only meant to serve as an example or reference and does not replace the need to follow actual and official documentation of referenced products.

## References

- https://github.com/confluentinc/csfle-examples/tree/master/aws_shared_kek
- https://docs.confluent.io/cloud/current/security/encrypt/csfle/quick-start.html

## AWS Setup

Let's set AWS `KMS` first. Under `Customer managed keys` create a key with all default settings. Let's label it `demo_csfle`. Copy its `ARN` for later.

## CC Setup

- Create environment in CC with **Advanced Streams Governance** package 
- Create *Basic* cluster under an AWS region
- Add **PII** tag to your environment
- Create a yopic on your cluster named `csfle-kek-shared-demo`

Let's get our API and secret for confluent cloud.

```shell
confluent login --save
```

List environments:

```shell
confluent env list
```

Using the id of your environment just created execute:

```shell 
confluent environment use <YOUR_ENV_ID>
```

List the clusters:

```shell
confluent kafka cluster list
```

Using the id of your cluster:

```shell
confluent kafka cluster use <YOUR_CLUSTER_ID>
```

And 

```shell
confluent api-key create --resource <YOUR_CLUSTER_ID>
```

Copy the API Key and Secret for later.

Execute: 

```shell
confluent kafka cluster describe
```

Copy the bootstrap server with the format `domain:port` from *Endpoint* without protocol for later.

Execute:

```shell
confluent sr cluster describe
```

Copy the Schema-Registry *Endpoint URL* for later.

Using the Schema-Registry *Cluster* id from command output before execute:

```shell
confluent api-key create --resource <YOUR_SR_CLUSTER_ID>
```

Copy the API Key and Secret for SR to use next.

Execute now:

```shell
echo -en "<SR_API_KEY>:<SR_API_SECRET>" | base64
```

Copy the result string as the *SR Base64 Credentials* to use next.

We will need a `.env` file with the variables required for our setup and compiled on steps before.

```shell
cat <<EOF > .env
#!/bin/sh
##
export TARGET_TOPIC=csfle-kek-shared-demo
export TARGET_CONSUMER_GROUP=csfle-cg
export CC_API_KEY=<CONFLUENT_KAFKA_API_KEY>
export CC_API_SECRET=<CONFLUENT_KAFKA_API_SECRET>
export CC_BOOTSRAP_SERVER=<CONFLUENT_BOOTSTRAP_URL>
export SR_URL=<SR_URL>
export SR_CRED=<SR_API_KEY>:<SR_API_SECRET>
export SR_BASE64_CRED=<SR_BASE64_CREDENTIALS>
export AWS_KEK_NAME=demo_csfle
export AWK_KMS_KEY_ID=<KMS_KEY_ARN>
EOF
```

Execute:

```shell
source .env
```

### Register Schema

```shell
curl --request POST \
  --url "${SR_URL}/subjects/csfle-kek-shared-demo-value/versions" \
  --header "Authorization: Basic ${SR_BASE64_CRED}" \
  --header "Content-Type: application/vnd.schemaregistry.v1+json" \
  --data '{
            "schemaType": "AVRO",
            "schema": "{  \"name\": \"PersonalData\", \"type\": \"record\", \"namespace\": \"com.csfleExample\", \"fields\": [{\"name\": \"id\", \"type\": \"string\"}, {\"name\": \"name\", \"type\": \"string\"},{\"name\": \"birthday\", \"type\": \"string\", \"confluent:tags\": [ \"PII\"]},{\"name\": \"timestamp\",\"type\": [\"string\", \"null\"]}]}",
            "metadata": {
            "properties": {
            "owner": "Paolo Venturini",
            "email": "pventurini@confluent.io"
            }
          }
    }'
```

### Register KEK

```shell
curl --request POST \
  --url "${SR_URL}/dek-registry/v1/keks" \
  --header "Authorization: Basic ${SR_BASE64_CRED}" \
  --header 'Content-Type: application/vnd.schemaregistry.v1+json' \
  --data "{
    \"name\": \"${AWS_KEK_NAME}\",
	  \"kmsType\": \"aws-kms\",
	  \"kmsKeyId\": \"${AWK_KMS_KEY_ID}\",
	  \"shared\": true	
}"
```

### Register Encryption Rules

```shell
curl --request POST \
  --url "${SR_URL}/subjects/csfle-kek-shared-demo-value/versions" \
  --header "Authorization: Basic ${SR_BASE64_CRED}" \
  --header 'Content-Type: application/vnd.schemaregistry.v1+json' \
  --data "{
        \"ruleSet\": {
        \"domainRules\": [
      {
        \"name\": \"encryptPII\",
        \"kind\": \"TRANSFORM\",
        \"type\": \"ENCRYPT\",
        \"mode\": \"WRITEREAD\",
        \"tags\": [\"PII\"],
        \"params\": {
           \"encrypt.kek.name\": \"${AWS_KEK_NAME}\"
          },
        \"onFailure\": \"ERROR,NONE\"
        }
        ]
      } 
    }"
```

### Allow DEK Registry to access KMS

Retrieve  permission statements:

```shell
curl --url "${SR_URL}/dek-registry/v1/policy" \
  --header "Authorization: Basic ${SR_BASE64_CRED}" \
  | jq -r '.policy' \
  | sed 's/\\"/"/g' \
  | sed 's/\\n/\n/g' \
  | jq .
```

Use the **Principal AWS** on the response to go back to AWS and update KMS key policy by adding an entry by switching to policy view and edit. The new entry on Statement array should be similar to this (replace Principal AWS accordingly):

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

Clone the examples project:

```shell
rm -fr csfle-examples
git clone https://github.com/confluentinc/csfle-examples
```

Let's run the producer:

```shell
cd csfle-examples/aws_shared_kek/KafkaProducer
```

Check jvm target on `csfle-examples/aws_shared_kek/KafkaProducer/build.gradle.kts`. In case it is different than your java change accordingly.

For example by doing:

```shell
sdk default java 17.0.11-tem
```

Now execute:

```shell
./gradlew run
```

You can check on Confluent Cloud messages being produced with birthday encrypted.

For consuming from another shell:

```shell
source ../../../.env
kafka-avro-console-consumer --topic ${TARGET_TOPIC}  --bootstrap-server   ${CC_BOOTSRAP_SERVER}   --property schema.registry.url=${SR_URL} --property basic.auth.user.info="${SR_CRED}" --property basic.auth.credentials.source=USER_INFO --from-beginning --consumer-property security.protocol=SASL_SSL --consumer-property sasl.mechanism=PLAIN --consumer-property sasl.jaas.config="org.apache.kafka.common.security.plain.PlainLoginModule required username=\"${CC_API_KEY}\" password=\"${CC_API_SECRET}\";"
```

You should see now all messages with birthday decrypted.

It is also possible not to share KEK with Confluent. Check `csfle-examples/aws_shared_kek/KafkaProducer/src/main/kotlin/ProducerProperties.kt`. In this scenario the clients requires KMS access and you should count that Confluent Cloud services as Flink won't be able to work with the encrypted field.

## Cleanup

Delete your CC environment and the Key `demo_csle` from AWS KMS.

```shell
rm -fr csfle-examples
rm .env
```
