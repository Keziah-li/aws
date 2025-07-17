# AWS Vault to Kubernetes ConfigMap Migration Guide

## Overview

This guide provides a detailed process for migrating data from AWS Vault (HashiCorp Vault) to Kubernetes ConfigMap. This migration involves extracting secrets from Vault and creating ConfigMap resources in Kubernetes for your applications to consume.

## Table of Contents

1. [Prerequisites](#prerequisites)
2. [Understanding the Architecture](#understanding-the-architecture)
3. [Migration Approaches](#migration-approaches)
4. [Step-by-Step Migration Process](#step-by-step-migration-process)
5. [Verification and Testing](#verification-and-testing)
6. [Best Practices](#best-practices)
7. [Troubleshooting](#troubleshooting)
8. [Security Considerations](#security-considerations)

## Prerequisites

Before starting the migration, ensure you have:

- **AWS Account** with access to Vault
- **Kubernetes cluster** (EKS, GKE, AKS, or on-premises)
- **kubectl** CLI configured for your cluster
- **Vault CLI** installed and configured
- **Helm** (if using Vault Helm charts)
- **jq** for JSON processing
- **base64** utility for encoding/decoding
- Appropriate **IAM permissions** for AWS resources
- **Vault authentication credentials** (root token or service account)

## Understanding the Architecture

### Current State: AWS Vault
```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   Application   │───▶│   AWS Vault     │───▶│   Secret Data   │
│                 │    │                 │    │                 │
└─────────────────┘    └─────────────────┘    └─────────────────┘
```

### Target State: Kubernetes ConfigMap
```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   Application   │───▶│   ConfigMap     │───▶│ Configuration   │
│                 │    │                 │    │     Data        │
└─────────────────┘    └─────────────────┘    └─────────────────┘
```

## Migration Approaches

### Approach 1: Direct Migration (Recommended for Simple Cases)
- Extract secrets directly from Vault
- Create ConfigMaps manually
- Suitable for small number of secrets

### Approach 2: Automated Migration with Scripts
- Use scripts to automate the extraction and creation process
- Suitable for large-scale migrations
- Provides consistency and repeatability

### Approach 3: Using External Secrets Operator
- Keep Vault as the source of truth
- Use External Secrets Operator to sync to ConfigMaps
- Suitable for dynamic configurations

## Step-by-Step Migration Process

### Step 1: Set Up Environment

#### 1.1 Configure Vault Access
```bash
# Set Vault address
export VAULT_ADDR="https://your-vault-server:8200"

# Authenticate with Vault (using token method)
export VAULT_TOKEN="your-vault-token"

# Verify connection
vault status
```

#### 1.2 Configure Kubernetes Access
```bash
# Verify kubectl connection
kubectl cluster-info

# Create namespace for migration (optional)
kubectl create namespace vault-migration
```

### Step 2: Inventory Vault Secrets

#### 2.1 List Available Secret Engines
```bash
# List all secret engines
vault secrets list

# List secrets in a specific path
vault kv list secret/
```

#### 2.2 Export Secret Paths
```bash
# Create a list of all secret paths
vault kv list -format=json secret/ | jq -r '.[]' > secret-paths.txt

# For nested paths, use recursive listing
vault kv list -format=json secret/app1/ | jq -r '.[]' >> secret-paths.txt
```

### Step 3: Extract Secrets from Vault

#### 3.1 Single Secret Extraction
```bash
# Extract a single secret
vault kv get -format=json secret/app1/config | jq -r '.data.data' > app1-config.json

# Example output format:
# {
#   "database_url": "postgres://user:pass@host:5432/db",
#   "api_key": "your-api-key",
#   "debug": "true"
# }
```

#### 3.2 Batch Secret Extraction Script
Create a script `extract-secrets.sh`:

```bash
#!/bin/bash

VAULT_PATH="secret"
OUTPUT_DIR="extracted-secrets"
NAMESPACE="default"

# Create output directory
mkdir -p "$OUTPUT_DIR"

# Function to extract secrets
extract_secret() {
    local secret_path="$1"
    local output_file="$2"
    
    echo "Extracting secret from: $secret_path"
    
    # Get secret data
    vault kv get -format=json "$secret_path" | jq -r '.data.data' > "$output_file"
    
    if [ $? -eq 0 ]; then
        echo "✓ Successfully extracted: $secret_path"
    else
        echo "✗ Failed to extract: $secret_path"
    fi
}

# Read secret paths and extract each one
while IFS= read -r path; do
    # Remove trailing slash if present
    clean_path=$(echo "$path" | sed 's/\/$//')
    
    # Create output filename
    output_file="$OUTPUT_DIR/${clean_path//\//-}.json"
    
    # Extract the secret
    extract_secret "$VAULT_PATH/$clean_path" "$output_file"
    
done < secret-paths.txt

echo "Secret extraction completed. Files saved in: $OUTPUT_DIR"
```

Make the script executable and run it:
```bash
chmod +x extract-secrets.sh
./extract-secrets.sh
```

### Step 4: Create ConfigMaps

#### 4.1 Single ConfigMap Creation
```bash
# Method 1: From JSON file
kubectl create configmap app1-config \
    --from-file=config.json=extracted-secrets/app1-config.json \
    --namespace=default

# Method 2: From individual key-value pairs
kubectl create configmap app1-config \
    --from-literal=database_url="postgres://user:pass@host:5432/db" \
    --from-literal=api_key="your-api-key" \
    --from-literal=debug="true" \
    --namespace=default
```

#### 4.2 Automated ConfigMap Creation Script
Create a script `create-configmaps.sh`:

```bash
#!/bin/bash

EXTRACTED_DIR="extracted-secrets"
NAMESPACE="default"
LABEL_SELECTOR="migrated-from=vault"

# Function to create ConfigMap from JSON
create_configmap_from_json() {
    local json_file="$1"
    local configmap_name="$2"
    
    echo "Creating ConfigMap: $configmap_name"
    
    # Create temporary file for key-value pairs
    temp_file=$(mktemp)
    
    # Convert JSON to key-value pairs
    jq -r 'to_entries[] | "\(.key)=\(.value)"' "$json_file" > "$temp_file"
    
    # Build kubectl command
    kubectl_cmd="kubectl create configmap $configmap_name --namespace=$NAMESPACE"
    
    # Add each key-value pair
    while IFS='=' read -r key value; do
        kubectl_cmd="$kubectl_cmd --from-literal=$key=\"$value\""
    done < "$temp_file"
    
    # Execute the command
    eval "$kubectl_cmd"
    
    # Add labels
    kubectl label configmap "$configmap_name" \
        "$LABEL_SELECTOR" \
        --namespace="$NAMESPACE"
    
    # Clean up
    rm "$temp_file"
    
    if [ $? -eq 0 ]; then
        echo "✓ Successfully created ConfigMap: $configmap_name"
    else
        echo "✗ Failed to create ConfigMap: $configmap_name"
    fi
}

# Process all extracted JSON files
for json_file in "$EXTRACTED_DIR"/*.json; do
    if [ -f "$json_file" ]; then
        # Extract configmap name from filename
        filename=$(basename "$json_file" .json)
        configmap_name="$filename"
        
        create_configmap_from_json "$json_file" "$configmap_name"
    fi
done

echo "ConfigMap creation completed."
```

Make the script executable and run it:
```bash
chmod +x create-configmaps.sh
./create-configmaps.sh
```

#### 4.3 YAML-based ConfigMap Creation
For more control, create ConfigMap YAML files:

```yaml
# app1-configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app1-config
  namespace: default
  labels:
    migrated-from: vault
    app: app1
data:
  database_url: "postgres://user:pass@host:5432/db"
  api_key: "your-api-key"
  debug: "true"
  config.json: |
    {
      "database_url": "postgres://user:pass@host:5432/db",
      "api_key": "your-api-key",
      "debug": "true"
    }
```

Apply the ConfigMap:
```bash
kubectl apply -f app1-configmap.yaml
```

### Step 5: Update Applications

#### 5.1 Modify Application Deployment
Update your application deployment to use ConfigMap instead of Vault:

**Before (Vault integration):**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app1
spec:
  template:
    spec:
      containers:
      - name: app1
        image: app1:latest
        env:
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: vault-secret
              key: database_url
```

**After (ConfigMap integration):**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app1
spec:
  template:
    spec:
      containers:
      - name: app1
        image: app1:latest
        env:
        - name: DATABASE_URL
          valueFrom:
            configMapKeyRef:
              name: app1-config
              key: database_url
        - name: API_KEY
          valueFrom:
            configMapKeyRef:
              name: app1-config
              key: api_key
        volumeMounts:
        - name: config-volume
          mountPath: /etc/config
      volumes:
      - name: config-volume
        configMap:
          name: app1-config
```

#### 5.2 Volume Mount Configuration
For file-based configuration:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app1
spec:
  template:
    spec:
      containers:
      - name: app1
        image: app1:latest
        volumeMounts:
        - name: config-volume
          mountPath: /etc/config
          readOnly: true
      volumes:
      - name: config-volume
        configMap:
          name: app1-config
          items:
          - key: config.json
            path: config.json
```

## Verification and Testing

### Step 1: Verify ConfigMap Creation
```bash
# List all ConfigMaps
kubectl get configmaps --namespace=default

# Describe a specific ConfigMap
kubectl describe configmap app1-config --namespace=default

# Get ConfigMap data
kubectl get configmap app1-config -o yaml --namespace=default
```

### Step 2: Test Application Configuration
```bash
# Create a test pod to verify configuration
kubectl run test-pod --image=busybox --restart=Never --rm -i --tty \
    --env="DATABASE_URL" --env-from=configmap/app1-config \
    -- sh -c 'echo "Database URL: $DATABASE_URL"'

# Test volume mount
kubectl run test-pod --image=busybox --restart=Never --rm -i --tty \
    --volume-mount=name=config,mountPath=/config \
    --volume=name=config,configMap=app1-config \
    -- sh -c 'cat /config/config.json'
```

### Step 3: Application Health Check
```bash
# Check application logs
kubectl logs deployment/app1

# Check application status
kubectl get pods -l app=app1

# Test application endpoints
kubectl port-forward deployment/app1 8080:8080
curl http://localhost:8080/health
```

## Best Practices

### 1. Security Considerations
- **Sensitive Data**: Use Secrets instead of ConfigMaps for sensitive information
- **Access Control**: Implement RBAC for ConfigMap access
- **Encryption**: Enable encryption at rest for etcd

### 2. Configuration Management
- **Versioning**: Use labels to track configuration versions
- **Backup**: Backup ConfigMaps before migration
- **Rollback Plan**: Have a rollback strategy ready

### 3. Monitoring and Logging
- **Track Changes**: Monitor ConfigMap modifications
- **Audit**: Enable audit logging for configuration changes
- **Alerts**: Set up alerts for configuration drift

### 4. Automation
- **CI/CD Integration**: Automate ConfigMap updates in pipelines
- **GitOps**: Use GitOps for configuration management
- **Validation**: Implement configuration validation

## Troubleshooting

### Common Issues and Solutions

#### Issue 1: ConfigMap Size Limit
**Problem**: ConfigMap data exceeds 1MB limit
**Solution**: 
```bash
# Split large configurations into multiple ConfigMaps
kubectl create configmap app1-config-part1 --from-file=part1.json
kubectl create configmap app1-config-part2 --from-file=part2.json
```

#### Issue 2: Character Encoding Issues
**Problem**: Special characters in configuration values
**Solution**:
```bash
# Use base64 encoding for problematic values
echo "special-value" | base64
# Add to ConfigMap as base64-encoded value
```

#### Issue 3: Application Not Reading New Configuration
**Problem**: Application not picking up ConfigMap changes
**Solution**:
```bash
# Restart deployment to pick up new configuration
kubectl rollout restart deployment/app1

# Or use configuration reloader
kubectl annotate deployment app1 configmap.reloader.stakater.com/reload="app1-config"
```

#### Issue 4: Vault Connection Issues
**Problem**: Cannot connect to Vault during extraction
**Solution**:
```bash
# Check Vault status
vault status

# Verify authentication
vault auth -method=token

# Check network connectivity
curl -k $VAULT_ADDR/v1/sys/health
```

## Security Considerations

### 1. Data Classification
- **Identify Sensitive Data**: Classify data before migration
- **Use Appropriate Resources**: ConfigMaps for non-sensitive, Secrets for sensitive
- **Encryption**: Ensure encryption in transit and at rest

### 2. Access Control
```yaml
# RBAC for ConfigMap access
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: configmap-reader
rules:
- apiGroups: [""]
  resources: ["configmaps"]
  verbs: ["get", "list"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: configmap-reader-binding
subjects:
- kind: ServiceAccount
  name: app1-service-account
roleRef:
  kind: Role
  name: configmap-reader
  apiGroup: rbac.authorization.k8s.io
```

### 3. Audit and Monitoring
```bash
# Enable audit logging
kubectl get events --field-selector involvedObject.kind=ConfigMap

# Monitor ConfigMap changes
kubectl get configmaps --watch
```

## Alternative Approaches

### Using External Secrets Operator
If you want to keep Vault as the source of truth:

1. **Install External Secrets Operator**:
```bash
helm repo add external-secrets https://charts.external-secrets.io
helm install external-secrets external-secrets/external-secrets \
    --namespace external-secrets \
    --create-namespace
```

2. **Configure SecretStore**:
```yaml
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: vault-backend
spec:
  provider:
    vault:
      server: "https://vault.example.com:8200"
      path: "secret"
      version: "v2"
      auth:
        tokenSecretRef:
          name: "vault-token"
          key: "token"
```

3. **Create ExternalSecret**:
```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: app1-config
spec:
  refreshInterval: "1h"
  secretStoreRef:
    name: vault-backend
    kind: SecretStore
  target:
    name: app1-config
    creationPolicy: Owner
  data:
  - secretKey: database_url
    remoteRef:
      key: app1/config
      property: database_url
```

## Conclusion

This guide provides a comprehensive approach to migrating from AWS Vault to Kubernetes ConfigMaps. The process involves:

1. **Planning**: Understanding your current Vault setup and target Kubernetes environment
2. **Extraction**: Safely extracting secrets from Vault
3. **Transformation**: Converting secrets to ConfigMap format
4. **Deployment**: Creating ConfigMaps in Kubernetes
5. **Integration**: Updating applications to use ConfigMaps
6. **Verification**: Testing and validating the migration

Remember to:
- Test thoroughly in a non-production environment first
- Have a rollback plan ready
- Consider security implications of the migration
- Monitor applications after migration
- Keep documentation updated

The migration from Vault to ConfigMaps can simplify your infrastructure while maintaining security and reliability when done correctly.