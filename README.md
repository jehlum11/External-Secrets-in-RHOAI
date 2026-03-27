[Tutorial - how to ensure ESO managed credentials with OpenShift AI.md](https://github.com/user-attachments/files/26312657/Tutorial.-.how.to.ensure.ESO.managed.credentials.with.OpenShift.AI.md)
This tutorial outlines how to use the **External Secrets Operator (ESO)** to securely pull credentials from **Vault** into **Red Hat OpenShift AI** without ever handling raw secret values.

## **1\. The Workflow**

The platform team manages Vault and the ClusterSecretStore. Your role as a data scientist is to:

1. Declare what you need via an ExternalSecret resource.  
2. ESO Syncs the data from an external secret store (we use Vault in this example) into a standard Kubernetes Secret.  
3. Consume that Secret in your Notebooks, Pipelines etc.

## **2\. Create the ExternalSecret Custom Resource**

Create a YAML file (e.g., s3-credentials.yaml) to map Vault values to Kubernetes keys. This file contains **no actual secrets** and is safe for Git.

YAML

```
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: s3-credentials
  namespace: demo-ai
spec:
  refreshInterval: 15s # ESO updates the secret if rotated in Vault 
  secretStoreRef:
    name: vault-backend
    kind: ClusterSecretStore
  target:
    name: s3-credentials # Name of the K8s Secret to be created 
  data:
    - secretKey: AWS_ACCESS_KEY_ID # Key name used in your code 
      remoteRef:
        key: demo/s3-credentials
        property: aws-access-key-id
```

**Apply it:** oc apply \-f s3-credentials.yaml.

---

## **3\. Consume Secrets in Workloads in Red Hat OpenShift AI:** 

## **Jupyter Notebooks**

Add the secret to your **Notebook CR** under envFrom to inject all keys as environment variables.

* **Configuration:**  
  YAML

```
spec:
  template:
    spec:
      containers:
        - name: my-workbench
          envFrom:
            - secretRef:
                name: s3-credentials
```

* **Python Usage:**

```
import os
access_key = os.environ["AWS_ACCESS_KEY_ID"
```

Mapping Specific Keys to Custom Environment Variable Names

If you only need certain keys, or want to rename them, use `env.valueFrom.secretKeyRef`
instead of `envFrom`:
(This is useful when the secret's key names don't match what your code expects, or when you only need a subset of the keys.)

```yaml
spec:
  template:
    spec:
      containers:
        - name: my-workbench
          env:
            - name: PGPASSWORD
              valueFrom:
                secretKeyRef:
                  name: db-credentials
                  key: password
            - name: PGHOST
              valueFrom:
                secretKeyRef:
                  name: db-credentials
                  key: host



Python Usage:
import os
pg_host = os.environ["PGHOST"]
pg_pass = os.environ["PGPASSWORD"]


## **Data Science Pipelines (KFP v2)**

Use the kfp.kubernetes extension to map secrets to environment variables in your pipeline steps.

Python

```
from kfp.kubernetes import use_secret_as_env

@dsl.pipeline(name="my-pipeline")
def my_pipeline():
    task = my_step()
    use_secret_as_env(
        task,
        secret_name="s3-credentials",
        secret_key_to_env={"AWS_ACCESS_KEY_ID": "AWS_ACCESS_KEY_ID"} [cite: 186, 200, 201]
    )
```

**Have the secrets show up in the OpenShift AI UI:** 

To make the secret visible in the **OpenShift AI Dashboard**, include the RHOAI labels in the `template` section in the External Secrets Custom Resource.

YAML

```
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: s3-credentials
  namespace: demo-ai
spec:
  refreshInterval: 15s
  secretStoreRef:
    name: vault-backend
    kind: ClusterSecretStore
  target:
    name: s3-credentials
    template:
      metadata:
        labels:
          opendatahub.io/dashboard: "true"
          opendatahub.io/managed: "true"
        annotations:
          opendatahub.io/connection-type: s3
          openshift.io/display-name: "S3 Credentials (from Vault)"
  data:
    - secretKey: AWS_ACCESS_KEY_ID
      remoteRef:
        key: demo/s3-credentials
        property: aws-access-key-id
```

**Apply it:** `oc apply -f s3-credentials.yaml`

The secret now appears in the **Data Connections** section of your project.

* When creating or editing a **Workbench**, simply select "S3 Credentials (from Vault)" from the existing connections list.

---

## **Why Use This?**

* **Security:** No plain-text secrets in code or Git.  
* **Automation:** ESO automatically updates Kubernetes Secrets when credentials rotate in Vault.  
* **Simplicity:** Workloads use standard os.environ—no Vault SDK or special libraries required.

Would you like me to generate a template for the **InferenceService** (Model Serving) configuration as well? 

