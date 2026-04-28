# Keytool Logging in OpenShift Pods

## Overview
This document explains how to view keytool command logs in OpenShift pods for the KIE (Knowledge Is Everything) modules.

## Changes Made

### 1. jboss-kie-workbench Module
The `configure_kie_keystore()` function in `jboss-kie-workbench/added/launch/jboss-kie-workbench.sh` now includes:
- Informational logging for each step of keystore configuration
- Success/failure logging for each keytool command execution
- Optional verbose keytool output when debug mode is enabled

### 2. jboss-kie-smartrouter Module
The TLS keystore configuration in `jboss-kie-smartrouter/added/launch/jboss-kie-smartrouter.sh` now includes:
- Logging for test keystore generation
- Logging for keystore verification
- Optional verbose keytool output when debug mode is enabled

## How to View Keytool Logs

### Default Mode (Basic Logging)
By default, you'll see informational logs about keystore operations:

```bash
# View pod logs
oc logs <pod-name>

# Follow logs in real-time
oc logs -f <pod-name>

# View logs from a specific container in a pod
oc logs <pod-name> -c <container-name>
```

Example output:
```
INFO Removing existing keystore: /opt/eap/standalone/configuration/kie-keystore.jceks
INFO Configuring KIE keystore at: /opt/eap/standalone/configuration/kie-keystore.jceks
INFO Creating keystore entry for alias: kieServerAlias
INFO Successfully created keystore entry for alias: kieServerAlias
INFO Creating keystore entry for alias: kieCtrlAlias
INFO Successfully created keystore entry for alias: kieCtrlAlias
INFO Keystore configuration completed
```

### Debug Mode (Verbose Keytool Output)
To see detailed keytool command output, set the `KIE_KEYSTORE_DEBUG` environment variable:

#### Option 1: Using DeploymentConfig
```yaml
apiVersion: apps.openshift.io/v1
kind: DeploymentConfig
metadata:
  name: myapp-kieserver
spec:
  template:
    spec:
      containers:
      - name: myapp-kieserver
        env:
        - name: KIE_KEYSTORE_DEBUG
          value: "true"
```

#### Option 2: Using oc command
```bash
# Set environment variable
oc set env dc/<deployment-config-name> KIE_KEYSTORE_DEBUG=true

# Or for Deployment
oc set env deployment/<deployment-name> KIE_KEYSTORE_DEBUG=true
```

#### Option 3: Using ConfigMap
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: kie-config
data:
  KIE_KEYSTORE_DEBUG: "true"
---
apiVersion: apps.openshift.io/v1
kind: DeploymentConfig
metadata:
  name: myapp-kieserver
spec:
  template:
    spec:
      containers:
      - name: myapp-kieserver
        envFrom:
        - configMapRef:
            name: kie-config
```

With debug mode enabled, you'll see detailed keytool output:
```
INFO Running keytool command for kieServerAlias (verbose mode enabled)
INFO keytool: Importing password
INFO keytool: Entry for alias kieServerAlias successfully imported.
INFO keytool: Import command completed:  1 entries successfully imported, 0 entries failed or cancelled
INFO Successfully created keystore entry for alias: kieServerAlias
```

## Troubleshooting

### View Recent Logs
```bash
# Last 100 lines
oc logs <pod-name> --tail=100

# Logs from last hour
oc logs <pod-name> --since=1h

# Logs with timestamps
oc logs <pod-name> --timestamps
```

### Search for Keytool-Related Logs
```bash
# Search for keytool logs
oc logs <pod-name> | grep -i keytool

# Search for keystore logs
oc logs <pod-name> | grep -i keystore

# Search for errors
oc logs <pod-name> | grep -i "error\|failed\|warning"
```

### Common Issues

#### 1. Keystore Creation Failure
If you see:
```
WARNING Failed to create keystore entry for alias: kieServerAlias (exit code: 1)
```

Enable debug mode to see the detailed error:
```bash
oc set env dc/<deployment-config-name> KIE_KEYSTORE_DEBUG=true
oc rollout latest dc/<deployment-config-name>
```

#### 2. TLS Keystore Verification Failure
If you see:
```
WARNING Unable to read TLS keystore (exit code: 1), skipping https setup
```

Check:
- The keystore file exists and is mounted correctly
- The keystore password is correct
- The keystore alias exists
- Enable debug mode for detailed output

### Export Logs for Analysis
```bash
# Export logs to file
oc logs <pod-name> > pod-logs.txt

# Export logs from all pods in a deployment
for pod in $(oc get pods -l app=myapp -o name); do
  oc logs $pod > ${pod##*/}-logs.txt
done
```

## Environment Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `KIE_KEYSTORE_DEBUG` | `false` | Enable verbose keytool output in logs |

## Log Levels

The scripts use standard logging functions:
- `log_info`: Informational messages (always shown)
- `log_warning`: Warning messages (always shown)
- `log_error`: Error messages (always shown)

When `KIE_KEYSTORE_DEBUG=true`, keytool verbose output is prefixed with `keytool:` for easy filtering.

## Best Practices

1. **Enable debug mode only when troubleshooting** - Verbose output can be extensive
2. **Use log filtering** - Grep for specific keywords to find relevant information
3. **Check logs immediately after pod startup** - Keystore configuration happens during initialization
4. **Save logs before pod restarts** - Logs are lost when pods are deleted

## Related Files

- `jboss-kie-workbench/added/launch/jboss-kie-workbench.sh` - Workbench keystore configuration
- `jboss-kie-smartrouter/added/launch/jboss-kie-smartrouter.sh` - Smart Router TLS keystore configuration
- `test-keystore-ubi9.sh` - Test script for keystore functionality