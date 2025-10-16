# RBAC Issue Analysis Report: email-integrations-oncall

**Date:** October 16, 2025  
**Repository:** zendesk/k8s-rbac-policies  
**Issue:** `email-integrations-oncall` role not working with `kubectl auth can-i` commands  

## Executive Summary

The `email-integrations-oncall` RBAC role was successfully deployed but customers experienced permission errors when using `kubectl --as=admin --as-group=email-integrations-oncall` commands. The root cause was missing impersonation ClusterRole resources that should have been auto-generated but were prevented by an incorrect `no_cluster_role: true` flag in the original definition.

## Issue Timeline

### Initial Problem Report
- **Issue:** Customer getting "Error from server (Forbidden): unknown" when using kubectl impersonation
- **Symptoms:** 
  - `kubectl auth can-i` commands failing
  - Server-side memcache errors: `memcache.go:256] "Unhandled Error" err"couldnt get current server API group list: unknown"`
  - Role appears deployed successfully in manifest files

### Investigation Process

#### 1. RBAC Structure Analysis
- Confirmed `email-integrations-oncall` role exists in `/definitions/oncall.yaml` (lines 462-470)
- Verified deployment in generated `/kubernetes/manifest.yaml`
- Role properly configured with:
  - `namespace_role: oncall`
  - `namespaces: [support-inbound-email]`
  - `clusters: [podded]`
  - Custom namespace rules for `pods/exec` permissions

#### 2. kubectl Syntax Clarification
**Question:** How to use subresources with kubectl?

**Answer:** Two valid syntaxes:
```bash
# Method 1: Using --subresource flag
kubectl auth can-i create pods --subresource=exec --namespace=support-inbound-email --as=admin --as-group=email-integrations-oncall

# Method 2: Using resource/subresource format
kubectl auth can-i create pods/exec --namespace=support-inbound-email --as=admin --as-group=email-integrations-oncall
```

#### 3. Server-Side Error Analysis
Memcache errors indicated API server discovery issues, but these were secondary to the underlying permission problem.

#### 4. Root Cause Discovery
Team member identified the real issue: **Missing impersonation ClusterRole**

## Root Cause Analysis

### The Problem
The `email-integrations-oncall` role was originally defined with `no_cluster_role: true` in commit `7d3b65a6`, which prevented automatic generation of essential impersonation resources:

- ClusterRole: `impersonate-email-integrations-oncall`
- ClusterRoleBinding: `okta-email-integrations-oncall`

### Historical Context

#### Commit 7d3b65a6 (Original Addition)
```yaml
---
name: email-integrations-oncall
namespace_role: oncall
namespaces:
  - support-inbound-email
clusters: [podded]
no_cluster_role: true  # ❌ This was the problem!
namespace_rules:
- apiGroups: [""]
  resources: ["pods/exec"]
  verbs: ["create"]
```

#### Why This Was Wrong
The `no_cluster_role: true` flag should only be used when:
1. Reusing an existing ClusterRole from another definition
2. The role doesn't need impersonation capabilities

In this case, `email-integrations-oncall` needed its own impersonation ClusterRole to work with `kubectl --as` commands.

### Comparison with Working Roles

#### email-processing-oncall (Line 434 - Creates ClusterRole)
```yaml
---
name: email-processing-oncall
namespace_role: oncall
namespaces:
  - classic
  - email-image-proxy
clusters: [podded]
# No no_cluster_role flag = auto-generates impersonation resources ✅
```

#### email-transit-oncall (Line 449 - Reuses ClusterRole)
```yaml
---
name: email-transit-oncall
namespace_role: oncall
namespaces:
  - halon-mta
  - support-inbound-email
clusters: [podded]
no_cluster_role: true # ✅ Correct usage - reuses existing ClusterRole
namespace_rules:
- apiGroups: [""]
  resources: ["pods/exec"]
  verbs: ["create"]
```

## Resolution Implementation

### Phase 1: Manual Fix (Commit 724f3d80)
Team member manually added missing impersonation resources:

```yaml
# Added ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: impersonate-email-integrations-oncall
rules:
- apiGroups: [""]
  resources: ["users"]
  verbs: ["impersonate"]
- apiGroups: [""]
  resources: ["groups"]
  verbs: ["impersonate"]

# Added ClusterRoleBinding  
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: okta-email-integrations-oncall
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: impersonate-email-integrations-oncall
subjects:
- apiGroup: rbac.authorization.k8s.io
  kind: Group
  name: okta:email-integrations-oncall
```

### Phase 2: Root Cause Fix (Current State)
Removed `no_cluster_role: true` from the definition:

```yaml
---
name: email-integrations-oncall
namespace_role: oncall
namespaces:
  - support-inbound-email
clusters: [podded]
# ✅ no_cluster_role flag removed - will auto-generate impersonation resources
namespace_rules:
- apiGroups: [""]
  resources: ["pods/exec"]
  verbs: ["create"]
```

## Technical Deep Dive

### RBAC Generation Process
1. **Definition Processing:** Ruby ERB template `/templates/role.yaml.erb` processes `/definitions/oncall.yaml`
2. **ClusterRole Generation:** When `no_cluster_role: true` is absent, template generates:
   - ClusterRole for impersonation
   - ClusterRoleBinding linking Okta group to ClusterRole
