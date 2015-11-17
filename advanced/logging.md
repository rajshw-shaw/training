# Logging With EFK
* overview of the stack
* overview of the architecture
    * ops vs non-ops

## admin user
create:

    htpasswd -b /etc/origin/openshift-passwd admin redhat

give cluster-admin **warning**

    oadm policy add-cluster-role-to-user cluster-admin admin

*do this in infra?*
## create logging project

    oc new-project logging

## delete quota, limits

    oc delete quota/logging-quota limits/logging-limits -n logging

## storage volumes

## edit logging project nodeselector

oc edit namespace logging

add 
openshift.io/node-selector: ""

* want kibana/es on infra
* want fluentd on every node (so it can slurp local container logs)

## create cert for kibana
Create the cert for kibana:

    CA=/etc/origin/master
    oadm ca create-server-cert --signer-cert=$CA/ca.crt \
          --signer-key=$CA/ca.key --signer-serial=$CA/ca.serial.txt \
          --hostnames='kibana.cloudapps.example.com' \
          --cert=/root/kibana.crt --key=/root/kibana.key

## create secrets
    
    oc secrets new logging-deployer \
       kibana.crt=/root/kibana.crt \
       kibana.key=/root/kibana.key \
       kibana-ops.crt=/root/kibana.crt \
       kibana-ops.key=/root/kibana.key

## create service account

    oc create -f ~/training/content/logging-sa.yaml
    
    oc policy add-role-to-user edit \
              system:serviceaccount:logging:logging-deployer

*Note:* change :logging: above to match the project name if you didn't use
`logging`.

## add template

    oc create -f ~/training/content/advanced/logging-deployer.yaml

## deploy deployer
we want to set:

* kibana hostname: will create a route for general user access
* kibana ops hostname: will create a route for the ops users
* public master url: points to the master's public url
* enable ops cluster: true, because we want a separate ops cluster
* es instance and cluster: small settings for our simple environment, see docs

the deployer pod creates all other resources

    oc new-app --template=logging-deployer-template \
    -p IMAGE_PREFIX=rcm-img-docker01.build.eng.bos.redhat.com:5001/openshift3/ \
    -p PUBLIC_MASTER_URL=https://ose3-master.example.com:8443 \
    -p KIBANA_HOSTNAME=kibana.cloudapps.example.com \
    -p ES_INSTANCE_RAM=1024M \
    -p ES_CLUSTER_SIZE=1 \
    -p ENABLE_OPS_CLUSTER=true \
    -p KIBANA_OPS_HOSTNAME=kibana-ops.cloudapps.example.com \
    -p ES_OPS_INSTANCE_RAM=1024M \
    -p ES_OPS_CLUSTER_SIZE=1

see:

    --> Deploying template logging-deployer-template for "logging-deployer-template"
         With parameters:
          IMAGE_PREFIX=rcm-img-docker01.build.eng.bos.redhat.com:5001/openshift3/
          IMAGE_VERSION=latest
          ENABLE_OPS_CLUSTER=true
          KIBANA_HOSTNAME=kibana.cloudapps.example.com
          KIBANA_OPS_HOSTNAME=kibana-ops.cloudapps.example.com
          PUBLIC_MASTER_URL=https://ose3-master.example.com:8443
          MASTER_URL=https://kubernetes.default.svc.cluster.local
          ES_INSTANCE_RAM=1024M
          ES_CLUSTER_SIZE=1
          ES_NODE_QUORUM=
          ES_RECOVER_AFTER_NODES=
          ES_RECOVER_EXPECTED_NODES=
          ES_RECOVER_AFTER_TIME=5m
          ES_OPS_INSTANCE_RAM=1024M
          ES_OPS_CLUSTER_SIZE=1
          ES_OPS_NODE_QUORUM=
          ES_OPS_RECOVER_AFTER_NODES=
          ES_OPS_RECOVER_EXPECTED_NODES=
          ES_OPS_RECOVER_AFTER_TIME=5m
    --> Creating resources ...
        Pod "logging-deployer-2zaq0" created
    --> Success
        Run 'oc status' to view your app.

The deployer created several `DeploymentConfig`s, `Template`s and other
resources:

    oc get template
    
    NAME                           DESCRIPTION                                                                        PARAMETERS        OBJECTS
    logging-deployer-template      Template for deploying everything needed for aggregated logging. Requires clu...   19 (10 blank)     1
    logging-es-template            Template for deploying ElasticSearch with proxy/plugin for storing and retrie...   6 (1 generated)   1
    logging-fluentd-template       Template for logging fluentd deployment. Currently uses terrible kludge to re...   0 (all set)       1
    logging-kibana-template        Template for deploying log viewer Kibana connecting to ElasticSearch to visua...   0 (all set)       1
    logging-support-pre-template   Template for deploying logging services and service accounts.                      0 (all set)       9
    logging-support-template       Template for deploying logging support entities: oauth, service accounts, ser...   0 (all set)       7
    
    oc get dc
    
    NAME                  TRIGGERS                    LATEST
    logging-es-gv6op6fm   ConfigChange, ImageChange   0
    logging-fluentd       ConfigChange, ImageChange   0
    logging-kibana        ConfigChange, ImageChange   0
    
    oc get sa

    NAME                               SECRETS   AGE
    aggregated-logging-elasticsearch   3         5m
    aggregated-logging-fluentd         3         5m
    aggregated-logging-kibana          5         5m
    builder                            2         1h
    default                            2         1h
    deployer                           2         1h
    logging-deployer                   3         59m


## create support template

    oc new-app --template=logging-support-template

you see

    --> Deploying template logging-support-template for "logging-support-template"
    --> Creating resources ...
        OAuthClient "kibana-proxy" created
        Route "kibana" created
        Route "kibana-ops" created
        ImageStream "logging-auth-proxy" created
        ImageStream "logging-elasticsearch" created
        ImageStream "logging-fluentd" created
        ImageStream "logging-kibana" created
    --> Success
        Run 'oc status' to view your app.

## **temporary** edit image streams
rcm-img-docker01.build.eng.bos.redhat.com:5001
openshift.io/image.insecureRepository: "true"

## edit dcs for correct nodeselector

need to edit es and kibana node selectors - do not edit fluentd
look for "component"

    spec:
      nodeSelector:
        region: infra
      containers:

wait for new deployment

## edit privileged scc
fluentd logger need to host mount docker filesystems to access container logs

    oc edit scc/privileged

add following:

    - system:serviceaccount:logging:aggregated-logging-fluentd

*Note:* Change `:logging:` to whatever project you used

we also need to be able to read pod labels so that elasticsearch can find
associations

    oadm policy add-cluster-role-to-user cluster-reader \
    system:serviceaccount:logging:aggregated-logging-fluentd

## elastic notes
https://github.com/openshift/origin-aggregated-logging/tree/master/deployment#elasticsearch

## configure elastic storage volumes
https://github.com/openshift/origin-aggregated-logging/tree/master/deployment#storage

## scale fluentd rc

in our environment, we'd need a fluentd on every node, which means a quantity of
3.

    oc scale dc/logging-fluentd --replicas=3

This will cause a deployment of the 3 pods:

    oc get rc | grep fluent

will see

    logging-fluentd-1           fluentd-elasticsearch
    rcm-img-docker01.build.eng.bos.redhat.com:5001/openshift3/logging-fluentd:latest
    component=fluentd,deployment=logging-fluentd-1,deploymentconfig=logging-fluentd,provider=openshift
    3          12m

## login to kibana ops
open browser to:

    https://kibana-ops.cloudapps.example.com

use admin/redhat
