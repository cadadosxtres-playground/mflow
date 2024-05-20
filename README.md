# Develop ML model with MLflow

This repo summarise:
 * Some of the steps from the tutorial [Develop ML model with MLflow and deploy to Kubernetes](https://mlflow.org/docs/latest/deployment/deploy-model-to-kubernetes/tutorial.html#develop-ml-model-with-mlflow-and-deploy-to-kubernetes) using a model the [Heart Disease Prediction using Logistic Regression](https://github.com/eduai-repo/ML-Demo/blob/main/2%20Classification/2.%20One%20with%20Heart%20Disease%20Prediction.ipynb)
 * How to deploy a [MLFlow Model to Kubernetes using InferenceService](https://kserve.github.io/website/0.10/modelserving/v1beta1/mlflow/v2/#deploy-with-inferenceservice)


## Requirements
* Python 3.10 and a virtualmente created
* Kind, kubectl already installed + [cloud-provider-kind](https://kind.sigs.k8s.io/docs/user/loadbalancer/) up and running
* Installing KServer on the cluster already created
* (Optional) gcloud CLI

## Preparing the python environment

Firstly, we create a virtual environment by executing ```python3 -m venv .venv```

Then, after activating the new python environment we install all the requirements by ```pip install -r requirements.txt```


## Developing a ML model with MLFlow

Please go to Jupyter Notebook [mflow-heart-disease.ipynb](./mflow-heart-disease.ipynb) to develop a model using MLFlow

## Deploying a MLFlow Model to Kubernetes

This step is based on the deploy with InferenceService step from [MLFlow guide of KServe](https://kserve.github.io/website/0.10/modelserving/v1beta1/mlflow/v2/#deploy-with-inferenceservice)

### Cluster

Creating a kind kubernetes cluster following the snippet below

```bash
cat <<EOF | kind create cluster --name mlflow --config=-
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
- role: worker
  extraMounts:
    - hostPath: ./data
      containerPath: /data
    - hostPath: ./local_models ## WARNING!! <<<--- This a relative path from repository root
      containerPath: /local_models
- role: worker
  extraMounts:
    - hostPath: ./data ## WARNING!! <<<--- This a relative path from repository root
      containerPath: /data
    - hostPath: ./local_models ## WARNING!! <<<--- This a relative path from repository root
      containerPath: /local_models
EOF
```

### Installing KServer using script

We will use the quickstart method based on a bash script from document [Getting Started with KServe]( https://kserve.github.io/website/latest/get_started/#install-the-kserve-quickstart-environment)

```bash
curl -s "https://raw.githubusercontent.com/kserve/kserve/release-0.12/hack/quick_install.sh" | bash
```

After the installation we should check out that kserver controller is running without issues by executing the commmand ```kubectl -n kserve get pod```


### Deploying a model with InferenceService

This step we are going to deploy a public model storaged in Google Cloud ```gs://kfserving-examples/models/mlflow/wine``` in the Kubernetes cluster all ready created

The goal is to model wine quality based on physicochemical tests https://archive.ics.uci.edu/dataset/186/wine+quality and is using the dataset [winequality-red.csv](http://archive.ics.uci.edu/ml/machine-learning-databases/wine-quality/winequality-red.csv)


#### Downlading the model package

Firstly, we have a look into the content of the package

```bash
mkdir -p /tmp/local_model
cd /tmp/local_model
gsutil cp -r gs://kfserving-examples/models/mlflow/wine .
```

if We explore the content of the package we will see

```bash
$ tree wine/
wine/
├── conda.yaml
├── MLmodel
├── model.pkl
└── requirements.txt

0 directories, 4 files
```

#### Deploying the model into Kubernetes

We execute the snippet below:

```bash
cat <<EOF | kubectl apply -f -
apiVersion: "serving.kserve.io/v1beta1"
kind: "InferenceService"
metadata:
  name: "mlflow-v2-wine-classifier"
spec:
  predictor:
    model:
      modelFormat:
        name: mlflow
      protocolVersion: v2
      storageUri: "gs://kfserving-examples/models/mlflow/wine"
EOF
```

Getting the ```INGRESS_HOST``` and ```INGRESS_PORT```
```bash
export INGRESS_HOST=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
export INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="http2")].port}')
```

Getting a predicction based on input file [mlflow-input.json](./doc/mlflow-input.json)
```bash
SERVICE_HOSTNAME=$(kubectl get inferenceservice mlflow-v2-wine-classifier -o jsonpath='{.status.url}' | cut -d "/" -f 3)
INPUT_FILE="@doc/mlflow-input.json"

curl -v \
  -H "Host: ${SERVICE_HOSTNAME}" \
  -H "Content-Type: application/json" \
  -d ${INPUT_FILE} \
  http://${INGRESS_HOST}:${INGRESS_PORT}/v2/models/mlflow-v2-wine-classifier/infer
```



### Troubleshooting

#### kserver controll is not working

Sometimes the installation seems to stuck with kserve controllorer pod in CreatingContainer state due to an issue with certificate manager services ```certificate.cert-manager.io/serving-cert```

Currently, the only workaround found out is to re-run the installation bash script and checking out the kserve controller pod is running.