3. **Manifest Generation:** Resources written to `/kubernetes/manifest.yaml`
4. **Deployment:** Spinnaker deploys to all specified clusters

### Why Impersonation ClusterRoles Are Required
When using `kubectl --as=user --as-group=group`:
1. Kubernetes checks if requesting user can impersonate the target group
2. Requires ClusterRole with `impersonate` verbs on `users` and `groups` resources
3. ClusterRoleBinding connects the Okta group to this ClusterRole

### Email Role Architecture
```
email-processing-oncall     (Creates ClusterRole)
├── Classic namespace
└── email-image-proxy namespace

email-transit-oncall       (Reuses ClusterRole - line 434)
├── Classic namespace  
├── email-storage-service namespace
└── [multiple other namespaces]

email-transit-oncall       (Reuses ClusterRole - line 449)
├── halon-mta namespace
└── support-inbound-email namespace

email-integrations-oncall  (Now creates own ClusterRole)
└── support-inbound-email namespace
```

## Commands Used in Investigation

### Git History Analysis
```bash
# Find email-integrations related commits
git log --oneline --grep="email-integrations" --grep="email integrations" -i

# Show original addition commit
git show 7d3b65a6

# Show manual fix commit  
git show 724f3d80

# Track definition file changes
git log --oneline --follow -p -- definitions/oncall.yaml | grep -A10 -B5 "email-integrations-oncall"
```

### RBAC Structure Inspection
```bash
# Check current definition
grep -A10 -B5 "email-integrations-oncall" definitions/oncall.yaml

# Verify manifest generation
grep -A20 "email-integrations-oncall" kubernetes/manifest.yaml
```

### kubectl Testing Commands
```bash
# Test permissions (both syntaxes work)
kubectl auth can-i create pods/exec --namespace=support-inbound-email --as=admin --as-group=email-integrations-oncall

kubectl auth can-i create pods --subresource=exec --namespace=support-inbound-email --as=admin --as-group=email-integrations-oncall

# Practical usage
kubectl --as=admin --as-group=email-integrations-oncall exec -it <pod-name> -n support-inbound-email -- /bin/bash
```

## Key Files Modified

### `/definitions/oncall.yaml`
- **Lines 462-470:** `email-integrations-oncall` definition
- **Change:** Removed `no_cluster_role: true` flag

### `/kubernetes/manifest.yaml`  
- **Auto-generated:** Contains all RBAC resources deployed to clusters
- **Includes:** RoleBindings and namespace-specific Roles

### Manual Additions (Commit 724f3d80)
Multiple manifest files updated with impersonation resources:
- `/kubernetes/manifests/production/pod13/roles.yml`
- `/kubernetes/manifests/production/pod15/roles.yml`
- [All other podded clusters]

## Lessons Learned

### 1. Understanding `no_cluster_role: true`
- **Use Case:** Only when reusing existing ClusterRole from previous definition
- **Don't Use:** For new roles that need independent impersonation capabilities

### 2. Role Definition Patterns
```yaml
# Pattern 1: Creates ClusterRole (first occurrence)
name: some-oncall
namespace_role: oncall
# no_cluster_role flag omitted

# Pattern 2: Reuses ClusterRole (subsequent occurrences)  
name: some-oncall
namespace_role: oncall
no_cluster_role: true # references first definition's ClusterRole
```

### 3. Testing Impersonation
Always test new oncall roles with:
```bash
kubectl auth can-i create pods/exec --namespace=<target-ns> --as=admin --as-group=<role-name>
```

### 4. Deployment Verification
Check both:
- Definition file correctness
- Generated manifest completeness
- Actual cluster deployment success

## Current Status

### ✅ Issue Resolved
- Customers can now use `kubectl --as=admin --as-group=email-integrations-oncall` commands
- Manual impersonation resources deployed (commit 724f3d80)
- Root cause fixed in definition file (removed `no_cluster_role: true`)

### 🔄 Ongoing Monitoring
- Verify next deployment cycle generates impersonation resources automatically
- Confirm all podded clusters received the manual fix
- Monitor for similar issues with other email-related roles

## Prevention Measures

### 1. Code Review Guidelines
- Verify `no_cluster_role: true` usage is appropriate
- Ensure new oncall roles can impersonate when needed
- Test kubectl commands before deployment

### 2. Deployment Validation
- Add post-deployment verification for impersonation capabilities
- Include kubectl permission tests in CI/CD pipeline

### 3. Documentation Updates
- Document proper use of `no_cluster_role` flag
- Provide kubectl syntax examples for testing
- Create troubleshooting guide for similar issues

## Contact Information

**Issue Reporter:** Customer via support ticket  
**Resolver:** Team member (provided root cause analysis)  
**Documentation:** Generated from investigation session  
**Repository:** zendesk/k8s-rbac-policies  

---

**Report Generated:** October 16, 2025  
**Status:** RESOLVED  
**Priority:** HIGH (Customer Impact)  
**Resolution Time:** Same day (expedited due to customer impact)
