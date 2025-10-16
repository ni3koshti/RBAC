# Kubernetes RBAC Use Case: Broken ClusterRole Dependency Chain

**Category:** Infrastructure Configuration Management  
**Severity:** High (Customer Impact)  
**Type:** Post-Mortem Analysis & Best Practices Guide  

## Executive Summary

A Kubernetes RBAC role was successfully deployed but users experienced permission errors when using `kubectl` impersonation commands. Investigation revealed a broken dependency chain where a role was configured to reuse another role's ClusterRole, but organizational changes to the configuration file broke this dependency while leaving the reuse flag intact.

## Problem Statement

### Symptoms
- Users receiving "Error from server (Forbidden): unknown" when using kubectl impersonation
- `kubectl auth can-i` commands failing for specific role
- Server-side API discovery errors in logs
- Role appears correctly deployed in manifest files

### Impact
- Customer operations blocked
- Manual intervention required for critical tasks
- Support escalation required

## Technical Background

### RBAC Architecture Overview
The organization uses a template-based RBAC generation system where:
1. YAML definitions describe roles and permissions
2. Ruby ERB templates process definitions into Kubernetes resources
3. Generated manifests are deployed across multiple clusters
4. Roles support impersonation via dedicated ClusterRoles

### Role Definition Patterns
```yaml
# Pattern 1: Creates ClusterRole (first occurrence)
name: service-team-oncall
namespace_role: oncall
namespaces: [app-namespace-1, app-namespace-2]
clusters: [cluster-group]
# Omitting no_cluster_role flag auto-generates impersonation resources

# Pattern 2: Reuses ClusterRole (same role name, different config)
name: service-team-oncall
namespace_role: oncall
namespaces: [different-namespace]
clusters: [cluster-group]
no_cluster_role: true # Reuses ClusterRole from first definition
```

## Root Cause Analysis

### Initial Configuration
```yaml
# Role A - Creates ClusterRole
name: transit-service-oncall
namespace_role: oncall
namespaces: [core-services, message-processing]
clusters: [production-clusters]

# Role B - Intended to reuse Role A's ClusterRole
name: integration-service-oncall
namespace_role: oncall
namespaces: [integration-layer]
clusters: [production-clusters]
no_cluster_role: true # Intended to reuse transit-service-oncall ClusterRole
```

### The Problem
1. **Role B** was configured with `no_cluster_role: true` to reuse **Role A's** ClusterRole
2. During configuration reorganization, role definitions were resequenced
3. The dependency chain broke, but `no_cluster_role: true` remained
4. **Role B** could no longer find the ClusterRole it was supposed to reuse
5. No impersonation resources were generated for **Role B**

### Missing Resources
Without proper ClusterRole generation, the following resources were missing:
```yaml
# Missing ClusterRole for impersonation
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: impersonate-integration-service-oncall
rules:
- apiGroups: [""]
  resources: ["users", "groups"]
  verbs: ["impersonate"]

# Missing ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: auth-integration-service-oncall
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: impersonate-integration-service-oncall
subjects:
- apiGroup: rbac.authorization.k8s.io
  kind: Group
  name: auth-provider:integration-service-oncall
```

## Resolution Implementation

### Phase 1: Immediate Fix
Manual addition of missing impersonation resources to restore service:

```yaml
# Manually added ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: impersonate-integration-service-oncall
rules:
- apiGroups: [""]
  resources: ["users"]
  verbs: ["impersonate"]
- apiGroups: [""]
  resources: ["groups"]  
  verbs: ["impersonate"]

# Manually added ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: auth-integration-service-oncall
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: impersonate-integration-service-oncall
subjects:
- apiGroup: rbac.authorization.k8s.io
  kind: Group
  name: auth-provider:integration-service-oncall
```

### Phase 2: Root Cause Fix
Updated the role definition to create its own ClusterRole:

```yaml
# Fixed configuration - creates own ClusterRole
name: integration-service-oncall
namespace_role: oncall
namespaces: [integration-layer]
clusters: [production-clusters]
# Removed no_cluster_role: true flag
namespace_rules:
- apiGroups: [""]
  resources: ["pods/exec"]
  verbs: ["create"]
```

## Verification Process

### Testing Impersonation
```bash
# Test permission check
kubectl auth can-i create pods/exec \
  --namespace=integration-layer \
  --as=admin \
  --as-group=integration-service-oncall

# Test actual usage  
kubectl --as=admin --as-group=integration-service-oncall \
  exec -it <pod-name> -n integration-layer -- /bin/bash
```

### Deployment Validation
```bash
# Verify ClusterRole exists
kubectl get clusterrole impersonate-integration-service-oncall

# Verify ClusterRoleBinding exists  
kubectl get clusterrolebinding auth-integration-service-oncall

# Check role permissions
kubectl describe clusterrole impersonate-integration-service-oncall
```

## Best Practices & Prevention

### 1. Role Definition Guidelines
- **Same Role Name:** Use `no_cluster_role: true` only for subsequent definitions of the same role name
- **Different Role Names:** Always omit `no_cluster_role` flag to create independent ClusterRoles
- **Documentation:** Comment the purpose when using `no_cluster_role: true`

### 2. Change Management Process
```yaml
# ✅ Good: Clear dependency documentation
name: service-team-oncall
namespace_role: oncall
namespaces: [additional-namespace]
clusters: [production-clusters]
no_cluster_role: true # Reuses ClusterRole from previous service-team-oncall definition
```

### 3. Validation Checklist
- [ ] Verify `no_cluster_role: true` references exist before the current definition
- [ ] Test impersonation commands before deployment
- [ ] Validate generated manifests contain expected ClusterRoles
- [ ] Document cross-role dependencies explicitly

### 4. Monitoring & Alerting
- Monitor kubectl authentication failures
- Alert on missing ClusterRole/ClusterRoleBinding resources
- Track RBAC-related support tickets

## Key Takeaways

### Technical Lessons
1. **Dependency Order Matters:** Role definitions with `no_cluster_role: true` depend on specific ordering
2. **Cross-Role Dependencies Are Fragile:** Avoid reusing ClusterRoles across different role names
3. **Testing Is Critical:** Always test impersonation after RBAC changes

### Process Improvements  
1. **Configuration Reviews:** Require peer review for RBAC changes
2. **Automated Testing:** Include kubectl permission tests in CI/CD pipeline
3. **Documentation:** Maintain clear dependency maps for complex role structures

### Operational Guidelines
1. **Immediate Response:** Manual resource creation for urgent customer issues
2. **Root Cause Fix:** Address configuration issues to prevent recurrence  
3. **Monitoring:** Implement proactive detection of similar issues

## Conclusion

This incident highlighted the importance of understanding Kubernetes RBAC dependency chains and the risks of cross-role ClusterRole reuse. The resolution involved both immediate service restoration and systematic configuration fixes to prevent similar issues.

**Key Success Factors:**
- Rapid problem identification through systematic investigation
- Two-phase resolution approach (immediate + root cause)
- Implementation of preventive measures and best practices
- Clear documentation for future reference

---

**Report Classification:** Internal Use  
**Document Type:** Post-Mortem Analysis  
**Audience:** Platform Engineering, DevOps Teams  
**Review Cycle:** Annual or after similar incidents