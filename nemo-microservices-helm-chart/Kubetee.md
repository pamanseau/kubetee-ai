# NEMO Microservices

## Create Namespace & Install the Secrets

```sh
kubectl create namespace nemo
```

```sh
kubectl create secret -n nemo docker-registry nvcrimagepullsecret --docker-server=nvcr.io --docker-username='$oauthtoken' --docker-password=YOUR_NVIDIA_API_KEY_HERE
```

```sh
kubectl create secret generic ngc-api -n nemo --from-literal=NGC_API_KEY=YOUR_NVIDIA_API_KEY_HERE \
--from-literal=NVIDIA_API_KEY=YOUR_NVIDIA_API_KEY_HERE
```

```sh
kubectl create secret generic hf-token -n nemo --from-literal=HF_TOKEN=YOUR_HUGGING_FACE_TOKEN_HERE
```

## Helm Chat Install

```sh
helm upgrade --install nemo nmp/nemo-microservices-helm-chart --version 25.9.0 -n nemo -f nemo-microservices-helm-chart/values-nemo-kubetee.yaml
```

## Deploy GPT-OSS-20B

```sh
kubectl apply -f ./nemo-microservices-helm-chart/nim-gpt-oss-20b-cc.yaml
```