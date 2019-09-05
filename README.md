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

You can similarly use AWS as well.

This image also supports three types of database:

* Filesystem Volume
* MySQL
* Postgres

In order to use MySQL, you need to create a database and then run the following:

```bash
docker run --rm -it mlflow-server mlflow db upgrade mysql://<username>:<password>@<host>/<database_name>
```

With Postgres, you will need to manually create the database, but no db upgrade command is required as we need with MySQL.

## Deploy to Kubernetes

Here is how you can deploy to Kubernetes on Google Cloud:

1) Deploy a GKE Cluster with VPC-native enabled

2) Apply a secret containing a service account with access to GCS by following the instructions here: [link](https://cloud.google.com/kubernetes-engine/docs/tutorials/authenticating-to-cloud-platform)

3) Create your Database

    * Postgres (Recommended!)
    
        * Create a Google SQL instance
        * Select MySQL and give it a name
        * Generate a password and __note it down__
        * Choose the same region as your GKE cluster
        * Enable Private IP and disable Public IP
        * Create the instance and wait till its created
        * Create an empty Database with any name such as _store_

    * MySQL
    
        * Create a Google SQL instance
        * Select MySQL and give it a name
        * Generate a password
        * Choose the same region as your GKE cluster
        * Enable Private IP and disable Public IP
        * Create the instance and wait till its created
        * Go to Users and create a new user with a password
        * Create an empty Database with any name such as _store_

    * PVC (Persistent-Volume-Claim)

        * Create a PVC with about 10Gi

4) Create a Deployment

    * Under Workfloads in GKE click 'Deploy'
    * Type in the following for the image path: "seeloz/mlflow-server"
    * Give it a name, such as _mlflow-server_
    * Click "Add Environment Variable"

        * Key: ARTIFACT_ROOT
        * Value: ```gs://<bucket_name>/<path_to_make_artifact_root>```
        * __Make sure to create that path in the Cloud bucket manually__

    * Click "Add Environment Variable"
        * Key: BACKEND_URI
        * Value (for databases): ```<db_type>:<username>:<password>@<internal_ip_of_database>/<database>```
            * e.g. ```postgresql://postgres:mypassword@10.54.12.4/store```
        * Value (for filesystem):
            ```/mlflow/store```

    * Click "Add Environment Variable"
        * Key: GOOGLE_APPLICATION_CREDENTIALS
        * Value: ```/var/secrets/google/key.json```

5) Since the GKE console doesn't allow us to modify the yaml before pod creation, we'll have to Edit it once it's created:
    * In Workfloads select your deployment and click Edit
    * Add the following to your spec/template/spec/containers section:

```yaml
        volumeMounts:
        - mountPath: /var/secrets/google
          name: google-cloud-key
```

    * Add the following to the spec/template/spec section:

```yaml
      volumes:
      - name: google-cloud-key
        secret:
          secretName: gcs-key
```

    * If you are using a PVC as your database backing, then you'll need to mount that here too at something like ```/mlflow/store```

6) Redeploy your pod deployment; this may happen automatically, but you can also force it by setting the Scale to 0, waiting for the pod count to go to zero, and then setting it back.

7) In your deployment, click on _Expose_ and create a Load Balancer service with an external IP.

    * For the port, choose 5000 on TCP

8) And that's it! Once the service is created, click on the web link and you should be good to go. That same load-balancer ip address is also what you should pass to things like your MLflowcontext class.
