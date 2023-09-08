# Infrastructure

Création d'un utilisateur IAM en suivant les [instructions d'AWS](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_users_create.html) et [documentation OpenShift](https://docs.openshift.com/container-platform/4.13/installing/installing_aws/installing-aws-account.html#installation-aws-iam-user_installing-aws-account).

Puis, utilisation de cet utilisateur avec la CLI AWS.

```sh
aws configure
```

Création du install-config.

```sh
openshift-install create install-config --dir cluster
```

Modifier le install-config.yaml.

```yaml
additionalTrustBundlePolicy: Proxyonly
apiVersion: v1
baseDomain: sandbox2218.opentlc.com
compute:
- architecture: amd64
  hyperthreading: Enabled
  name: worker
  platform:
    aws:
      type: m5.metal
      zones:
      - eu-west-3a
      - eu-west-3b
      - eu-west-3c
  replicas: 3
controlPlane:
  architecture: amd64
  hyperthreading: Enabled
  name: master
  platform:
    aws:
      rootVolume:
        iops: 4000
        size: 500
        type: io1 
      type: m5.2xlarge
      zones:
      - eu-west-3a
      - eu-west-3b
      - eu-west-3c
  replicas: 3
metadata:
  creationTimestamp: null
  name: summitconnect
networking:
  clusterNetwork:
  - cidr: 10.128.0.0/14
    hostPrefix: 23
  machineNetwork:
  - cidr: 10.0.0.0/16
  networkType: OVNKubernetes
  serviceNetwork:
  - 172.30.0.0/16
platform:
  aws:
    region: eu-west-3
publish: External
pullSecret: REDACTED
sshKey: |
  REDACTED
```

```
$ openshift-install create cluster --dir cluster --log-level info
INFO Credentials loaded from the "default" profile in file "/home/nmasse/.aws/credentials"
INFO Consuming Install Config from target directory
INFO Creating infrastructure resources...
INFO Waiting up to 20m0s (until 6:34PM) for the Kubernetes API at https://api.summitconnect.sandbox2218.opentlc.com:6443... 
INFO API v1.26.7+0ef5eae up                       
INFO Waiting up to 30m0s (until 6:44PM) for bootstrapping to complete... 
INFO Destroying the bootstrap resources...        
INFO Waiting up to 40m0s (until 6:54PM) for the cluster at https://api.summitconnect.sandbox2218.opentlc.com:6443 to initialize... 
INFO Checking to see if there is a route at openshift-console/console... 
INFO Install complete!                            
INFO To access the cluster as the system:admin user when using 'oc', run 'export KUBECONFIG=/home/nmasse/tmp/summit-connect-2023/cluster/auth/kubeconfig' 
INFO Access the OpenShift web-console here: https://console-openshift-console.apps.summitconnect.REDACTED.opentlc.com 
INFO Login to the console with user: "kubeadmin", and password: "REDACTED" 
INFO Time elapsed: 7s                             

$ oc login -u kubeadmin -p REDACTED https://api.summitconnect.REDACTED.opentlc.com:6443

$ oc get node -o custom-columns="NAME:.metadata.name,CPU:.status.capacity.cpu,RAM:.status.capacity.memory" 
NAME                                         CPU   RAM
ip-10-0-145-104.eu-west-3.compute.internal   96    395846468Ki
ip-10-0-150-98.eu-west-3.compute.internal    8     32037804Ki
ip-10-0-170-57.eu-west-3.compute.internal    8     32381868Ki
ip-10-0-185-77.eu-west-3.compute.internal    96    395846468Ki
ip-10-0-202-228.eu-west-3.compute.internal   96    395846468Ki
ip-10-0-204-250.eu-west-3.compute.internal   8     32037804Ki
```

**Configuration de l'authentification Google**

Il faut ajouter l'URL `https://oauth-openshift.apps.summitconnect.REDACTED.opentlc.com/oauth2callback/RedHatSSO` au [projet Google](https://console.cloud.google.com/apis/credentials). Cf. [Secure your OpenShift 4 cluster with OpenID Connect authentication](https://www.itix.fr/blog/secure-openshift-4-openid-connect-authentication/).

```sh
export GOOGLE_CLIENT_SECRET=REDACTED
export GOOGLE_CLIENT_ID=REDACTED
export KUBECONFIG=/home/nmasse/tmp/summit-connect-2023/cluster/auth/kubeconfig
oc create secret generic redhat-sso-client-secret -n openshift-config --from-literal="clientSecret=$GOOGLE_CLIENT_SECRET"
oc apply -f - <<EOF
apiVersion: config.openshift.io/v1
kind: OAuth
metadata:
  name: cluster
spec:
  identityProviders:
  - google:
      clientID: "$GOOGLE_CLIENT_ID"
      clientSecret:
        name: redhat-sso-client-secret
      hostedDomain: redhat.com
    mappingMethod: claim
    name: RedHatSSO
    type: Google
EOF
oc apply -f - <<EOF
apiVersion: user.openshift.io/v1
kind: Group
metadata:
  name: demo-admins
users:
- nmasse@redhat.com
- slallema@redhat.com
- alegros@redhat.com
- atiouajn@redhat.com
- amazeas@redhat.com
- dmartini@redhat.com
- feven@redhat.com
- fdavalo@redhat.com
- gestrem@redhat.com
- jbrusq@redhat.com
- mamokhta@redhat.com
- mouachan@redhat.com
- ptruong@redhat.com
EOF
oc adm policy add-cluster-role-to-group cluster-admin demo-admins
```

**Let's Encrypt**

```sh
# kubeconfig can be downloaded from the Assisted Installer console
export KUBECONFIG=/home/nmasse/tmp/summit-connect-2023/cluster/auth/kubeconfig

# Cluster DNS domain
export DOMAIN=summitconnect.REDACTED.opentlc.com

# Get a valid certificate
sudo dnf install -y golang-github-acme-lego
lego -d "api.$DOMAIN$" -d "*.apps.$DOMAIN" -a -m nmasse@redhat.com --dns route53 run

# Install it on the router
cert_dn="$(openssl x509 -noout -subject -in .lego/certificates/api.$DOMAIN.crt)"
cert_cn="${cert_dn#subject=CN = }"
kubectl create secret tls router-certs-$(date "+%Y-%m-%d") --cert=".lego/certificates/api.$DOMAIN.crt" --key=".lego/certificates/api.$DOMAIN.key" -n openshift-ingress --dry-run -o yaml > router-certs.yaml
kubectl apply -f "router-certs.yaml" -n openshift-ingress
kubectl patch ingresscontroller default -n openshift-ingress-operator --type=merge --patch-file=/dev/fd/0 <<EOF
{"spec": { "defaultCertificate": { "name": "router-certs-$(date "+%Y-%m-%d")" }}}  
EOF

# Install it on the api-server
kubectl create secret tls api-certs-$(date "+%Y-%m-%d") --cert=".lego/certificates/api.$DOMAIN.crt" --key=".lego/certificates/api.$DOMAIN.key" -n openshift-config --dry-run -o yaml > "api-certs.yaml"
kubectl apply -f "api-certs.yaml" -n openshift-config
kubectl patch apiserver cluster --type=merge --patch-file=/dev/fd/0 <<EOF
{"spec":{"servingCerts":{"namedCertificates":[{"names":["$cert_cn"],"servingCertificate":{"name": "api-certs-$(date "+%Y-%m-%d")"}}]}}}
EOF
```
