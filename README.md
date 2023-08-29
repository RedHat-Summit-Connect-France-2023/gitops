# Backstage

## Install backstage

This repo contains the material to deploy a Janus on OpenShift.

```shell
oc apply -f gitops/ns.yaml
oc apply -f gitops/sub.yaml
oc apply -f gitops/ClusterRoleBinding.yaml 
```


Create the cluster application
```shell
oc apply -f gitops/argocd/applications/cluster-operators/application.yaml
```

Download a Quay.io pullSecret associated with your Red Hat Developer Hub subscription. The pull secret grants you access to the developer preview version of the Red Hat Developer Hub.
```
cat <<EOF | oc create -f - -n backstage
apiVersion: v1
kind: Secret
metadata:
  name: rhdh-pull-secret
data:
  .dockerconfigjson: YOUR_QUAY_IO_TOKEN
type: kubernetes.io/dockerconfigjson
EOF
```

Create the backstage application
```shell
oc apply -f gitops/argocd/applications/backstage/application.yaml
```
For testing purposal, you can create your kustomized backstage application and deploy it in your own namespace (check `gitops/argocd/applications/backstage/application-slallemand.yaml` as an example)



Then go in your argocd application, login using admin as user and take the password in the openshift-gitops-cluster secret. You can get it with the following command:

```shell
oc extract secret/openshift-gitops-cluster -n openshift-gitops --to=-
``` 

Then go in Settings>Account>admin and click on Generate New. Keep the generated token, he will be create in the secret on next step.


## Configure integration with GitHub

Go in ```https://github.com/settings/developers``` click in ```New Oauth App``` and complete the form. In Homepage URL put the URL of the backstage route (ex: https://red-hat-developer-hub-backstage.apps.rosa-neve.kset.p1.openshiftapps) and for Authorization callback URL put the backstage route and add /api/auth/github/handler/frame (ex https://backstage-developer-hub-backstage-developer-hub.apps.rosa-neve.kset.p1.openshiftapps.com//api/auth/github/handler/frame). Then create your Oauth app and click on ```Generate a new client secret```. Keep the client ID and the Client Secret.   

Finally, go in ```https://github.com/settings/tokens``` and Generate a New Token. Give access to repo and workflow and click on Generate token. Keep the value as GH_TOKEN.

##techdocs
To use techdocs you need access to s3 storage. Create a s3 bucket in AWS and then get the a

## Generate backstage secret 


```
oc create secret generic backstage-secret -n backstage --from-literal=GH_TOKEN=<GH_TOKEN> --from-literal=GH_CLIENT_ID=<GH_CLIENT_ID> --from-literal=GH_CLIENT_SECRET=<GH_CLIENT_SECRET> --from-literal=ARGO_API_TOKEN=<ARGO_API_TOKEN> --from-literal=AWS_ID=<AWS_ID> --from-literal=AWS_ACCESS_KEY_ID=<AWS_ACCESS_KEY_ID> --from-literal=AWS_SECRET_ACCESS_KEY=<AWS_SECRET_ACCESS_KEY> --from-literal=AWS_REGION=<AWS_REGION>
```

NOTE: TODO faire ca dans la partie deployment

After the creation, you need to update the psql database using the following command

```shell
oc rsh -n backstage $(oc get pods -n backstage -l app=postgres -o name) /usr/bin/psql -c 'ALTER USER pg WITH CREATEDB;'
```
