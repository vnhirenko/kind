# Kind and infra setup

## Create Cluster
`kind create cluster --config kind-cluster.yaml`

## Deploy Metallb
`kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.13.9/config/manifests/metallb-native.yaml`
`kubectl apply -f metallb-cm.yaml`

## Deploy Mertics Server
`kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml`
`kubectl patch deploy metrics-server -n kube-system --type='json'  -p='[{"op": "add", "path": "/spec/template/spec/containers/0/args/0", "value":"--kubelet-insecure-tls"}]'`

## Deploy Nginx Ingress
`helm upgrade --install ingress-nginx ingress-nginx \
  --repo https://kubernetes.github.io/ingress-nginx \
  --namespace ingress-nginx --create-namespace`

## add to hosts

```
myhosts="llab.com grafana-server.llab.com argocd.llab.com"
ip=$(kubectl get svc ingress-nginx-controller -n ingress-nginx -ojson |jq -r .status.loadBalancer.ingress[0].ip)
for myhost in $myhosts
do
toreplace=$(echo "$ip $myhost")
if grep -q " $myhost" /etc/hosts
then
sudo sed -i "/ $myhost/c\\$toreplace" /etc/hosts
else
echo $toreplace | sudo tee -a /etc/hosts
fi
done
```
## Install monitoring
`helm repo add bitnami https://charts.bitnami.com/bitnami`
`helm repo add prometheus-community https://prometheus-community.github.io/helm-charts`
`helm repo add grafana https://grafana.github.io/helm-charts`
`helm repo update`
`helm install node-exporter bitnami/node-exporter`
`helm upgrade -i prometheus prometheus-community/prometheus`
`helm upgrade -i  grafana grafana/grafana`

## Install cert-manager
`kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.11.0/cert-manager.yaml`
```
echo """
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: selfsigned-cluster-issuer
spec:
  selfSigned: {}
""" | kubectl apply -f -
```

## Install Argo
`kubectl create namespace argocd`
`kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml`
```
echo """
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  annotations:
    cert-manager.io/issuer: selfsigned-cluster-issuer
    nginx.ingress.kubernetes.io/force-ssl-redirect: 'true'
    nginx.ingress.kubernetes.io/ssl-passthrough: 'true'
    nginx.ingress.kubernetes.io/backend-protocol: 'HTTPS'
  name: argocd
  namespace: argocd
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - argocd.llab.com
  rules:
    - host: argocd.llab.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: argocd-server
                port:
                  number: 80
""" | kubectl apply -f -

cp argo-key.yaml tmp.yaml && cat ~/.ssh/id_rsa | xargs -i echo "    {}" >> tmp.yaml; kubectl apply -f tmp.yaml; rm tmp.yaml
```

## Install Applicationset
`kubectl apply -f argo-applicationset.yaml`


## Install CrossPlane

```
kubectl create namespace crossplane-system
helm repo add crossplane-stable https://charts.crossplane.io/stable
helm repo update

helm install crossplane --namespace crossplane-system crossplane-stable/crossplane
```

## Install crossplane aws provider

`BASE64ENCODED_AWS_ACCOUNT_CREDS=$(echo -e "[default]\naws_access_key_id = MYID\naws_secret_access_key = MYKEY" | base64  | tr -d "\n")`

```
echo """
apiVersion: pkg.crossplane.io/v1
kind: Provider
metadata:
  name: provider-aws
spec:
  package: xpkg.upbound.io/crossplane-contrib/provider-aws:v0.33.0
  packagePullPolicy: IfNotPresent
  revisionActivationPolicy: Automatic
  revisionHistoryLimit: 1
---
apiVersion: v1
kind: Secret
metadata:
  name: aws-account-creds
  namespace: crossplane-system
type: Opaque
data:
  creds: $BASE64ENCODED_AWS_ACCOUNT_CREDS
---
apiVersion: aws.crossplane.io/v1beta1
kind: ProviderConfig
metadata:
  name: default
spec:
  credentials:
    source: Secret
    secretRef:
      namespace: crossplane-system
      name: aws-account-creds
      key: creds

"""  | kubectl apply -f -
```
