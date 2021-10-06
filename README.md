# GitLab Pipeline

This project includes a serie of different operations to run in a [GitLab-CI](https://gitlab.com) pipeline.

## Using it

In order to use this pipeline, you must edit the following environment variables according to your pipeline configuration.

The `INFRA_IMAGE` is a Docker OCI compatible imaged, pulled to a registry which is able to execute, [`Helmfile`](https://github.com/roboll/helmfile), with the `Diff` plugin.

You can build the infra image by using the following `Dockerfile`:

```docker
FROM quay.io/roboll/helmfile:helm3-v0.138.7

WORKDIR /
RUN apk add --no-cache gnupg curl &&\
  helm plugin install https://github.com/mumoshu/helm-x --version v0.8.1

ENTRYPOINT ["/usr/local/bin/helmfile"]
```

### Editing `variables.yaml`:

```yaml
##----------------------------------------------
## Shared variables
##----------------------------------------------
---
variables:
  # Imaged used to deploy infra services
  DOCKER_HOST: tcp://127.0.0.1:2376
  DOCKER_DRIVER: overlay2
  DOCKER_TLS_VERIFY: 1
  DOCKER_TLS_CERTDIR: "/certs"
  DOCKER_CERT_PATH: "/certs/client"
  DOCKER_LINT_IGNORE: "--ignore DL3013 --ignore DL3018 --ignore DL3000 --ignore DL3059"
  GITLAB_URI: https://gitlab.com/training-gitops/"
  GIT_STRATEGY: fetch
  INFRA_IMAGE: "registry.gitlab.com/training-gitops/kubernetes-infra/infra-image/infra-image:latest"
  TF_IMAGE: "docker.io/hashicorp/terraform:1.0.7"
  YAMLLINT_OPTIONS: ""
```

### Adding environments

The `environments.yaml` controls all managed enviromnets. We can say each environment is similar to one cluster. You can add Kubernetes Clusters in GitLab by following this [doc](https://docs.gitlab.com/ee/user/project/clusters/add_existing_cluster.html). Once the cluster is managed by GitLab it automatically is converted to an environment, update variables according to your needs.

```yaml
##----------------------------------------------
## This template contains all environments where
## different microservices can be deployed.
##----------------------------------------------
## Normally one environment means one cluster
## tags specifies the runner where jobs will run.
---
.env-gitops-dev:
  environment:
    name: gitops-dev
    kubernetes:
      namespace: ${KUBE_NAMESPACE}
  variables:
    ENV: "dev"

.env-gitops-prod:
  environment:
    name: gitops-prod
    kubernetes:
      namespace: ${KUBE_NAMESPACE}
  variables:
    ENV: "prod"
```
### GPG Encryption
If you wish to encrypt secrets using GPG you need to create a Public/Private keypair compatible with [RFC-4880](https://datatracker.ietf.org/doc/html/rfc4880). Then you have to export the armor private key and include it in a GitLab variableCre (with the file format enabled).

For creating the Public/Private keypair please use:
```bash
$: gpg --full-generate-key --rfc4880
```
Then you have to export the generated private key and encode it using Base64:

```bash
gpg --list-keys
/Users/tty0/.gnupg/pubring.kbx
------------------------------
pub   rsa2048 2020-05-20 [SC] [expires: 2022-05-20]
      95A6C65F0BC6BF647AF386866C9E5CCCF742C8CA
uid           [ultimate] Facundo de la Cruz <fmdlc.unix@gmail.com>
sub   rsa2048 2020-05-20 [E] [expires: 2022-05-20]

pub   rsa4096 2021-09-24 [SC]
      FEC29192FE7204E670A6C10EA668AAA16DE15F67
uid           [ultimate] GitLab CI pipeline (GitLab CI) <gitlab@ekoparty.org>
sub   rsa4096 2021-09-24 [E]

$: gpg --export --armor FEC29192FE7204E670A6C10EA668AAA16DE15F67 | base64
```
Populate the project-wide environment variable called `GPG_PRIVATE_KEY` with the resulting output.

## Using the pipeline

To build a Docker image using Kaniko, please use:

```yam
##------------------------------------------------------------------------------
## Include templates
##------------------------------------------------------------------------------
include:
  - project: 'training-gitops/pipeline'
    ref: main
    file: 'linters/hadolint.yaml'
  - project: 'training-gitops/pipeline'
    ref: main
    file: 'utils/build_tag.yaml'
  - project: 'training-gitops/pipeline'
    ref: main
    file: 'docker/build_kaniko.yaml'

##------------------------------------------------------------------------------
## Variables
##------------------------------------------------------------------------------
variables:
  DOCKER_LINT_IGNORE: "--ignore DL3013 --ignore DL3018 --ignore DL3000 --ignore DL3059"
  GIT_PUSH: http://${GITLAB_CI_USER}:${GITLAB_CI_TOKEN}@${CI_PAGES_DOMAIN}/${CI_PROJECT_NAMESPACE}/${CI_PROJECT_NAME}.git

##------------------------------------------------------------------------------
## Stages
##------------------------------------------------------------------------------
stages:
  - lint
  - build_tag
  - build_image

##------------------------------------------------------------------------------
## LINT
##------------------------------------------------------------------------------
lint:
  stage: lint
  extends: .docker_lint

##------------------------------------------------------------------------------
## Build and push tags
##------------------------------------------------------------------------------
build_tag:
  stage: build_tag
  extends: .build_tag

##------------------------------------------------------------------------------
## MicroServices
##------------------------------------------------------------------------------
build_adservice:
  stage: build_image
  extends: .build
  variables:
    SERVICE: "adservice"
    IMAGE: "adservice"
    DOCKER_CONTEXT: "./src/adservice/"
```

## Contributing
Pull requests are welcome. For major changes, please open an issue first to discuss what you would like to change.

Please make sure to update tests as appropriate.

## License
[MIT](https://choosealicense.com/licenses/mit/)
