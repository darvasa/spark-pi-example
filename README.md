# Spark Pi Example 

Spark Pi is a sample project to trigger a short lived Spark CI/CD workflow on Banzai Cloud's [Pipeline](https://github.com/banzaicloud/pipeline) PaaS platform.

## Prerequisites

* Account on one of the following cloud providers: Amazon, Azure, GoogleCloud
* A Banzai Cloud Control Plane needs to be running and accessible
* Oauth2 authentication enabled on the Banzai Cloud Control Plane

## Setup

* fork the project into your repository
* make a copy of the flow descriptor template corresponding to your chosen cloud provider:
```
cp .pipeline.yml.[azure|aws|gke].template .pipeline.yml
```
* check the flow descriptor file and replace the placeholders with your specific values (cluster name, bucket / folder names etc ...)

> In short: the CI/CD flow descriptor file contains the steps needed from provisioning the environment, building the code, running tests to being deployed and executed along with project specific variables (eg.:credentials, program arguments, etc ...).

* navigate to the CI/CD user interface (that usually runs on the Banzai Cloud Control plane instance)
* enable the project build from the list of available repositories
* add the following secrets to the build:

```
PLUGIN_ENDPOINT = [control-plane]/pipeline/api/v1
PLUGIN_TOKEN = "oauthToken"
```


The project is configured now for the Banzai Cloud CI/CD flow. On each commit to the repository a new flow will be triggered. You can check the progress on the CI/CD user interface.

## Banzai Cloud CI/CD flow descriptor templates

All the templates describe the following CI/CD flow:

* Create a Kubernetes cluster
* Set up monitoring on the cluster (optional, you can skip / remove this part from the template)
* Install spark resources
* Checkout the project
* Build the SparkPi project (the fork of this project)
* Run the SparkPi application (issue a spark-submit command to the above set up environment)

The templates only differ in the steps related to setting up the environment and using the cloud resources (storage, buckets, folders).

Eg.: the *create_cluster* step:

[Amazon]
```yaml
create_cluster:
    image: banzaicloud/plugin-pipeline-client:0.3.0
    cluster_name: "[[your-cluster-name]]"
    cluster_provider: "amazon"

    secrets: [plugin_endpoint, plugin_token]
```

[Azure]
```yaml
create_cluster:
    image: banzaicloud/plugin-pipeline-client:0.3.0
    cluster_name: "[[your-cluster-name]]"
    cluster_provider: "azure"
    azure_resource_group: "[[az_resource_group]]"

    secrets: [plugin_endpoint, plugin_token]
```

[Google Cloud]
```yaml
create_cluster:
    image: banzaicloud/plugin-pipeline-client:0.3.0
    cluster_name: "[[your-cluster-name]]"
    cluster_provider: "google"
    google_project: "[[google-project]]"

    secrets: [plugin_endpoint, plugin_token]
```

The *create_cluster* step listed above can be further customised by providing more details like the cluster location, instance type etc...
(more details [here](https://github.com/banzaicloud/drone-plugin-pipeline-client))

The other major difference is in the configuration related to the persistent storage used by the spark (to write logs to) and the Spark History Server (to read from)

[Amazon]
```yaml
install_spark_resources:
    image: banzaicloud/plugin-pipeline-client:0.3.0

    deployment_name: "banzaicloud-stable/spark"
    deployment_values:
      historyServer:
        enabled: true
      spark-hs:
        app:
          logDirectory: "s3a://[[your-s3-bucket]]/"

    secrets: [plugin_endpoint, plugin_token]
```

[Azure]
```yaml
install_spark_resources:
    image: banzaicloud/plugin-pipeline-client:0.3.0

    deployment_name: "banzaicloud-stable/spark"
    deployment_values:
      historyServer:
        enabled: true
      spark-hs:
        app:
          logDirectory: "wasb://[your-blob-container]]@{{ .PLUGIN_AZURE_STORAGE_ACCOUNT }}.blob.core.windows.net/"
          azureStorageAccountName: "{{ .PLUGIN_AZURE_STORAGE_ACCOUNT }}"
          azureStorageAccountName: "{{ .PLUGIN_AZURE_STORAGE_ACCOUNT_ACCESS_KEY }}"

    secrets: [plugin_endpoint, plugin_token,plugin_azure_storage_account, plugin_azure_storage_account_access_key]

```

[Google Cloud]
```yaml
install_spark_resources:
    image: banzaicloud/plugin-pipeline-client:0.3.0

    deployment_name: "banzaicloud-stable/spark"
    deployment_values:
     historyServer:
        enabled: true
     spark-hs:
       app:
         logDirectory: "gs://[[your-gs-bucket]]/"

    secrets: [plugin_endpoint, plugin_token]
```
> Note: currently the storage resources need to be set up manually in case of all cloud providers. In case of azure the
storage location is dynamically  assembled using the azure secrets.
