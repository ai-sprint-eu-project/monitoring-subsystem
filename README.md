AI-SPRINT Monitoring Subsystem (AMS)
====================================

![](https://github.com/ai-sprint-eu-project/monitoring-subsystem/blob/master/ams-logo.png?raw=true)


About this manual
-----------------

### Purpose
To provide a single place which combines general instructions
and technical information about the Monitoring Subsystem.

### Intended audience
DevOps, integrators, administrators, technical users.

### Format
*Technical* information about high-level architecture, use-cases, deployment
and usage instructions.


Links
-----

AMS resources:
*   https://github.com/ai-sprint-eu-project/ams-core-helm-chart
*   https://github.com/ai-sprint-eu-project/ams-client
*   https://github.com/ai-sprint-eu-project/ams-logging-helm-chart
*   https://github.com/ai-sprint-eu-project/ams-log-forwarder-helm-chart
*   https://github.com/ai-sprint-eu-project/ams-manager
*   https://github.com/ai-sprint-eu-project/ams-influxdb-synchroniser
*   https://github.com/ai-sprint-eu-project/ams-api


Manual structure
----------------

This manual is split into three parts, one for every high-level logical
sub-component of the Monitoring Subsystem:
1.  AMS core,
2.  AMS logging framework,
3.  AMS InfluxDB synchroniser.

AMS core is the basic component of the Monitoring Subsystem and consists of
the core building blocks defining the AI-SPRINT metric framework
and the RESTful API intended (but not limited to) for SPACE4AI-R.


Documents of interest
---------------------

*   [D3.4][d34] - "Final release and evaluation of the monitoring system"
*   [D2.2][d22] - "Initial AI-SPRINT design and runtime tools integration"
*   [D2.3][d23] - "Second Release and Evaluation of the AI-SPRINT Design Tools"

[d34]:  https://drive.google.com/file/d/1vTk4T06-NaoiVduUspsubESy8HK9zxIp/view
[d22]:  https://drive.google.com/drive/folders/1OpyeWzW1-GQ6Aus1TPKbT_RLoTW0BjRt
[d23]:  https://drive.google.com/drive/folders/1UZ1SBGgt7r8_PMt1volFUwKNs0Q2lQ6X


Additional information
======================

Exposure, forwarding, security
------------------------------

The topic of tunnelling, securing exposed services, etc. is out of scope
of this manual. An easy exemplary command to achieve such configuration
on an already secured cluster is:
```sh
kubectl expose service $SERVICE_NAME --port=$PORT \
        --name=$EXPOSURE_NAME \
        --external-ip=$EXTERNAL_INTERFACE_ADDRESS
```
where:
*   `EXTERNAL_INTERFACE_ADDRESS` is the address of the external
    interface for the cloud cluster,
*   `EXPOSURE_NAME` is a unique name for the exposed service,
*   `SERVICE_NAME` and `PORT` are parameters dependant on the service
    that is to be exposed.

K8s documentation articles worth looking at regarding the topic
of service exposure:
*   [proxies][proxy],
*   [exposure][expose],
*   [ingress][ingress].

[proxy]:    https://kubernetes.io/docs/concepts/cluster-administration/proxies/
[expose]:   https://kubernetes.io/docs/tutorials/stateless-application/expose-external-ip-address/
[ingress]:  https://kubernetes.io/docs/concepts/services-networking/ingress/



AMS core manual
===============

AMS core is the most basic collection of Monitoring Subsystem components,
providing basic AI-SPRINT features. It consists of:
*   InfluxDB - the metric "database",
*   Grafana - visualisation tool,
*   infrastructure agent (Telegraf) - infrastructure metric collection tool,
*   application agent (Telegraf) - a proxy and cache between applications
    and the database,
*   AMS REST API - allows SPACE4AI-R to query various
    system state parameters.

Every K8s cluster on every level of cloud continuum should have
an AMS deployment installed, thus should have a *local* metric database,
and local querying API. By design all communication on the given level
is performed locally and is contained within the same cluster. The only
component performing external operations is
the AMS InfluxDB synchroniser.


Installation
============

```sh
helm install $DEPLOYMENT_NAME ams-core.tar.gz -n $K8S_NAMESPACE
```


Access
------

The user will expect access to Grafana. Login is: "admin", acquire
password with:
```sh
kubectl get secret ai-sprint-monit-grafana
```
(filter out encoded password, then base64 decode).

The topic of service exposure has been mentioned above.
The example expose command in this case is:
```sh
kubectl expose service ai-sprint-monit-grafana --port=3000 \
        --name=ai-sprint-monit-grafana-external \
        --external-ip=$EXTERNAL_INTERFACE_ADDRESS
```

A typical user is expected to not access InfluxDB or its UI.


Post installation
-----------------

After successful deployment:
1.  call setup monitoring MM function, providing core AMS configuration file,
2.  call setup app MM function, providing QoS configuration file,
3.  if applicable, call setup app custom MM function,
    providing custom metrics configuration file.


More details
============

Manager functions
-----------------

Available functions:
1.  `setup_monitoring` - AMS (re)configuration,
2.  `setup_app` - application (re)configuration,
3.  `setup_app_custom` - custom metrics (re)configuration,
4.  `remove_app` - application cleanup.
Set up functions must be called after installation, but can be called
multiple times to re-configure AMS.

MM functions can be called via Kubernetes REST [API][kubernetes-api],
clients for which are available for every major programming language.
Equivalents using kubectl are:

### setup monitoring
```sh
kubectl cp monitoring.yaml $(kubectl get pods -l=app=ai-sprint-monit-manager -o jsonpath='{.items[0].metadata.name}'):templates/monitoring_setup.yaml
kubectl exec deployment/ai-sprint-monit-manager -- ./setup_monitoring.sh
```

### setup app
```sh
kubectl cp qos_consraints.yaml $(kubectl get pods -l=app=ai-sprint-monit-manager -o jsonpath='{.items[0].metadata.name}'):templates/qos_constraints.yaml
kubectl exec deployment/ai-sprint-monit-manager -- ./setup_app.sh
```

### setup app custom
```sh
kubectl cp custom.yaml $(kubectl get pods -l=app=ai-sprint-monit-manager -o jsonpath='{.items[0].metadata.name}'):templates/custom_setup.yaml
kubectl exec deployment/ai-sprint-monit-manager -- ./setup_app_custom.sh
```

### remove app
```sh
kubectl exec deployment/ai-sprint-monit-manager -- ./remove_app.sh
```

[kubernetes-api]:   https://kubernetes.io/docs/reference/


Alert definition
----------------

Syntax:
```YAML
name:
  condition:
    metric:
      name: metric name
      field: field name
    threshold:
      type: <, >, or range
      value: threshold value
  every: check period
  notification_endpoint:
    url: notification endpoint url
```

Example:
```YAML
my_alert:
  condition:
    metric:
      name: my_custom_field
      field: my_custom_value
    threshold:
      type: ">"
      value: 12.345
  every: 123s
  notification_endpoint:
    url: http://1.2.3.4:5678/
```


AMS configuration
-----------------

Syntax:
```YAML
monitoring:
  metrics:
    metrics + config dictionary
  time_period: collection period
  parameters:
    AMS parameters dictionary
  alerts:
    alert definitions dictionary
```

Example:
```YAML
monitoring:
  metrics:
    cpu: null
    mem: null
  time_period: 10s
  parameters:
    performance_metrics_time_window_width: 90s
    default_notification_endpoint: http://my.endpoint/
  alerts: {}
```

Example with custom plugin configuration:
```YAML
monitoring:
  metrics:
    disk:
      mount_points:
        - /
  time_period: 10s
  parameters:
    performance_metrics_time_window_width: 90s
    default_notification_endpoint: http://my.endpoint/
  alerts: {}
```

Example with an alert:
```YAML
monitoring:
  metrics:
    cpu: null
  time_period: 10s
  parameters:
    performance_metrics_time_window_width: 90s
    default_notification_endpoint: http://my.endpoint/
  alerts:
    cpu_usage_alert:
      condition:
        metric:
          name: cpu
          field: usage_user
        threshold:
          type: ">"
          value: 75
      every: 30s
      notification_endpoint:
        url: http://my.endpoint/
```

This will result in reconfiguration of Telegrafs, creation of alert tasks
and creation/reconfiguration of Grafana system dashboard. 

More on Telegraf plugins can be found in its documentation
([here][telegraf-inputs-gh]).

[telegraf-inputs-gh]:   https://github.com/influxdata/telegraf/tree/master/plugins/inputs/


App configuration
-----------------

This configuration file comes from AI-SPRINT design tools, but if necessary
can be adjusted manually.

Example:
```YAML
system:
  name: app-name
  local_constraints:
    local_constraint_1:
      component_name: component-1
      threshold: 12.3
    local_constraint_2:
      component_name: component-2
      threshold: 23.4
  global_constraints:
    global_constraint_1:
      path_components:
        - component-1
        - component-2
      threshold: 34.5
  throughput_component: component-1
```

Local/global constraint names are just unique identifiers and don't require
explicit numbering.

Throughput component is the name of the component to interpret as the entry
point to the processing pipeline.

This will result in creation/reconfiguration of application Telegraf agent,
application config map, constraint tasks, constraint alert tasks,
throughput task and Grafana AI-SPRINT dashboard.


Custom metrics
--------------

Syntax:
```YAML
monitoring:
  name: application name (same as in QoS constraints)
  metrics:
    metric name:
      fields:
        - field 1 name
  alerts: alert definition dictionary
```

Example:
```YAML
monitoring:
  name: app-name
  metrics:
    my_custom_metric:
      fields:
        - custom_field_1
        - custom_field_2
  alerts:
    custom_alert:
      condition:
        metric:
          name: my_custom_metric
          field: custom_field_2
        threshold:
          type: ">"
          value: 2.34
      every: 345s
      notification_endpoint:
        url: http://5.6.7.8:9012/
```


App integration
---------------

Application must have its environment populated with AMS config map.

AMS will create a config map named `[app name]-config`, where app name
is the name used in app setup script (taken by default from QoS YAML file).

Kubernetes can dereference config maps via manifest file, but any method that
populates the environment (even manual) will work.


Python integration
------------------

Reporting QoS information (execution times, throughput) is performed
transparently via AI-SPRINT `@exec_time` and `@throughput` annotations.
No additional actions are required on application level.

To report a custom metric:
```py
from aisprint.monitoring import report_metric
...
report_metric('my_custom_metric', {'field_1': 1.23,
                                   'field_2': 4,
                                   'field_3': True,
                                   'field_4': 'text is also possible'},
              {'custom_tag': 'xxx'})
```

Additional tags are optional, but can be helpful in advanced Flux queries
or integrating multiple data sources in the same InfluxDB.

Text and boolean values are possible, however must be filtered with custom
queries and render terribly in Grafana.


REST API
========

API endpoints
-------------

### session execution time
```
GET /monitoring/execution_time/session/UUID
```
Response:
```
{
    {UUID}: [
        {component 1}: {
            "execution_time": ...,
            "timestamp": ...
        },
        ...
    ],
}
```

### throughput
```
GET /monitoring/throughput/
```
Response:
```
{
    "throughput": ...,
    "timestamp": ...
}
```

### component mean execution time
```
GET /monitoring/execution_time/components/component
```
Response:
```
{
    {component}: {
        "mean": ...,
        "throughput": ...,
        "count": ...
    }
}
```

### component path mean execution time
```
GET /monitoring/execution_time/components/component1,component2,...
```
Response:
```
{
    {component1,component2,...}: {
        "mean": ...,
        "throughput": ...,
        "count": ...
    },
    {component1}: {
        "mean": ...,
        "throughput": ...,
        "count": ...
    },
    ...
}
```



AMS InfluxDB synchroniser manual
================================

The AMS InfluxDB synchroniser is a tool that allows connecting
AMS deployments on different clusters, thus enabling AI-SPRINT
to run across cloud computing continuum.

Underneath this tool moves stored metrics from one InfluxDB to another one,
without strong network connectivity, allowing edge devices to operate
for long periods of time disconnected from the Internet.


Architecture
------------

This tool is composed only of a single pod running a custom Python tool
that synchronises metrics from one bucket to another.

The tool runs in the lower-level cluster - in push model.

In order to operate it is assumed that the higher-level InfluxDB (usually
in the cloud) API endpoint is at least periodically available under
a specified address.

This tool is not tied to AI-SPRINT constraints or metric names and does
not require reconfiguration.


Installation
------------

The installation process is being handled by the Infrastructure Manager.

Internally, Monitoring Manager `gen_sync_config` and `setup_sync` functions
handle the installation.  It is expected that InfluxDB REST API endpoint URL
and token are provided to the lower-level cluster.

Token can be generated throuch InfluxDB UI or via InfluxDB REST API, e.g.:
```sh
influx auth create -o ai-sprint -d 'InfluxDB synchroniser token' \
        --read-bucket $BUCKET_ID --write-bucket $BUCKET_ID
```
where `BUCKET_ID` is the identifier of the target bucket.

Usually source and target buckets bear the same name as the application
(`name` parameter in the QoS YAML file).



AMS logging framework manual
============================

Contents of this document:
1.  purpose of this framework,
2.  how this framework is composed and what the building blocks are,
3.  when to use which piece,
4.  how to deploy, manage, configure,
5.  how to integrate with AI-SPRINT applications,
6.  how to use.


Introduction
------------

The purpose of the AMS logging framework is to simplify application debugging
by transparently collecting and storing logged messages into a central
ElasticSearch instance.


Architecture
------------

The logging framework is split into two sub-components:
1.  central component,
2.  log forwarder.

The entire logging framework stack consists of:
1.  ElasticSearch - the "database" for collected messages,
2.  Kibana - the UI for visualising data,
3.  fluentd - collector, aggregator, buffer, forwarder - a proxy between
    application and database (provides Logstash protocol).


Deployment
----------

### Central component

To deploy the central component:
```sh
helm install $DEPLOYMENT_NAME ams-logging-framework.tar.gz -n $K8S_NAMESPACE
```
It is assumed that the Kubernetes namespace already exists, the user
is privileged to administer the cluster, `kubectl` command is configured
locally and `helm` is available.

In order for the edge component to be able to successfully forward data 
to the central service, ElasticSearch must be accessible by fluent.
The topic of service exposure has been mentioned above.
The example expose command in this case is:
```sh
kubectl expose service ai-sprint-monit-elasticsearch --port=9200 \
        --name=ai-sprint-monit-elasticsearch-external \
        --external-ip=$EXTERNAL_INTERFACE_ADDRESS
```

The end user will expect access to Kibana. Analogous principles apply
in this case. Example expose command is:
```sh
kubectl expose service ai-sprint-monit-kibana --port=5601 \
        --name=ai-sprint-monit-kibana-external \
        --external-ip=$EXTERNAL_INTERFACE_ADDRESS
```
Kibana dashboard can be accessed with a web browser by entering:
{cloud cluster address}:5601.


### Log forwarder

To deploy the log forwarder component:
```sh
helm install --set elasticSearchHost=$CLOUD_CLUSTER_ADDRESS \
        --set elasticSearchPort=$EXPOSED_ES_PORT \
        $DEPLOYMENT_NAME ams-log-forwarder.tar.gz -n $K8S_NAMESPACE
```
or use a helm values file with corresponding values names.

No external exposures are expected for edge component.

As mentioned above, the central ElasticSearch instance must be accessible
by fluent.


App integration
---------------

From developer's perspective only standard Python logging library is being
used to gather the log messages transparently.

To integrate with the logging framework run:
```py
from aisprint.monitoring import initialise_logging, tear_down_logging
...
ctx = initialise_logging()
...
tear_down_logging(ctx)
```
Be sure to eventually call the tear down function, as data is buffered
underneath (for efficiency) and might be lost in case of a premature
program termination.

To log a message use standard Python constructs, such as:
```py
import logging
...
logger = logging.getLogger('my_name')
...
logger.info('an info message')
```

Hint: tags can be used, e.g.:
```py
logger.debug({'message': 'here has been performed calculation of magic values',
              'magic_number': 5,
              'magic_label': 'a_label'})
```
Using this construction the message contents will additionally be tagged
and later it will be possible to:
*   filter messages based on tags,
*   dynamically draw charts, histograms,
*   sort by values, ranges, labels.


Analogously to AMS core, the following environment variables are expected
at runtime:
*   `MONIT_LOG_HOST`, `MONIT_LOG_PORT` - the address to ElasticSearch REST API,
*   `COMPONENT_NAME` - the name of current component, an AI-SPRINT standard
    environment variable.


Basic usage
-----------

The logging framework is an independent component. It does not rely on
the AMS core. It can be deployed alongside it.

The cloud component should be deployed *once*, preferably in a central (from
AI-SPRINT perspective) location. ElasticSearch must be reachable by all
edge components. The log forwarders should be deployed once on every cluster
that is supposed to log data into ElasticSearch.

Underneath the log messages are being sent to a *local* fluentd instance,
which works as a caching proxy with filters between applications
and ElasticSearch. There are no obstacles to having a standard console
or file log handler and using the AMS logging framework.

Underneath a local file cache is used before writing messages
to ElasticSearch. In case of connectivity disruptions the file acts
as a buffer and is flushed only after the messages have successfully
been written to the database. In case of a dirty buffer the file will
survive cluster shutdowns.
