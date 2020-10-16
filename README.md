# README

Solution based on:
https://medium.com/google-cloud/kafka-to-bigquery-using-dataflow-6ec73ec249bb

### Start a testing Kafka VM

### Create BigQuery dataset and table
```
gcloud config set project warm-actor-291222
bq mk --location US --dataset kafka_to_bigquery
bq mk --table \
--schema schema.json \
--time_partitioning_field transaction_time \
kafka_to_bigquery.transactions
```
### Review the kafka-to-bigquery code

Ref: https://github.com/GoogleCloudPlatform/DataflowTemplates/tree/master/v2/kafka-to-bigquery

Another reference:  
https://github.com/GoogleCloudPlatform/java-docs-samples/tree/master/dataflow/flex-templates/kafka_to_bigquery


### Ue Cloud Shell to build the template java code using maven
```
sudo apt-get install git

sudo apt-get install maven

git clone https://github.com/GoogleCloudPlatform/DataflowTemplates
cd DataflowTemplates/v2/
export PROJECT=warm-actor-291222
export IMAGE_NAME=mykafkaimage
export TARGET_GCR_IMAGE=gcr.io/${PROJECT}/${IMAGE_NAME}
export BASE_CONTAINER_IMAGE=gcr.io/dataflow-templates-base/java8-template-launcher-base
export BASE_CONTAINER_IMAGE_VERSION=latest
export TEMPLATE_MODULE=kafka-to-bigquery
export APP_ROOT=/template/${TEMPLATE_MODULE}
export COMMAND_SPEC=${APP_ROOT}/resources/${TEMPLATE_MODULE}-command-spec.json
mvn clean package -Dimage=${TARGET_GCR_IMAGE} \
                  -Dbase-container-image=${BASE_CONTAINER_IMAGE} \
                  -Dbase-container-image.version=${BASE_CONTAINER_IMAGE_VERSION} \
                  -Dapp-root=${APP_ROOT} \
                  -Dcommand-spec=${COMMAND_SPEC} \
                  -am -pl ${TEMPLATE_MODULE}

mvn clean package -Dimage=${TARGET_GCR_IMAGE} \
                  -Dbase-container-image=${BASE_CONTAINER_IMAGE} \
                  -Dbase-container-image.version=${BASE_CONTAINER_IMAGE_VERSION} \
                  -Dapp-root=${APP_ROOT} \
                  -Dcommand-spec=${COMMAND_SPEC}
```

### Submit the jar as Dataflow job
```
export BUCKET_NAME=gs://warm-actor-291222_cloudbuild
export TEMPLATE_IMAGE_SPEC=${BUCKET_NAME}/images/kafka-to-bigquery-image-spec.json

gsutil cp kafka-to-bigquery-image-spec.json ${TEMPLATE_IMAGE_SPEC}

export TOPICS=txtopic
export BOOTSTRAP=10.128.0.2:9092


export OUTPUT_TABLE=${PROJECT}:kafka_to_bigquery.transactions
export JS_PATH=${BUCKET_NAME}/my_function.js
export JS_FUNC_NAME=transform

export JOB_NAME="${TEMPLATE_MODULE}-`date +%Y%m%d-%H%M%S-%N`"

# enable dataflow API
gcloud services enable dataflow.googleapis.com
# submit streaming job
gcloud beta dataflow flex-template run ${JOB_NAME} \
        --project=${PROJECT} --region=us-west1 \
        --template-file-gcs-location=${TEMPLATE_IMAGE_SPEC} \
        --parameters \
^~^outputTableSpec=${OUTPUT_TABLE}~inputTopics=${TOPICS}~javascriptTextTransformGcsPath=${JS_PATH}~javascriptTextTransformFunctionName=${JS_FUNC_NAME}~bootstrapServers=${BOOTSTRAP}

# I was trying it on us-central1 region but it failed with this
Workflow failed. Causes: Error: Message: Required 'compute.images.get' permission for 'projects/dataflow-service-producer-prod/global/images/dataflow-dataflow-owned-resource-20200921-14-rc01' HTTP Code: 403

Changed to us-west1 and all works well. $#@#!
```

### Test streaming by publish to Kafka topic

```
/opt/kafka/bin/kafka-topics.sh --create --zookeeper localhost:2181 \
--replication-factor 1 \
--partitions 1 --topic txtopic

/opt/kafka/bin/kafka-topics.sh --list --zookeeper localhost:2181

sudo apt-get install jq
(echo -n "1|"; cat message.json | jq . -c) | /opt/kafka/bin/kafka-console-producer.sh \
--broker-list localhost:9092 \
--topic txtopic \
--property "parse.key=true" \
--property "key.separator=|"
```