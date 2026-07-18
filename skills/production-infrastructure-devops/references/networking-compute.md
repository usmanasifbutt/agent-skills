# Networking & Compute (VPC, Subnets, Security Groups, NAT, EKS, Autoscaling)

## VPC & subnet design

- **Multi-AZ, four-tier subnet layout** computed by a CIDR-decomposer helper module that slices the
  account CIDR into parallel tiers by fixed offsets: **private** (workloads), **public** (ingress),
  **database**, **cache/elasticache** — each tagged with its `tier`. Built on the upstream VPC
  module.
- `enable_dns_hostnames = true`. An "extended" computation covers extra subnets across a long AZ
  list so the VPC can grow without renumbering.
- **Kubernetes/ingress-aware subnet tags:** public subnets `kubernetes.io/role/elb=1`, private
  `kubernetes.io/role/internal-elb=1`, per-cluster `kubernetes.io/cluster/<name>=shared`, and
  (if used) `karpenter.sh/discovery/<name>=true`.
- **Why:** tiering isolates blast radius (a compromised public tier can't reach DB subnets) and lets
  each tier have its own routing/ACLs.
- **Common mistake:** flat subnets, or DB subnets sharing routes with public subnets.

## NAT

```hcl
enable_nat_gateway     = true
single_nat_gateway     = false
one_nat_gateway_per_az = true
reuse_nat_ips          = true          # pre-allocated, stable egress EIPs
external_nat_ip_ids    = [for e in aws_eip.nat : e.id]
```
- **One NAT gateway per AZ** for HA, with **pre-allocated stable EIPs** so egress IPs can be
  allowlisted by partners.
- **Common mistake:** a single shared NAT gateway — a SPOF, and a moving egress IP that breaks
  partner allowlists.

## Route tables, security groups, defaults

- **Never rely on defaults:** `manage_default_security_group/route_table/network_acl = false`
  (leave them empty). Create the DB subnet group and DB route table explicitly.
- **DB internet/NAT routes gated behind explicit booleans that default closed**
  (`database_publicly_accessible`, `enable_database_internet_access` = false).
- **Security groups via a dedicated (upstream) SG module**, not scattered `aws_security_group_rule`
  resources — consumers pass structured rule inputs; rules stay reviewable and least-privilege.
- **Common mistake:** leaving default SG/route-table/NACL managed; opening DB subnets to the
  internet by default rather than behind an explicit, default-false flag.

## Load balancing

- **Load balancing lives at the Kubernetes layer:** rather than hand-rolling ALB/NLB resources,
  the **AWS Load Balancer Controller** (granted a scoped OIDC role) provisions ALBs/NLBs from
  Ingress/Service objects. Terraform owns the IAM + the subnet tags that make that possible.
- Ingress details (ALB groups, ACM certs, path routing, traffic "formations") are covered in
  `kubernetes-helm-gitops.md`.
- **Common mistake:** duplicating LB definitions in both Terraform and Kubernetes; giving every
  service its own ALB instead of a shared group.

## EKS

- **One cluster == one Helmfile environment**, identified by account + region + name.
- **Workload identity via IRSA:** service accounts annotated `eks.amazonaws.com/role-arn`; ArgoCD
  and controllers assume per-cluster/per-service roles. **No long-lived AWS keys in pods.**
- **On-the-fly kubeconfig:** CLI access generated via `aws eks update-kubeconfig` in a temporary
  context (torn down after) — no reliance on hardcoded/local context names.
- **Node placement:** `nodeSelector` + taint `tolerations` keyed on a node-group label pin web vs
  worker pods to dedicated node groups; pod anti-affinity spreads across AZs.
- **Common mistake:** AWS access keys in pod env instead of IRSA; a `nodeSelector` without the
  matching `toleration` so pods never schedule onto tainted nodes.

## Node groups & autoscaling

- **Declarative node-group model:** node groups are a map input; the module does a `setproduct` of
  groups × AZs to expand `multi_az` groups into one ASG per AZ (auto-resolving per-AZ subnets). Each
  group carries `asg_min/desired/max`.
- **Architecture & GPU auto-detection:** the module inspects the instance type to pick the right AMI
  (arm64/amd64/GPU) and injects GPU labels/taints automatically — the caller just names an instance
  type.
- **Two autoscaler strategies, switchable:** classic **cluster-autoscaler** (ASG tags + node-template
  hints) or **Karpenter** (subnet/SG discovery tags + a minimal always-on "general" node group with
  a `high_availability` toggle). A flag chooses one exclusively.
- **Safe node defaults:** encrypted root volumes, **IMDSv2** enforced, optional warm pools,
  namespaced taints/labels, CSI policies attached only when storage is enabled.
- **Pod-level autoscaling (HPA)** is configured as chart values per workload (`min/max`, often
  queue-aware). Because pods autoscale, **GitOps must ignore the `replicas` field** (see the
  GitOps reference) or ArgoCD and the HPA fight.
- **Common mistake:** hard-coding one instance architecture (breaks arm/GPU); per-AZ ASGs sharing a
  single min/max so capacity can't balance across zones; managing `replicas` in git while an HPA is
  active.
