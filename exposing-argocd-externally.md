# Exposing Argo CD Externally - Configuration Guide

This guide covers different options for accessing Argo CD over a web browser without port forwarding.

## Option 1: Change Service to LoadBalancer (Quickest)

This creates an AWS Network Load Balancer automatically.

### Steps:

```bash
# Patch the service to LoadBalancer type
kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "LoadBalancer"}}'

# Wait a minute, then get the load balancer URL
kubectl get svc argocd-server -n argocd
```

You'll see an `EXTERNAL-IP` with an AWS hostname like `a1b2c3d4...elb.amazonaws.com`. 

Access Argo CD at:
```
https://a1b2c3d4...elb.amazonaws.com
```

### Pros:
- Quick and simple
- Works immediately

### Cons:
- Creates a separate NLB (~$16/month)
- Uses self-signed certificate
- No custom domain

---

## Option 2: Use ALB Ingress (Recommended for Production)

Create an Ingress to use the AWS Application Load Balancer.

### Create Ingress Resource

**File: `argocd-ingress.yaml`**

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: argocd-server
  namespace: argocd
  annotations:
    # ALB Configuration
    alb.ingress.kubernetes.io/scheme: internet-facing  # Use 'internal' for private access
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/backend-protocol: HTTPS  # Argo CD uses HTTPS
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTP": 80}, {"HTTPS": 443}]'
    alb.ingress.kubernetes.io/ssl-redirect: '443'
    
    # Health check (Argo CD specific)
    alb.ingress.kubernetes.io/healthcheck-path: /healthz
    alb.ingress.kubernetes.io/healthcheck-protocol: HTTPS
    alb.ingress.kubernetes.io/healthcheck-interval-seconds: '15'
    alb.ingress.kubernetes.io/backend-protocol-version: HTTP1
    
    # Optional: Use ACM certificate for HTTPS
    # alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:us-east-2:ACCOUNT:certificate/CERT-ID
    
    # Optional: Group with your app's ALB to save costs
    # alb.ingress.kubernetes.io/group.name: shared-alb
spec:
  ingressClassName: alb
  rules:
    - host: argocd.yourdomain.com  # Replace with your domain, or remove for ALB DNS
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: argocd-server
                port:
                  number: 443  # Argo CD HTTPS port
```

### Apply the Ingress:

```bash
kubectl apply -f argocd-ingress.yaml

# Wait for ALB to provision (2-3 minutes)
kubectl get ingress argocd-server -n argocd -w
```

### Get the ALB URL:

```bash
kubectl get ingress argocd-server -n argocd -o jsonpath='{.status.loadBalancer.ingress[0].hostname}'
```

### Pros:
- Production-ready
- Can share ALB with other services (cost-efficient)
- Supports custom domains and SSL certificates
- Better health checking

### Cons:
- Takes 2-3 minutes to provision
- Slightly more complex configuration

---

## Option 3: Add DNS with Route53

Once you have the Load Balancer hostname, create a DNS record.

### Using AWS Console:

1. Go to **Route53** → **Hosted Zones** → Your domain
2. Click **Create Record**
3. **Record name:** `argocd` (creates `argocd.yourdomain.com`)
4. **Record type:** A - Alias
5. **Route traffic to:** Alias to Application Load Balancer
6. **Region:** us-east-2 (or your region)
7. Choose your ALB from the dropdown
8. Click **Create records**

### Using AWS CLI:

```bash
# Get your hosted zone ID
aws route53 list-hosted-zones

# Get your ALB DNS name
ALB_DNS=$(kubectl get ingress argocd-server -n argocd -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')

# Get ALB Hosted Zone ID (varies by region)
# us-east-2: Z3AADJGX6KTTL2
# us-east-1: Z35SXDOTRQ7X7K
# us-west-2: Z1H1FL5HABSF5

# Create the record (replace values)
aws route53 change-resource-record-sets \
  --hosted-zone-id Z1234567890ABC \
  --change-batch '{
    "Changes": [{
      "Action": "CREATE",
      "ResourceRecordSet": {
        "Name": "argocd.yourdomain.com",
        "Type": "A",
        "AliasTarget": {
          "HostedZoneId": "Z3AADJGX6KTTL2",
          "DNSName": "'$ALB_DNS'",
          "EvaluateTargetHealth": false
        }
      }
    }]
  }'
```

### ALB Hosted Zone IDs by Region:

| Region | Hosted Zone ID |
|--------|----------------|
| us-east-1 | Z35SXDOTRQ7X7K |
| us-east-2 | Z3AADJGX6KTTL2 |
| us-west-1 | Z368ELLRRE2KJ0 |
| us-west-2 | Z1H1FL5HABSF5 |

[Full list](https://docs.aws.amazon.com/general/latest/gr/elb.html)

---

## Option 4: Add SSL Certificate (Secure HTTPS)

### Request ACM Certificate via AWS Console:

1. Go to **AWS Certificate Manager (ACM)**
2. Click **Request certificate**
3. Choose **Request a public certificate**
4. **Domain name:** `argocd.yourdomain.com`
5. **Validation method:** DNS validation (recommended)
6. Click **Request**
7. Click **Create records in Route 53** to auto-validate

### Request ACM Certificate via AWS CLI:

```bash
# Request certificate
aws acm request-certificate \
  --domain-name argocd.yourdomain.com \
  --validation-method DNS \
  --region us-east-2

