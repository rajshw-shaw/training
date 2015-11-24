# Metrics with Hawkular

## metrics project
as `admin`

    oc new-project metrics

delete quota/limits

    oc delete quota/metrics-quota limits/metrics-limits

## metrics sa

    oc create -f ~/training/content/advanced/metrics-deployer-setup.yaml

## metrics permissions/accounts

as `root`:

    oadm policy add-role-to-user edit \
       system:serviceaccount:metrics:metrics-deployer 

    oadm policy add-cluster-role-to-user cluster-reader \
          system:serviceaccount:metrics:heapster

## certs / secrets

    oc secrets new metrics-deployer nothing=/dev/null

## metrics template

    oc create -f ~/training/content/advanced/metrics.yaml

## set up volumes

## create metrics deployer

*temporarily not using persistent storage*

    oc new-app --template=metrics-deployer-template \
    -p HAWKULAR_METRICS_HOSTNAME=metrics.cloudapps.example.com \
    -p IMAGE_PREFIX=rcm-img-docker01.build.eng.bos.redhat.com:5001/openshift3/ \
    -p IMAGE_VERSION=3.1.0 \
    -p USE_PERSISTENT_STORAGE=false

    --> Deploying template metrics-deployer-template for "metrics-deployer-template"
         With parameters:
          IMAGE_PREFIX=rcm-img-docker01.build.eng.bos.redhat.com:5001/openshift3/
          IMAGE_VERSION=3.1.0
          MASTER_URL=https://kubernetes.default.svc:443
          HAWKULAR_METRICS_HOSTNAME=metrics.cloudapps.example.com
          REDEPLOY=false
          USE_PERSISTENT_STORAGE=true
          CASSANDRA_NODES=1
          CASSANDRA_PV_SIZE=1Gi
          METRIC_DURATION=7
    --> Creating resources ...
        Pod "metrics-deployer-1s7nu" created
    --> Success
        Run 'oc status' to view your app.

## update config

/etc/origin/master/master-config.yaml

    assetConfig:
        ...
        metricsPublicURL: "https://metrics.cloudapps.example.com/hawkular/metrics"
