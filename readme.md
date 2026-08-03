# AWS Load Balancer Controller — Setup on EKS

> Prerequisite: EKS cluster `roboshop` is running and `kubectl` / `eksctl` are pointed at it (region `us-east-1`).
> This is the shared foundation — it's identical whether you expose apps with Ingress or Gateway API.

## 1. Associate the OIDC provider

Lets the controller's ServiceAccount assume an IAM role (IRSA).

```bash
eksctl utils associate-iam-oidc-provider \
  --region us-east-1 \
  --cluster roboshop \
  --approve
```

Verify: `aws eks describe-cluster --name roboshop --query "cluster.identity.oidc.issuer" --output text`

## 2. Download the IAM policy

The permissions the controller needs to manage ELB resources. Pin the version to the controller you'll install.

```bash
curl -o iam-policy.json \
  https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v3.4.2/docs/install/iam_policy.json
```

## 3. Create the IAM policy

```bash
aws iam create-policy \
  --policy-name AWSLoadBalancerControllerIAMPolicy \
  --policy-document file://iam-policy.json
```

Verify: note the returned `Arn` — it feeds step 4.

## 4. Create the ServiceAccount (IRSA)

Binds the `aws-load-balancer-controller` SA to the IAM policy above.

```bash
eksctl create iamserviceaccount \
  --cluster=roboshop \
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --attach-policy-arn=arn:aws:iam::160885265516:policy/AWSLoadBalancerControllerIAMPolicy \
  --override-existing-serviceaccounts \
  --region us-east-1 \
  --approve
```

Verify: `kubectl -n kube-system get sa aws-load-balancer-controller -o yaml | grep eks.amazonaws.com/role-arn`

## 5. Install the AWS Load Balancer Controller (Helm)

```bash
helm repo add eks https://aws.github.io/eks-charts
helm repo update

helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=roboshop \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller
```

Verify: `kubectl -n kube-system get deploy aws-load-balancer-controller` → `2/2` Ready.

## Ingress setup — two apps behind one ALB

Two demo apps, each with a Deployment + Service + Ingress.

Both Ingresses carry `alb.ingress.kubernetes.io/group.name: roboshop`. That's **IngressGroup** — it merges both Ingresses onto a *single shared ALB* with host-based routing, instead of one ALB per Ingress.

### Apply

```bash
kubectl apply -f ingress/app1/manifest.yaml
kubectl apply -f ingress/app2/manifest.yaml
```

Verify pods and services are up:

```bash
kubectl get deploy,svc,pods -l app=app-1
kubectl get deploy,svc,pods -l app=app-2
```

### Confirm the shared ALB

```bash
kubectl get ingress
```

Both `app-1` and `app-2` should show the **same ADDRESS** — one ALB, two host rules. If you see two different addresses, the grouping didn't take (see disadvantage #4).

Verify the `alb` IngressClass exists (the controller ignores the Ingress silently if it's missing):

```bash
kubectl get ingressclass
```

### DNS

Point both hosts at the shared ALB — a Route 53 alias record per host to the ALB DNS from `kubectl get ingress`, or ExternalDNS from the `host` field:

- `*.daws90s.shop` → ALB

### Test

```bash
curl -I https://app1.daws90s.shop
curl -I https://app2.daws90s.shop
```

Both should return `HTTP/1.1 200`, served by the same load balancer.

---

## Disadvantages of Ingress

These aren't abstract — every one is visible in the two-app setup above.

| # | Disadvantage | Where you see it in this setup |
|---|---|---|
| 1 | **Duplicated infra config** | The same six annotations (`scheme`, `certificate-arn`, `target-type`, `tags`, `listen-ports`, `group.name`) are copy-pasted into *both* `app-1` and `app-2`. Rotate the cert and you edit every Ingress. No single source of truth. |
| 2 | **Annotations are unvalidated** | The `listen-ports` JSON string and the `certificate-arn` are opaque to the API server. Typo either and `kubectl apply` still succeeds — it fails **silently at runtime**, not at apply. |
| 3 | **Blurred ownership** | Operator concerns (scheme, cert ARN, tags) sit in the same object the app team owns (host, path, backend). Both app teams must know and paste the operator's cert ARN into their own file. |
| 4 | **Grouping is annotation-magic** | The shared ALB exists only because both files happen to carry the identical `group.name: roboshop` string. Nothing enforces it — misspell it in one file and that app silently spins up a **second ALB** (a real cost surprise). |
| 5 | **Vendor lock-in** | All six annotations are AWS-only. Move EKS → AKS/GKE and you rewrite them in *every* app. Because they're app-facing, your app teams get pulled into an infra migration. |
| 6 | **HTTP/HTTPS only + sparse status** | Ingress is L7 web traffic only. And a wrong `backend.service.name` just 503s — nothing on the Ingress object tells you why. |

# Gateway

```bash
# Standard Gateway API CRDs (REQUIRED), built against v1.5.0
kubectl apply --server-side=true -f \
  https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.5.0/standard-install.yaml

# LBC AWS-specific CRDs (LoadBalancerConfiguration, TargetGroupConfiguration, ListenerRuleConfiguration)
kubectl apply -f \
  https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/refs/heads/main/config/crd/gateway/gateway-crds.yaml
```

The controller enables its Gateway reconcilers only at startup, based on whether the Gateway CRDs exist. If the controller was installed before these CRDs (which is the normal order — controller in the Ingress session, CRDs now), it isn't watching Gateways yet. Restart it once so it re-detects them:

```bash
kubectl -n kube-system rollout restart deploy aws-load-balancer-controller
kubectl -n kube-system rollout status  deploy aws-load-balancer-controller
```