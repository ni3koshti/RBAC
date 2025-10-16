#!/bin/bash

# Debug script for email-integrations-oncall RBAC issues
# Replace <CLUSTER_CONTEXT> with your actual cluster context

CLUSTER_CONTEXT="<CLUSTER_CONTEXT>"
NAMESPACE="support-inbound-email"
GROUP_NAME="email-integrations-oncall"

echo "=== Debugging RBAC for email-integrations-oncall ==="
echo

echo "1. Check if oncall ClusterRole exists:"
kubectl get clusterrole oncall --context=$CLUSTER_CONTEXT

echo
echo "2. Check if namespace exists:"
kubectl get namespace $NAMESPACE --context=$CLUSTER_CONTEXT

echo
echo "3. Check if RoleBinding exists:"
kubectl get rolebinding $GROUP_NAME -n $NAMESPACE --context=$CLUSTER_CONTEXT

echo
echo "4. Check RoleBinding details:"
kubectl describe rolebinding $GROUP_NAME -n $NAMESPACE --context=$CLUSTER_CONTEXT

echo
echo "5. Check if custom namespace rules Role exists:"
kubectl get role ${GROUP_NAME}-namespace-rules -n $NAMESPACE --context=$CLUSTER_CONTEXT

echo
echo "6. Check Role details:"
kubectl describe role ${GROUP_NAME}-namespace-rules -n $NAMESPACE --context=$CLUSTER_CONTEXT

echo
echo "7. Check your current user/groups:"
kubectl auth whoami --context=$CLUSTER_CONTEXT

echo
echo "8. Test permissions:"
echo "Can create pods/exec in $NAMESPACE:"
kubectl auth can-i create pods/exec -n $NAMESPACE --context=$CLUSTER_CONTEXT

echo "Can delete pods in $NAMESPACE:"
kubectl auth can-i delete pods -n $NAMESPACE --context=$CLUSTER_CONTEXT

echo "Can scale deployments in $NAMESPACE:"
kubectl auth can-i patch deployments -n $NAMESPACE --context=$CLUSTER_CONTEXT


