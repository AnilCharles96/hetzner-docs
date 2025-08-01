# Install gateway crds
```
kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/gateway-api/v1.2.0/config/crd/standard/gateway.networking.k8s.io_gatewayclasses.yaml
kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/gateway-api/v1.2.0/config/crd/standard/gateway.networking.k8s.io_gateways.yaml
kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/gateway-api/v1.2.0/config/crd/standard/gateway.networking.k8s.io_httproutes.yaml
kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/gateway-api/v1.2.0/config/crd/standard/gateway.networking.k8s.io_referencegrants.yaml
kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/gateway-api/v1.2.0/config/crd/standard/gateway.networking.k8s.io_grpcroutes.yaml

kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/gateway-api/v1.2.0/config/crd/experimental/gateway.networking.k8s.io_tlsroutes.yaml
```

# Install cilium
```
helm upgrade cilium cilium/cilium --version 1.18.0 \
    --namespace kube-system -f values.yaml
  
kubectl -n kube-system rollout restart deployment/cilium-operator
kubectl -n kube-system rollout restart ds/cilium
```

# `values.yaml`
```
k8sServiceHost: 10.0.0.100
k8sServicePort: 6443

kubeProxyReplacement: true
ipam:
  mode: cluster-pool
  operator:
    clusterPoolIPv4PodCIDRList: ["192.168.0.0/16"] # To avoid overlap with IP of k8s nodes.


hubble:
  enabled: true
  observe:
    traceSummary: true
  metrics:
    enableOpenMetrics: true
    enabled:
      - dns
      - drop
      - tcp
      - flow
      - port-distribution
      - icmp
      - "httpV2:exemplars=true;labelsContext=source_ip,source_namespace,source_workload,destination_ip,destination_namespace,destination_workload,traffic_direction"
  
  relay:
    enabled: true
    extraEnv:
      - name: OTEL_EXPORTER_OTLP_ENDPOINT
        value: "http://tempo-distributor.monitoring.svc.cluster.local:4317"
      - name: OTEL_EXPORTER_OTLP_INSECURE
        value: "true"
  ui:
    enabled: true
  
  
  tracing:
    enabled: true
    backend: opentelemetry
    otlp:
      endpoint: "http://tempo-distributor.monitoring.svc.cluster.local:4317"
      insecure: true

prometheus:
  enabled: true
operator:
  prometheus:
    enabled: true


gatewayAPI:
  enabled: true
  gatewayClassName: "cilium"



dns:
  proxy:
    enable: true
```