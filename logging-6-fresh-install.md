# Installing OpenShift Logging 6.1+

A lot has changed between OpenShift Logging 5.x and 6.1.  The logging stack has been completely changed (Elastic/FluentD/Kibana for Loki/Vector), the "Cluster Observability Operator" has been introduced, and RBAC has changed.  The purpose of this doc is to help get the "Quick Start" of Logging 6.1 up and running.

## Prerequisites

This demo uses ODF for block/file/object (RWO/RWX/S3) storage classes.  If you are using something else, you'll need to adapt accordingly.  The main purpose of this doc is to highlight important steps, so the storage details should not be materially important.

## Docs

A good place to start is the [Logging 6.1 docs](https://docs.redhat.com/en/documentation/openshift_container_platform/4.17/html/logging/logging-6-1).  Although they are currently thin on certain details (for example, how to format your storage secret), they do have everything you need to get going.

## Getting Started

### Install the Operators

The first thing to do is [install three operators](https://docs.redhat.com/en/documentation/openshift_container_platform/4.17/html/logging/logging-6-1#quick-start-viaq_logging-6x-6.1):

* Red Hat Cluster Observability Operator
* Red Hat Loki Operator
* Red Hat Cluster Logging Operator

### Create Object Storage

One major difference between Loki and Elasticsearch is the fact that Loki requires Object Storage.  Since the cluster I'm using has OpenShift Data Foundation installed, I'll create an Object Bucket Claim to provide S3-compatible storage.

```
apiVersion: objectbucket.io/v1alpha1
kind: ObjectBucketClaim
metadata:
  name: loki-bucket
  namespace: openshift-logging
spec:
  additionalConfig:
    bucketclass: noobaa-default-bucket-class
  generateBucketName: oadp-object-storage
  storageClassName: openshift-storage.noobaa.io
```

Next, we'll need to extract the auth details from the secret that was created automatically for this bucket:

```
oc project openshift-logging

ACCESS_KEY_ID=$(oc get secret loki-bucket -o go-template --template="{{.data.AWS_ACCESS_KEY_ID|base64decode}}")
SECRET_ACCESS_KEY=$(oc get secret loki-bucket -o go-template --template="{{.data.AWS_SECRET_ACCESS_KEY|base64decode}}")
BUCKET_NAME=$(oc get cm loki-bucket -o go-template --template="{{.data.BUCKET_NAME}}")
```

Using these values, create a secret.  The Loki operator will use this secret to connect your Loki instance to your new object bucket.

```
oc create -n openshift-logging secret generic logging-loki-odf \
  --from-literal=access_key_id="$ACCESS_KEY_ID" \
  --from-literal=access_key_secret="$SECRET_ACCESS_KEY" \
  --from-literal=bucketnames="$BUCKET_NAME" \
  --from-literal=endpoint="https://s3-openshift-storage.apps.<cluster url>"
```

You're now ready to continue with the "Quick Start" to get Loki up and running!

### Install Loki

From here on out, we'll [follow the quick start docs](https://docs.redhat.com/en/documentation/openshift_container_platform/4.17/html/logging/logging-6-1#quick-start-viaq_logging-6x-6.1) to install Loki.

First, you'll need to create a Loki CRD.  The one I used looks like this:

```
apiVersion: loki.grafana.com/v1
kind: LokiStack
metadata:
  name: logging-loki
  namespace: openshift-logging
spec:
  managementState: Managed
  size: 1x.pico
  storage:
    schemas:
    - effectiveDate: '2024-10-01'
      version: v13
    secret:
      name: logging-loki-odf
      type: s3
  storageClassName: ocs-external-storagecluster-ceph-rbd
  tenants:
    mode: openshift-logging
```

Key things to note about the CRD above:

1. The **size** is set to `1x.pico`.  Please refer to the [Loki sizing guide](https://docs.redhat.com/en/documentation/openshift_container_platform/4.17/html/logging/logging-6-1#log6x-loki-sizing_log6x-loki-6.1) to determine the size you will require.
2. **Secret name / type**:  This is the name of the secret we created above.  The `type` corresponds to the type of object storage that is used.  Since we're using ODF, Noobaa presents object storage using the standard "S3" interface, so we choose "s3".
3. **StorageClassName**:  Loki stores logs in object storage, but it still requires RWO storage for other purposes.  Please use the storage class that's available in your cluster.

Once this CRD is applied, you will see the Loki stack spin up in the `openshift-logging` namespace.  So far, so good!  But there are a few more steps before you're done.

### ServiceAccount and RBAC

Another big difference between the old and new logging stacks is RBAC.  With Logging 6.1+, the log collector doesn't have access by default to collect application/infrastructure/audit logs.  This is an intentional change to better align with "principle of least privilege" security standards.  It's not a difficult step, but if you miss this step, you will be wondering why your logs aren't showing up in the logging UI :)

If you continue on in the doc, you will see the steps required to provide the collector with the requires `ServiceAccount` and RBAC.  The steps look like this:

```
oc project openshift-logging

oc create sa collector -n openshift-logging

oc adm policy add-cluster-role-to-user logging-collector-logs-writer -z collector

oc adm policy add-cluster-role-to-user collect-application-logs -z collector

oc adm policy add-cluster-role-to-user collect-audit-logs -z collector

oc adm policy add-cluster-role-to-user collect-infrastructure-logs -z collector
```

With that `ServiceAccount` and role binings in place, we're ready to continue.

### UI Plugin

Create the following UI Plugin to enable the **Log** view under **Observe** in the UI:

```
apiVersion: observability.openshift.io/v1alpha1
kind: UIPlugin
metadata:
  name: logging
spec:
  type: Logging
  logging:
    lokiStack:
      name: logging-loki
```

### Log Forwarding

The final step is to configure the `ClusterLogForwarder` to forward logs to Loki.

This is also detailed if you are following along in the docs, but the basic configuration looks like this:


```
apiVersion: observability.openshift.io/v1
kind: ClusterLogForwarder
metadata:
  name: collector
  namespace: openshift-logging
spec:
  serviceAccount:
    name: collector
  outputs:
  - name: default-lokistack
    type: lokiStack
    lokiStack:
      authentication:
        token:
          from: serviceAccount
      target:
        name: logging-loki
        namespace: openshift-logging
    tls:
      ca:
        key: service-ca.crt
        configMapName: openshift-service-ca.crt
  pipelines:
  - name: default-logstore
    inputRefs:
    - application
    - infrastructure
    outputRefs:
    - default-lokistack
```

This tells the log forwarder to send **application** and **infrastructure** logs to Loki.  This config leaves **audit** logs alone, as many organizations prefer to send audit logs to an external secure storage location.

## DONE!

With the previos config in place, you should have a basic Loki Stack setup in place and collecting logs.  Provided your apps are logging to `stdin/stderr`, you should be able to visualize and search logs under the "Log" tab in the "Observe" are of the OpenShift UI!

## Next Steps

Familiarize yourself with Loki Log Queries:
[Loki Log Query Syntax](https://grafana.com/docs/loki/latest/query/log_queries/)