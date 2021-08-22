# eck-sidecars-monitoring

_(Work in Progress)_

This page will guide you through how to monitoring ECK clusters using filebeat and metricbeat as sidecar containers, sending monitoring information to an external monitoring cluster running on Elastic Cloud.
As part of our best practices and recommendation, utilize an external monitoring cluster is always recommended, with that setup we avoid overload our current production environment with monitoring data. We also can run queries to collect monitoring/audit information without care about the performance impact on the production cluster.

In the scenario, we have ECK (1.7.0) running Elasticsearch version 7.14.0, we recently released the ability to configure Stack monitoring using sidecards, instead of deploying beats, daemonsets and other resouces to collect ECK information, you can run it as a sidecard container, it means all the connection between filebeat/metricbeat is done via localhost, it also worth to remember sidecar containers on Kubernetes don't share resources with the main container (except for the network) so there would be no resource contention between the two containers.

### Resources
You are going to find the following resources:
- ingress controller
- elasticsearch 
- kibana
- filebeat & metricbeat configmap

### Scenario
Let's assume you have elasticsearch up and running on top of Kubernetes and it was deployed by ECK, and you store production data into this environment. Somehow you must implement proper monitoring for this cluster because it carries valuable data and you want to collect metrics and logs information.
You have some options to have it implemented,
- Centralized monitoring cluster
- Dedicated monitoring
- Self-monitoring cluster

We do not recommend self-monitoring cluster, due to the fact it will share the production environment resource to handle also monitoring data, and in most cases it will overload and cause performance issues in your production cluster. What we are going to set up here is option 1, centralized monitoring cluster. Imagine you have 3 elasticsearch running (ECK, self-managed, or even running in our cloud) and all the monitoring data for these clusters will be sent to a central monitoring cluster, where you can run complex queries to understand and visualize your data.

These examples will contemplate and production environment running on ECK and sending monitoring data to an external elasticseasrch running in our Elastic Cloud (ESS)

*Keep in mind it will collect only elasticsearch monitoring data, it's related to the Stack Monitoring and not with your application data*.

### First thing first

Before understand and deploy elasticsearch resource, we need to create first the metricbeat and filebeat confimap, if you deploy elasticsearch without deploy them first, we are going to get an error.
The configmap-filebeat.yml and configmap-metricbeat.yml file are respectively filebeat and metricbeat yml files. This is where you should define the proper settings. ECK will mount and refer to it as a confimap and be mounted as a volume inside the elasticsearch container.

### [configmap-filebeat.yml](https://github.com/framsouza/eck-sidecars-monitoring/configmap-filebeat.yml)

Here, I am enabling elasticsearch module and configuring in which directory filebeat will find the elasticsearch output logs regarding gc, audit, slowlogs, deprecation, and server logs. Be sure to adjust it accordingly with your elastichsearch logs files.
As I mentioned, we are using an external monitoring cluster running in our Elastic Cloud, it means I am using the _cloud.id_ and _cloud.auth_ reference to send logs to my cluster running on Cloud and using processors to collect cloud and host metadata to enrich our data.

### [configmap-metricbeat.yml](https://github.com/framsouza/eck-sidecars-monitoring/configmap-metricbeat.yml)
Same as filebeat, we need to enable the elasticsearch module in order to collect infromation and metrics about the engine. Here, I am defining some metricsets to be collect each 10s. As it's running as sidecar container, I am refering to Elasticsearch using localhost, 9200 port and using SSL certificate. The output is the monitoring cluster running on Cloud and instead of use _output.elasticsearch_ we should use _cloud.id_ and _cloud_auth_
