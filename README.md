[![Build Status](https://travis-ci.org/garethahealy/custom-metrics-node-ocp-monitoring.svg?branch=master)](https://travis-ci.org/garethahealy/custom-metrics-node-ocp-monitoring)
[![License](https://img.shields.io/hexpm/l/plug.svg?maxAge=2592000)]()

# custom-metrics-node-ocp-monitoring
## Disclaimer
**Supportability:**
*If you are a RedHat customer; it is __suggested__ you raise a support ticket to clarify the supportability of the below components.
The below are the views of my own, and not representative of RedHat nor of any communities mentioned.*
- Hawkular OpenShift Agent is only supported [upstream](http://www.hawkular.org/community/docs/getting-involved/)
- Prometheus Node Exported is only supported [upstream](https://github.com/prometheus/node_exporter/issues)
- Hawkular Grafana DataSource is only supported [upstream](https://github.com/hawkular/hawkular-grafana-datasource/issues)

## Preface
This blog post is a continuation of [FIS2.0 OCP Monitoring via Hawkular OpenShift Agent](https://github.com/garethahealy/fis2-ocp-monitoring). 
Before continuing; it is expected you have deployed the Hawkular OpenShift Agent and monitoring an example application. 

### Scenario
You have some legacy node monitoring or a custom script that collects metrics based on the output of an 'oc' command.
How can this be ingested into Hawkular Metrics, without the need to write a custom integration for Hawkular Metrics.

### Solution
We can use the [Prometheus Node Exporter](https://github.com/prometheus/node_exporter) image with the textfile exporter enabled, which uses the [Prometheus text format.](https://prometheus.io/docs/instrumenting/exposition_formats/#text-format-details)
The textfile exporter reads any files with a extension of .prom and aggregates them, thus when a call is made to the node exporters /metrics URL, all .prom file metrics are displayed as one.
With this configuration, we can update our original monitoring output to be a textformat and ingested into Hawkular Metrics via the Hawkular OpenShift Agent.

## Example Deployment
### Create example textfile.prom
Firstly, we want to create the folder on the node which the container will mount:

    sudo mkdir /usr/local/node_exporter

And create an example textfile.prom file:

    cat <<EOT >> /usr/local/node_exporter/example.prom
    # HELP node_exporter_textfile_exampletestnumber Example format using a static metric
    # TYPE node_exporter_textfile_exampletestnumber counter
    node_exporter_textfile_exampletestnumber 1
    EOT

The above textfile could contain any metric you want to ingest. The node exporter also supports multiple files which makes flexible.

Next, we need to shown the newly created folder and file so that the pod has read access:

    sudo chown -R 0:0 /usr/local/node_exporter

## Deploy node-exporter
As the node-exporter runs under the 'privileged' SCC (a requirement of host mount), we'll create a new project to segregate it from the rest of our estate:

    oc new-project node-exporters
    oadm policy add-scc-to-user privileged system:serviceaccount:node-exporters:node-exporter
    oc process -f https://raw.githubusercontent.com/garethahealy/custom-metrics-node-ocp-monitoring/master/ocp-template/node-exporter.yaml | oc create -f -

Finally, we want to verify that the Hawkular OpenShift Agent is collecting the metrics:

    AGENT_POD=$(oc get pod -n openshift-infra | grep hawkular-openshift-agent | cut -d' ' -f1)
    oc logs -n openshift-infra -f $AGENT_POD
    
![alt text](https://github.com/garethahealy/custom-metrics-node-ocp-monitoring/raw/master/screenshots/agent_log.png "Agent Log")
    
We are now ready to start visualizing with Grafana!

![alt text](https://github.com/garethahealy/custom-metrics-node-ocp-monitoring/raw/master/screenshots/grafana_example.png "Grafana Example")

## Conclusion
Hopefully, this has shown that by combing the Hawkular OpenShift Agent and the Prometheus Node Exporter, you can easily ingest metrics that are not directly part of your OCP cluster but maybe related.

### Whats next?
If you want to start using this example and extend it for your own metric requirements, you will need to:
- Generate valid textformat files on the OCP node at the host mount path (i.e.: /usr/local/node_exporter)
- Configure the metric name/type in the Hawkular OpenShift Agent configmap, i.e.:
  - https://github.com/garethahealy/custom-metrics-node-ocp-monitoring/blob/master/ocp-template/node-exporter.yaml#L28-L30
