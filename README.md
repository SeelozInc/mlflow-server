# MLflow Server

A Dockerfile that produces a miniconda3 image with [MLflow](https://www.mlflow.org) installed.

## Usage

To run this image locally:

```bash
docker build --rm -f "Dockerfile" -t mlflow-server:latest .
docker run --rm --name mlflow-server -v /tmp/mlruns:/mlflow/ -p 5000:5000 mlflow-server
```

To run this image from docker hub:

```bash
docker run -v /tmp/mlruns:/mlflow/ -p 5000:5000 seeloz/mlflow-server
```

To set the artifact repository with Google Cloud Storage (GCS):

First create the volume directory, e.g., /tmp/mlruns, and then copy your Google Cloud Storage bucket credentials file to it and then set the GOOGLE_APPLICATION_CREDENTIALS environment variable to that file.

```bash
mkdir -p /tmp/mlruns
cp ~/.credentials/storage.json /tmp/mlruns
docker run -v /tmp/mlruns:/mlflow/ -e "ARTIFACT_ROOT=gs://<my_gcs_bucket>/<sub_directories>" -e GOOGLE_APPLICATION_CREDENTIALS="storage.json" -p 5000:5000 seeloz/mlflow-server
```
