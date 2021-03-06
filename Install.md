# Introduction

The guide help you to provide step by step instruction on setting up certificate exporter.

* Ops Manager UAA User Setup
* Deployment to Cloud Foundry `cf push`
* Deployment to Kubernetes
* Prometheus Setup
* Grafana Dashboard
* Multiple Foundations Support

## Ops Manager UAA User Setup

In order to connect and access the Ops Manager API for extracting all the certificates a minimum of readonly uaa username
and password must be provided, i.e the username must have minimum `opsman.restricted_view` privilege.

Here is a quick step to create a read only API user.

For more information on creating users on Ops Manager checkout the [documentation](https://docs.pivotal.io/pivotalcf/2-6/customizing/opsman-users.html) and check
this [documentation](https://docs.pivotal.io/pivotalcf/2-6/opsguide/config-rbac.html) for different role based access control available.

```
# Connect to ops manager UAA using the admin account
uaac target https://<OPSMAN URL>/uaa --skip-ssl-validation
uaac token owner get opsman admin -s "" -p <PASSWORD>

# Create a new read only user and assign a read permission
uaac user add prometheus-cert-exporter -p prometheus-cert-exporter-password --emails prometheus-cert-exporter@prometheus.com
uaac member add opsman.restricted_view prometheus-cert-exporter

# If you are using SAML to authenticate user on opsmanager, then you need to setup a UAAC read only client instead of a read only user mentioned above
uaac client add prometheus-cert-exporter -s prometheus-cert-exporter-secret --scope opsman.restricted_view --authorized_grant_types client_credentials --authorities opsman.restricted_view
```

## Deployment to Cloud Foundry

Push the code to cloud foundry

```
# Clone the repository
git clone https://github.com/pivotal-gss/tanzu-certificate-exporter.git
cd tanzu-certificate-exporter

# Open and edit the manifest and provide the values of the env variable.
vi manifest

# if you are want use https connection b/w API request make sure you set appropriate value in the manifest for
SKIP_SSL_VALIDATION: false
CACERTFILE: "certificate-file-name"

NOTE: The "certificate-file-name" should be copied and available at the root directory of the app when performing cf push

# push the app to cloud foundry
cf push

# You can verify if metric is being emitted by adding the path "/metrics" to the URL
# Note down the route for the app for later use
```

## Deployment to Kubernetes

+ Use the [Dockerfile](./Dockerfile)
+ Create the Kubernetes secret for Ops Manager API username and password
+ Update the [deployment yaml](deployments/kubernetes/tanzu-certificate-exporter.yaml) environment variables and service type (ClusterIP vs LoadBalancer)
+ Add into Prometheus target

```
kubectl create secret generic ops-manager-secret --from-literal=username=admin --from-literal=password='<password>'
kubectl apply -f tanzu-certificate-exporter.yaml
```

## Prometheus Setup

**NOTE:** These steps will restart the prometheus agent and might result in downtime.

### Option 1: Using [prometheus-boshrelease](https://github.com/bosh-prometheus/prometheus-boshrelease) release

If you are using prometheus which is part of the [prometheus-boshrelease](https://github.com/bosh-prometheus/prometheus-boshrelease) then in order to register the route:

```
# Open the manifests/prometheus.yml file
vi manifests/prometheus.yml

# Add in additional jobs under scrape config i.e under the section jobs > properties > prometheus > scrape_configs
Say my cert exporter route is "vmware-tanzu-cert-exporter.domain1.com, vmware-tanzu-cert-exporter.domain2.com, etc" obtained frrom step 2 above,
my basic scrape config would be something like this

scrape_configs:
- file_sd_configs:
  - files:
    - /var/vcap/store/bosh_exporter/bosh_target_groups.json
  job_name: prometheus
  relabel_configs:
  - action: keep
    regex: prometheus\d?
    source_labels:
    - __meta_bosh_job_process_name
  - regex: (.*)
    replacement: ${1}:9090
    source_labels:
    - __address__
    target_label: __address__
- job_name: vmware-tanzu-cert-exporter
  static_configs:
  - targets:
    - vmware-tanzu-cert-exporter.domain1.com
    - vmware-tanzu-cert-exporter.domain2.com
......

Save the file and update the deployment

bosh -d prometheus deploy manifests/prometheus.yml --vars-store tmp/deployment-vars.yml

if you are using additional operator please don't forget to include them like eg.s below,
check the prometheus-boshrelease for more information on it

bosh -d prometheus deploy manifests/prometheus.yml \
    --vars-store tmp/deployment-vars.yml \
    -o manifests/operators/monitor-bosh.yml \
    -v bosh_url= \
    -v bosh_username= \
    -v bosh_password= \
    --var-file bosh_ca_cert= \
    -v metrics_environment= \
    -o manifests/operators/monitor-cf.yml \
    -v metron_deployment_name= \
    -v system_domain= \
    -v uaa_clients_cf_exporter_secret= \
    -v uaa_clients_firehose_exporter_secret= \
    -v traffic_controller_external_port= \
    -v skip_ssl_verify=

Ensure before confirming that the deployment is only updating the changes implemented above and nothing else,
  eg.s below show the deployment is only going to publish our changes.

  instance_groups:
  - name: prometheus2
    jobs:
    - name: prometheus2
      properties:
        prometheus:
          scrape_configs:
+         - job_name: "<redacted>"
+           static_configs:
+           - targets:
+             - "<redacted>"

in case you find many variables being modified cancel the deployment during confirmation and ensure you have included
all the bosh operator when you deployed this release the last time or continue if you are comfortable

# If deployment had successfully completed, the connect to prometheus GUI and see if you can find in metrics from
"vmware_tanzu_cert_exporter_cert_expires_in_seconds"
```

### Option 2: Bring Your Own Prometheus

If you are managing your own prometheus in-house, then follow the below steps

```
# Open the prometheus.yml
vi prometheus.yml

# Add in additional jobs under scrape_configs

....
scrape_configs:
  - job_name: 'prometheus'
    static_configs:
    - targets: ['127.0.0.1:9090']

  - job_name: 'vmware_tanzu_cert_exporter'
    static_configs:
    - targets:
      - vmware-tanzu-cert-exporter.domain1.com
      - vmware-tanzu-cert-exporter.domain2.com

# Restart the prometheus to reload the configuration

# If deployment had successfully completed, the connect to prometheus GUI and see if you can find in metrics from
"vmware_tanzu_cert_exporter_cert_expires_in_seconds"
```

## Grafana Dashboard

Once the prometheus scraping is setup, navigate to Grafana UI to setup the dashboard.

+ Click on the + sign on the left nav bar
+ Select import
+ Open and copy the content from [Grafana.Json](https://github.com/pivotal-gss/tanzu-certificate-exporter/blob/master/resources/Grafana.json)
+ Paste the Json onto the Grafana Import Page and click on Load
+ Correct any error if found, once satisfied click on import.

## Multiple Foundations Support

Repeat the steps listed in this document for additional foundation you would like to monitor. Use environment variable `ENVIRONMENT` to differentiate the foundation when starting the exporter
