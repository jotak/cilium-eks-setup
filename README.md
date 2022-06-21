# cilium-eks-setup

(Personal setup)

## Start

```bash
export NAME=jtakvori-test-cilium-5
sed -i "s/name: jtak.*$/name: ${NAME}/" eks-config.yaml

eksctl create cluster -f ./eks-config.yaml --timeout 40m

cilium install
cilium hubble enable --ui

cilium status --wait
cilium connectivity test

cilium hubble ui
```


## Setup Hubble metrics & monitoring

```bash
# METRICS
kubectl apply -f https://raw.githubusercontent.com/cilium/cilium/v1.11/examples/kubernetes/addons/prometheus/monitoring-example.yaml

kubectl patch -n kube-system daemonset.apps/cilium --type='json' -p='[{"op": "add", "path": "/spec/template/metadata/annotations", "value":{"prometheus.io/port": "9090","prometheus.io/scrape": "true"}}]'
kubectl patch -n kube-system daemonset.apps/cilium --type='json' -p='[{"op": "add", "path": "/spec/template/spec/containers/0/ports", "value":[{"name":"prometheus","containerPort":9090,"hostPort":9090,"protocol":"TCP"},{"name":"hubble-metrics","containerPort":9091,"hostPort":9091,"protocol":"TCP"}]}]'

cilium config set prometheus-serve-addr ":9090"
cilium config set hubble-metrics-server ":9091"
cilium config set hubble-metrics "flow http:destinationContext=pod-short|dns|ip;sourceContext=pod-short|dns|ip flow tcp icmp drop:destinationContext=pod-short|dns|ip;sourceContext=pod-short|dns|ip dns:query;ignoreAAAA"
k apply -f ./hubble-metrics-service.yaml

# For DNS / L7: annotate pods with:
# io.cilium.proxy-visibility: <Egress/53/UDP/DNS>,<Egress/8080/TCP/HTTP>
# (that's enabled in mesh-arena)

kubectl -n cilium-monitoring port-forward service/prometheus --address 0.0.0.0 --address :: 9090:9090
kubectl -n cilium-monitoring port-forward service/grafana --address 0.0.0.0 --address :: 3000:3000

xdg-open http://localhost:3000/
```

### In Prometheus:
```
1000 * sum(rate(hubble_http_request_duration_seconds_sum[1m])) by (source,destination) / sum(rate(hubble_http_request_duration_seconds_count[1m])) by (source,destination)

1000 * histogram_quantile(0.9, sum(rate(hubble_http_request_duration_seconds_bucket[1m])) by (le,source,destination))
```

## Grant EKS access

```bash
kubectl -n kube-system edit configmap aws-auth
```

Add `mapUsers`, e.g:

```yaml
  mapUsers: |
    - userarn: arn:aws:iam::xxxxxx:user/skhoury
      username: skhoury
      groups:
        - system:masters
```

Then, from the other machine:

```bash
aws eks update-kubeconfig --region eu-west-1 --name jtakvori-test-cilium-X
kubectl config set-context arn:aws:eks:eu-west-1:xxxxxx:cluster/jtakvori-test-cilium-X
```

## Deploy mesh-arena

```bash
kubectl create namespace mesh-arena && kubectl apply -f <(curl -L https://raw.githubusercontent.com/jotak/demo-mesh-arena/zizou/quickstart-naked.yml) -n mesh-arena
```

## Setup Ingress

```bash
eksctl utils associate-iam-oidc-provider --cluster=${NAME} --region=eu-west-1 --approve
aws iam create-policy --policy-name ELB-IAMPolicy-jtakvori-temporary --policy-document file://iam_policy.json

eksctl create iamserviceaccount \
  --cluster=${NAME}\
  --region=eu-west-1\
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --attach-policy-arn=arn:aws:iam::xxxxxxxxxxx:policy/ELB-IAMPolicy-jtakvori-temporary \
  --role-name "AmazonEKSLoadBalancerControllerRole" \
  --approve

helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=${NAME} \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller 

# From mesh-arena directory
kubectl apply -f alb-ingress.yaml -n mesh-arena
# Get URL from:
kubectl get ingress -n mesh-arena
```

## Destroy

```bash
cilium uninstall
kubectl delete namespace mesh-arena
eksctl delete cluster -f ./eks-config.yaml
aws iam delete-policy --policy-arn arn:aws:iam::xxxxxxx:policy/ELB-IAMPolicy-jtakvori-temporary
```

## First init eks-config


```bash
export NAME=jtakvori-test-cilium-4
cat <<EOF >eks-config.yaml
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: ${NAME}
  region: eu-west-1

managedNodeGroups:
- name: ng-1
  desiredCapacity: 2
  privateNetworking: true
  # taint nodes so that application pods are
  # not scheduled/executed until Cilium is deployed.
  # Alternatively, see the note below.
  taints:
   - key: "node.cilium.io/agent-not-ready"
     value: "true"
     effect: "NoExecute"
- name: ng-2
  desiredCapacity: 2
  privateNetworking: true
  # taint nodes so that application pods are
  # not scheduled/executed until Cilium is deployed.
  # Alternatively, see the note below.
  taints:
   - key: "node.cilium.io/agent-not-ready"
     value: "true"
     effect: "NoExecute"
EOF
```
