export REGION=us-west1
export BUCKET=warm-actor-291222_cloudbuild
export PROJECT=warm-actor-291222
python3 -m apache_beam.examples.wordcount \
  --region $REGION \
  --input gs://dataflow-samples/shakespeare/kinglear.txt \
  --output gs://$BUCKET/results/outputs \
  --runner DataflowRunner \
  --project $PROJECT \
  --temp_location gs://$BUCKET/tmp/