# Note the CertificateArn from the output
```

### Update Ingress with Certificate:

Once the certificate is validated, update your `argocd-ingress.yaml`:

```yaml
metadata:
  annotations:
    alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:us-east-2:ACCOUNT:certificate/CERT-ID
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTP": 80}, {"HTTPS": 443}]'
    alb.ingress.kubernetes.io/ssl-redirect: '443'
```

Apply the changes:

```bash
kubectl apply -f argocd-ingress.yaml
```

---

## Security Considerations

### For Production:

- **Private Access:** Use `scheme: internal` for VPN-only access
- **SSL Certificate:** Always use ACM certificate for proper HTTPS
- **Authentication:** Consider adding AWS Cognito or OAuth integration
- **IP Whitelisting:** Add source IP restrictions in ALB security group
- **WAF:** Consider AWS WAF for additional protection

### For Development/Testing:

- `scheme: internet-facing` is acceptable
- Can use ALB DNS directly without custom domain
- Self-signed cert warning is acceptable temporarily

---

## Cost Comparison

| Option | Monthly Cost | Notes |
|--------|-------------|-------|
| LoadBalancer (NLB) | ~$16 + data transfer | Dedicated NLB per service |
| Ingress (ALB) | ~$16-22 + data transfer | Can be shared across services |
| Shared ALB (group.name) | Share costs | Most cost-efficient for multiple services |
| ACM Certificate | FREE | No cost for public certificates |
| Route53 Hosted Zone | $0.50/month | Plus $0.40 per million queries |

**Cost Optimization Tip:** Use `alb.ingress.kubernetes.io/group.name` annotation to share a single ALB across multiple services (Argo CD + your applications).

---

## Recommended Approach by Environment

### Development/Testing
**Option 1 (LoadBalancer)** - Quick and simple
- No domain required
- Access via AWS-provided hostname
- Self-signed certificate acceptable

### Staging
**Option 2 (ALB Ingress)** - More production-like
- Can share ALB with other staging services
- Optional custom domain
- Self-signed cert or basic ACM cert

### Production
**Options 2 + 3 + 4 (Full Setup)** - Complete solution
- ALB Ingress with custom domain
- Route53 DNS records
- ACM SSL certificate
- Internal scheme (VPN access)
- Proper security configurations

---

## Complete Production Setup Example

Here's a complete example for production:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: argocd-server
  namespace: argocd
  annotations:
    # ALB Configuration
    alb.ingress.kubernetes.io/scheme: internal  # VPN access only
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/backend-protocol: HTTPS
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTPS": 443}]'
    alb.ingress.kubernetes.io/backend-protocol-version: HTTP1
    
    # SSL Certificate
    alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:us-east-2:123456789012:certificate/abc-123
    
    # Health Checks
    alb.ingress.kubernetes.io/healthcheck-path: /healthz
    alb.ingress.kubernetes.io/healthcheck-protocol: HTTPS
    alb.ingress.kubernetes.io/healthcheck-interval-seconds: '15'
    alb.ingress.kubernetes.io/healthy-threshold-count: '2'
    alb.ingress.kubernetes.io/unhealthy-threshold-count: '2'
    
    # Cost Optimization - Share ALB
    alb.ingress.kubernetes.io/group.name: internal-services
    alb.ingress.kubernetes.io/group.order: '10'
    
    # Security
    alb.ingress.kubernetes.io/ssl-policy: ELBSecurityPolicy-TLS-1-2-2017-01
spec:
  ingressClassName: alb
  rules:
    - host: argocd.internal.yourdomain.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: argocd-server
                port:
                  number: 443
```

---

## Troubleshooting

### ALB Not Creating
```bash
# Check controller logs
kubectl logs -n kube-system deployment/aws-load-balancer-controller --tail=100

# Check ingress events
kubectl describe ingress argocd-server -n argocd
```

### Can't Access via DNS
- Verify DNS propagation: `nslookup argocd.yourdomain.com`
- Check Route53 record is pointing to correct ALB
- Verify security groups allow traffic on ports 80/443

### Certificate Issues
- Ensure certificate status is "Issued" in ACM
- Verify certificate is in the same region as your ALB
- Check certificate ARN is correct in Ingress annotations

### Health Check Failing
- Argo CD health check path is `/healthz`
- Health check must use HTTPS protocol
- Backend protocol must be HTTPS

---

## Getting Started Checklist

- [ ] Choose your approach (LoadBalancer, ALB, or both)
- [ ] Create Ingress resource if using ALB
- [ ] Request ACM certificate (if using custom domain)
- [ ] Create Route53 DNS record
- [ ] Update Ingress with certificate ARN
- [ ] Test access via browser
- [ ] Configure security settings for production
- [ ] Set up monitoring and alerts

---

## Additional Resources

- [AWS Load Balancer Controller Documentation](https://kubernetes-sigs.github.io/aws-load-balancer-controller/)
- [Argo CD Ingress Documentation](https://argo-cd.readthedocs.io/en/stable/operator-manual/ingress/)
- [AWS ALB Ingress Annotations](https://kubernetes-sigs.github.io/aws-load-balancer-controller/v2.4/guide/ingress/annotations/)
- [AWS Certificate Manager](https://aws.amazon.com/certificate-manager/)
- [Amazon Route 53](https://aws.amazon.com/route53/)
