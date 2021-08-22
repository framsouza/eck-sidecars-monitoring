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

### [configmap-filebeat.yml](https://github.com/framsouza/eck-sidecars-monitoring/blob/main/configmap-filebeat.yml)

Here, I am enabling elasticsearch module and configuring in which directory filebeat will find the elasticsearch output logs regarding gc, audit, slowlogs, deprecation, and server logs. Be sure to adjust it accordingly with your elastichsearch logs files.
As I mentioned, we are using an external monitoring cluster running in our Elastic Cloud, it means I am using the _cloud.id_ and _cloud.auth_ reference to send logs to my cluster running on Cloud and using processors to collect cloud and host metadata to enrich our data.

### [configmap-metricbeat.yml](https://github.com/framsouza/eck-sidecars-monitoring/blob/main/configmap-metricbeat.yml)
Same as filebeat, we need to enable the elasticsearch module in order to collect infromation and metrics about the engine. Here, I am defining some metricsets to be collect each 10s. As it's running as sidecar container, I am refering to Elasticsearch using localhost, 9200 port and using SSL certificate. The output is the monitoring cluster running on Cloud and instead of use _output.elasticsearch_ we should use _cloud.id_ and _cloud_auth_

Once we created both configmaps, we are ready to deploy elasticsesarch.

### [es.yml](https://github.com/framsouza/eck-sidecars-monitoring/blob/main/es.yml)

The name of my cluster is _cluster1_ (be more creative in your tests), running 3 nodes (7.14.0) 
The http session is necessary because in the metricbeat configuration we are telling metricbeat to connect in elasticsearch using HTTPS, which means I am defining a custom domain name to be used with the self-signed certificate, in this case, _localhost_.
At the configuration level, we are enabling elasticsearch monitoring and disabling the _xpack.monitoring.elasticsearch.collection.enabled_ because we are not interested in [legacy monitoring data](https://www.elastic.co/guide/en/elasticsearch/reference/7.9/collecting-monitoring-data.html)

```
apiVersion: elasticsearch.k8s.elastic.co/v1
kind: Elasticsearch
metadata:
  name: cluster1
spec:
  version: 7.14.0
  http:
    tls:
      selfSignedCertificate:
        subjectAltNames:
        - dns: localhost 
  nodeSets:
  - name: elasticsearch
    count: 3
    config:
      xpack.monitoring.collection.enabled: true
      xpack.monitoring.elasticsearch.collection.enabled: false
```

At the _podTemplate_ level, instead of change the _node.store.sllow.mmap_ I am increasing the _max_map_count_ (recommended for production environments), you can read more about it [here](https://www.elastic.co/guide/en/cloud-on-k8s/1.7/k8s-virtual-memory.html).
I am also giving 4Gi of memory elasticsearch container, it will automatically give me 2Gi of heap. There's also an environment variable defined called _ES_LOGS_STYPE_ this is for rolling file appender, you can also read more about it [here](https://github.com/elastic/elasticsearch/pull/65778)

That's all regarding the elasticsearch container, you don't need any other additional information, now let's jump into metricbeat container definition.


```
    podTemplate:
      spec:
        initContainers:
        - name: sysctl
          securityContext:
            privileged: true
          command: ['sh', '-c', 'sysctl -w vm.max_map_count=262144']
        containers:
          - name: elasticsearch
            resources:
              requests:
                memory: 4Gi
              limits:
                memory: 4Gi
            env:
            - name: ES_LOG_STYLE
              value: file
```

### [metricbeat container](https://github.com/framsouza/eck-sidecars-monitoring/blob/main/es.yml#L35)

Metricbeat is running version 7.14.0 and we saying the metricbeat.yml file should be read from the following path: _/etc/metricbeat.yml_. These args will be used to start the container.

Then  you will find a set of environment variables that are used on the metricbeat.yml / confimap-metricbeat.yml
Once we have all the environments regarding user/pass and ULR, You will find a _volumeMounts_ block, which is used to mount the configmap and secrets.

That's all about metricbeat, in summary it's composed of environment variables and volumes.

### [filebeat container](https://github.com/framsouza/eck-sidecars-monitoring/blob/main/es.yml#L77)

As all the components of the stack, filebeat is running verstion 7.14.0, and the started file is located in _/etc/filebeat.yml_.
As well as metricbeat, filebeat is also composed of environment variables and volumes.
The environment variable is used to establish connections (definition of users and password) and the volumes are used to mount configuration files and ssl certificates.

At the end of the session you will see _volumes_ and _volumeClaimTemplates_, the _volumes_ is used to define from where the volume must be mounted, and this session needs to be defined only once. Inside each container, you should refer to it on the _volumeMounts_ session.
The _volumeClaimTemplate_ is used to define how many GB I will have to store elasticsearch data (adjust it adequately) 

### [kibana.yml](https://github.com/framsouza/eck-sidecars-monitoring/blob/main/kibana.yml)

Last by not least, kibana requires only the elasticsearchRef and monitoring configuration enabled.

```
apiVersion: kibana.k8s.elastic.co/v1
kind: Kibana
metadata:
  name: kibana-cluster1
spec:
  version: 7.14.0
  count: 1
  elasticsearchRef:
    name: cluster1
  config:
    monitoring.ui.container.elasticsearch.enabled: true
```

### [ingress.yml](https://github.com/framsouza/eck-sidecars-monitoring/blob/main/ingress.yml)

In oder to be able to access Kibana using my own domain, I am using an ingress controller. With that, I will access Kibana via https://framsouza.co. If you want to know how to setup SAML configuration, you can take a look at [this article](https://github.com/framsouza/eck-saml-hot-warm-cold)


### [Stack monitoring](https://www.elastic.co/guide/en/kibana/current/xpack-monitoring.html#xpack-monitoring)

If you access your central monitoring cluster on Cloud and click on Stack Monitoring, you should see the following:

<img width="1667" alt="Screenshot 2021-08-22 at 20 02 15" src="https://user-images.githubusercontent.com/16880741/130365348-472ba083-0c32-4d82-a72d-69afb31dfa86.png">

All information you need to check regarding your production cluster, you can see it centralized in a single page, you can jump into Nodes, Indices or logs details, it will also provide pre buil dashboards and ILM configurations.

Cheers!
