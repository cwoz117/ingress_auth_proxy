# 1. Mutating Admission Webhook in Kubernetes
Kubernetes webhook to intercept and modify resource requests before they hit etcd. Inject whitelists before saving from the webhook. The webhook can maintain the ip addr list and inject it for us.

Still requires us to update our terraform to modify the whitelist though, as the webhook only gets called when we update ingress resources.

# 2. Patching an IP to the whitelist temporarily

```
kubectl get ingress my-ingress -n my-namespace -o=jsonpath='{.metadata.annotations.nginx\.ingress\.kubernetes\.io/whitelist-source-range}' | \
awk '{print $0",203.0.113.6/32"}' | \
xargs -I{} kubectl patch ingress my-ingress -n my-namespace --type='merge' -p '{"metadata": {"annotations": {"nginx.ingress.kubernetes.io/whitelist-source-range": "{}"}}}'

```
This grabs the old ip whitelist and adds the new IP, issues when dealing w/ multiple additions (if nesting is not enforced essentially)


## 3. ArgoCD annotations to ignore Syncing during this time
```
kubectl annotate ingress my-ingress -n my-namespace \
  argocd.argoproj.io/compare-options="IgnoreExtraneous"
```

### OR we can ignore specific line-items:

Argo manifest updates tho
```
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-argo-app
spec:
  ignoreDifferences:
  - group: networking.k8s.io
    kind: Ingress
    jsonPointers:
    - /metadata/annotations/nginx.ingress.kubernetes.io~1whitelist-source-range
```


## 4. Remove loadbalancerSourceRange
IP Whitelists can exist at the ingress level, but the loadbalancer service needs to be opened up (Security concerns?)

so-called global IP's (office cidr range for ex) can exist in a config map and be pulled with any additional ip's added for clients
