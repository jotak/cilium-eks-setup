# cilium-eks-setup

(Personal setup)

## Start

```bash
export NAME=jtakvori-test-cilium-4
sed -i "s/name: jtak.*$/name: ${NAME}/" eks-config.yaml

eksctl create cluster -f ./eks-config.yaml --timeout 40m

cilium install
cilium hubble enable --ui

cilium status --wait
cilium connectivity test

cilium hubble ui

kubectl apply -f https://raw.githubusercontent.com/cilium/cilium/v1.11/examples/kubernetes/addons/prometheus/monitoring-example.yaml
```

To setup IAM OIDC (necessary to install ingress controller)
https://docs.aws.amazon.com/eks/latest/userguide/enable-iam-roles-for-service-accounts.html
Then:
https://docs.aws.amazon.com/eks/latest/userguide/aws-load-balancer-controller.html
(To get identity: `aws sts get-caller-identity`)

https://www.eksworkshop.com/beginner/130_exposing-service/ingress_controller_alb/

```bash
    eksctl create iamserviceaccount \
      --cluster=${NAME}\
      --region=eu-west-1\
      --namespace=kube-system \
      --name=aws-load-balancer-controller \
      --role-name "AmazonEKSLoadBalancerControllerRole" \
      --attach-policy-arn=arn:aws:iam::xxxxxxxxxxx:policy/AWSLoadBalancerControllerIAMPolicy \
      --approve
```

Setup Hubble metrics & monitoring

```bash
# METRICS
kubectl patch -n kube-system daemonset.apps/cilium --type='json' -p='[{"op": "add", "path": "/spec/template/metadata/annotations", "value":{"prometheus.io/port": "9090","prometheus.io/scrape": "true"}}]'
kubectl patch -n kube-system daemonset.apps/cilium --type='json' -p='[{"op": "add", "path": "/spec/template/spec/containers/0/ports", "value":[{"name":"prometheus","containerPort":9090,"hostPort":9090,"protocol":"TCP"},{"name":"hubble-metrics","containerPort":9091,"hostPort":9091,"protocol":"TCP"}]}]'

cilium config set prometheus-serve-addr ":9090"
cilium config set hubble-metrics-server ":9091"
cilium config set hubble-metrics "flow http:destinationContext=pod-short|dns|ip;sourceContext=pod-short|dns|ip flow tcp icmp drop:destinationContext=pod;sourceContext=pod dns:query;ignoreAAAA"
k apply -f ./hubble-metrics-service.yaml

# For DNS / L7: annotate pods with:
# io.cilium.proxy-visibility: <Egress/53/UDP/DNS>,<Egress/8080/TCP/HTTP>
# (that's enabled in mesh-arena)

kubectl -n cilium-monitoring port-forward service/prometheus --address 0.0.0.0 --address :: 9090:9090
kubectl -n cilium-monitoring port-forward service/grafana --address 0.0.0.0 --address :: 3000:3000

xdg-open http://localhost:3000/
```

In Prometheus:
```
1000 * sum(rate(hubble_http_request_duration_seconds_sum[1m])) by (source,destination) / sum(rate(hubble_http_request_duration_seconds_count[1m])) by (source,destination)

1000 * histogram_quantile(0.9, sum(rate(hubble_http_request_duration_seconds_bucket[1m])) by (le,source,destination))
```

## Destroy

```bash
eksctl delete cluster -f ./eks-config.yaml
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
