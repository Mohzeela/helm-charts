# HomeChoice Service Helm Chart

## Overview

This repository contains a Helm Chart that is used to deploy a standard HomeChoice microservice to the Kubernetes cluster. The Helm Chart ensures a consistent deployment across all services. It ensure that best practices are followed and that the Kubernetes manifest files adheres to security standards.

## Structure

The repository contains only a few files and a single folder. It is based off a basic Helm Chart that was generated using the command `helm create helm-chart`.

Below is the folder structure of the Chart:

```bash
├── Chart.yaml
├── README.md
├── charts
│   ├── NOTES.txt
│   ├── _helpers.tpl
│   ├── configmaps.yaml\
│   ├── deployment.yaml
│   ├── hpa.yaml
│   ├── ingress.yaml
│   ├── service.yaml
│   ├── serviceaccount.yaml
│   └── tests
│       └── test-connection.yaml
└── values_example.yaml
```

The most important piece of the puzzle is the `templates/` directory. This is where Helm finds the YAML definitions for your Services, Deployments and other Kubernetes objects.

The templates in the `templates/` directory makes use of the Helm-specific objects `.Chart` and `.Values.`. The former provides metadata about the chart to your definitions such as the name, or version. The latter `.Values` object is a key element of Helm charts, used to expose configuration that can be set at the time of deployment. The defaults for this object are defined in the `values.yaml` file located in the project root.

All the values defined in the `values.yaml` file can be overridden by your service at deploy time.

The `_helpers.tpl` contains a set of reusable functions that help you build your helm template files.

## Deploying the Helm Chart

For service to use the Helm Chart, the chart needs to be made available for them to download. We have opted to make the Helm Chart available via an AWS S3 bucket. A Helm Plugin is installed via the build pipeline to help us package and upload the chart to S3. Don't worry to much about that, all you need to care about is that the Helm Chart is uploaded to an S3 bucket automatically via this repositories build pipeline.

You can view the `.gitlab-ci.yml` file if you are interested to know more.

## Using the Helm Chart to deploy your service

Below is a snippet that shows how you can utilise the Helm Chart for deploying your service via Gitlab CI.

```yaml
deploy-dev-devops:
  stage: deploy
  script:
    - echo "Deploying with helm"
    - aws configure set aws_access_key_id $ACCESS_KEY_ID
    - aws configure set aws_secret_access_key $SECRET_KEY
    - aws configure set default.region eu-west-1
    - aws eks update-kubeconfig --name HC-DevStage-App-EKS
    - rm -r ~/.aws
    - unset AWS_ACCESS_KEY AWS_SECRET_ACCESS_KEY
    - aws sts get-caller-identity
    - helm repo add hc $CHART_S3_BUCKET_URL
    - helm repo list
    - helm repo update
    - aws configure set aws_access_key_id $ACCESS_KEY_ID
    - aws configure set aws_secret_access_key $SECRET_KEY
    - helm upgrade --install $DEPLOY_NAME --namespace dev-devops --set environment=dev-devops,image.repository=$ECR_URL/$IMAGE_NAME,image.tag=$IMAGE_TAG -f k8s/dev.yaml hc/service
```

The most important parts of the snippet are:

```yaml
    - helm repo add hc $CHART_S3_BUCKET_URL
    - helm repo list
    - helm repo update
```

This sets up the `helm` tool to be able to use the Chart.

```yaml
    - helm upgrade --install $DEPLOY_NAME --namespace dev-devops --set environment=dev-devops,image.repository=$ECR_URL/$IMAGE_NAME,image.tag=$IMAGE_TAG -f k8s/dev.yaml hc/service
```

This calls helm to perform the deployment. Installing the service if it is not yet running in Kubernetes, or updating it if it is already available.

Note how we are overridden the image repository and environment:

```bash
... --set environment=dev-devops,image.repository=$ECR_URL/$IMAGE_NAME,image.tag=$IMAGE_TAG
```

Also note how we are providing an override file for the `values.yml`:

```bash
... -f k8s/dev.yaml hc/service
```

## Testing the output YAMl

The command below allows you to run the helm template against a `values.yml` and output the generated Kubernetes yaml.

```bash
helm template hc-chatops-test -f helm-chart/example/values_example.yaml ./helm-chart
```
