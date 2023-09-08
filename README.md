# Red Hat Summit Connect 2023 - GitOps Artefacts

## Pre-requisites

See [INFRASTRUCTURE.md](doc/INFRASTRUCTURE.md).

## Deploy OpenShift resources with OpenShift GitOps

* Install the OpenShift GitOps operator.

```yaml
oc apply -f gitops/pre-requisites
```

* Fix the ArgoCD ingress route in order to use the router default TLS certificate.

```sh
oc patch argocd openshift-gitops -n openshift-gitops -p '{"spec":{"server":{"insecure":true,"route":{"enabled": true,"tls":{"termination":"edge","insecureEdgeTerminationPolicy":"Redirect"}}}}}' --type=merge
```

* Get the Webhook URL of your OpenShift Gitops installation

```sh
oc get route -n openshift-gitops openshift-gitops-server -o jsonpath='https://{.spec.host}/api/webhook'
```

* Add a webhook to your GitHub/GitLab repo

  * Payload URL: *url above*
  * Content-Type: Application/json

* Deploy the ArgoCD Registry application

```sh
oc apply -f gitops/argocd/registry/registry.yaml
```

## Deploy Red Hat Developer Hub

See [BACKSTAGE.md](doc/BACKSTAGE.md).
