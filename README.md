# Helmfile with trunk repo model

## Repo layout 

```
├── README.md
├── environments
│   ├── base.yaml                           # imported by helmfile to load the environments
│   └── globals                             # collection of environment specific profiles
│       ├── _common.yaml
│       ├── production.yaml
│       └── staging.yaml
├── helmfile.yaml                           # single entrypoint "imports" environments/base.yaml, executes the helmfiles from releases folder
├── clusters
│   ├── xx1-prd-x                           # production
│   │   └── locals
│   │       └── values.yaml                 # concrete cluster "xx1-prd-x" profile
│   └── xx1-stg-x                           # staging
│       └── locals
│           └── values.yaml                 # concrete cluster "xx1-stg-x" profile
└── releases
    └── apps                                # collection of releases
        └── scdf
            ├── helmfile.yaml               # reference to concrete helm charts are here, for eample scdf (sping cloud data flow)
            └── values
                └── _common.yaml.gotmpl     # templated values passed down as a values to coherent release (scdf in this case)
```

## Execution model overview
root `helmfile.yaml` is a single entry point to execute any deployment, it expects to be called against a concrete cluster passed as an environment varaible `CLUSTER_ID` (ex: `CLUSTER_ID=xx1-stg-x`) and passed environment profile name `--environment` (ex: `--environment staging`)

It will load the environment profile and execute the helmfiles defined in it, which should point to helmfiles with concrete releases under `releases/*` directory


## Environment profiles
It is important to understand the environment profile load model. It is based on 3 levels:

* `environments/globals/_common.yaml` - Level 1 generic profile values properties used as a base
* `environments/globals/${environment}.yaml` - Level 2 concrete environment profile properties used as an overlay to common properties
* `clusters/${cluster}/locals/values.yaml` - Level 3 concrete cluster profile properties used as an overlay to environment and common properties


## How To deploy a new custom release?

We will base our following steps on already added `releases/apps/scdf/helmfile.yaml` release. We want to to be deployed with 1 replica of RabbitMQ server in staging and 3 replicas in productiuon environment. We will deploy it first on staging and after on production. Also this is ArgoCD focused tutorial so the "deployment" would mean a creation of k8s manifests (we don't run the `helm install` here).

* define level 1 generic profile values properties in `environments/globals/_common.yaml` add
```yaml
scdf:
  version: 2.10.0
  rabbitmq:
    replicas: 3
```
* define the level 2 concrete environment profile properties for staging `environments/globals/staging.yaml` add
```yaml
scdf:
  rabbitmq:
    replicas: 1
```
* define the level 3 concrete cluster profile properties for `xx1-stg-x` in `clusters/xx1-stg-x/locals/values.yaml`
```yaml
scdf:
  installed: true
```
* produce the k8s manifests for `xx1-stg-x` by running `CLUSTER_ID=xx1-stg-x helmfile --environment staging template` in the root directory. You can verify the number of RabbitMQ server replicas by appending `| grep "rabbitmq/templates/statefulset.yaml" -A 15 | grep replicas` to the previous command.
* now in order to "deploy" to production one needs to define the level 3 concrete cluster profile properties for `xx1-prd-x` in `clusters/xx1-prd-x/locals/values.yaml`
```yaml
scdf:
  installed: true
```
* produce the k8s manifests for `xx1-prd-x` by running `CLUSTER_ID=xx1-prd-x helmfile --environment production template` in the root directory. You can verify the number of RabbitMQ server replicas by appending `| grep "rabbitmq/templates/statefulset.yaml" -A 15 | grep replicas` to the previous command.

## How To update the conrete release?
It is a question of the updating the relevant profile property value, wether it is needed to be done on one single cluster only or across all environment or all clusters globaly.

## How To deploy a release shared in the git somewhere?
Now this is where the most beatuful part commes in. You can reuse the helmfiles produced by the community or your teams internally. Here is how to do that:

We will base this example on [cluster-autoscaler](https://github.com/cloudposse/helmfiles/tree/master/releases/cluster-autoscaler) cloudposse's helmfile. 
* add the corrsepondent helmfile reference in your root helmfile
```yaml
helmfiles:

- path: git::https://github.com/cloudposse/helmfiles.git@/releases/cluster-autoscaler/helmfile.yaml?ref=a9e11bf
  values:
  - {{ toYaml .Values.cluster_autoscaler | nindent 4 }}
```
here we do a pin to concrete git commit with `ref=a9e11bf` it can be any valid git ref though
* define level 1 generic profile values properties in `environments/globals/_common.yaml` add
```yaml
cluster_autoscaler:
    installed: true
    chart_version: 7.3.1
    image_repository: us.gcr.io/k8s-artifacts-prod/autoscaling/cluster-autoscaler
    image_tag: v1.16.5
    replica_count: 1
    rbac_enabled: true
    psp_enabled: false
    service_account_name: "cluster-autoscaler"
    limit_cpu: "200m"
    limit_memory: "256Mi"
    request_cpu: "100m"
    request_memory: "128Mi"
```
* define the level 3 concrete cluster profile properties for `xx1-stg-x` in `clusters/xx1-stg-x/locals/values.yaml`
```yaml
cluster_autoscaler:
    aws_region: "eu-west-3"
    iam_role_arn: "arn:partition:service:region:account:resource"
    cluster_name: "xx1-stg-x"
```
* produce the k8s manifests for `xx1-stg-x` by running `CLUSTER_ID=xx1-stg-x helmfile --environment staging template` in the root directory.

Satisfied with the default values shipped with the helmfile and do not want to repeat them over and over again? 

* Modify the `helmfiles` section in root helmfile to be:
```yaml
- path: git::https://github.com/cloudposse/helmfiles.git@/releases/cluster-autoscaler/helmfile.yaml?ref=a9e11bf
  values:
  - git::https://github.com/cloudposse/helmfiles.git@/releases/cluster-autoscaler/defaults.yaml?ref=a9e11bf
  - {{ toYaml .Values.cluster_autoscaler | nindent 4 }}
```
* cleanup the `cluster_autoscaler` values from generic profile values properties in `environments/globals/_common.yaml`