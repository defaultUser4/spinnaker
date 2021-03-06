# Getting Started using Docker Compose

This will run through getting a version of Spinnaker running locally that can deploy server groups to a DC/OS cluster.

For testing, the easiest way to get started is by running a full Spinnaker stack locally using the experimental [docker-compose](https://github.com/spinnaker/spinnaker/tree/master/experimental/docker-compose) in the main Spinnaker repository. For the most part, the instructions there can be followed. Most of this document details the additional specific DC/OS configuration that is needed.

##### Supported Functionality/Limitations
* This has been tested against DC/OS Enterprise version 1.9, but not the open source version.
* Server Groups
  * Create, resize, clone, and destroy supported
  * Disable operation - this is equivalent to a "Suspend" in DC/OS. However, in most other Spinnaker cloud providers, this takes a service out of load balancing instead of suspending tasks. 
* Load Balancers
  * For now, load balancers are implemented using marathon-lb instances. Functionality is mostly limited to managing the marathon-lb instance since it automatically routes to instances rather than requiring direct interaction.
  * Create, edit, delete supported
* Security groups
  * No support
* Pipelines
  * Stages supported: Check preconditions, Deploy, Destroy Server Group, Disable Cluster, Disable Server Group, Find Image from Cluster, Jenkins, Manual Judgement, Pipeline, Resize Server Group, Run Job, Scale Down Cluster, Script, Shrink Cluster, Wait
* Server groups created in Spinnaker will be placed under a marathon group in DC/OS that corresponds to the account name you have defined in Spinnaker. For example, if you are using an account named "dcosUser" to create a server group with the name "service1", the Marathon application id will be "/dcosUser/service1-v000".
  * Along the same lines, server groups in Spinnaker are expected to have a certain naming format. DC/OS applications that don't have that format or exist in the Spinnaker account will not show up in Spinnaker.

## Local Installation

Follow the steps the spinnaker [docker-compose](https://github.com/spinnaker/spinnaker/tree/master/experimental/docker-compose) README, with the following updates/changes for DC/OS:

When running `spinnaker/experimental/docker-compose/docker-compose.yml`:

* Update the `clouddriver`, and `deck` entries to reference the images with DC/OS support:

  ```
  clouddriver:
    image: cerner/clouddriver:latest
    
  deck:
    image: cerner/deck:latest
  ```

In the `spinnaker/config` directory:
* Add DC/OS config in `clouddriver-local.yml` (create if it doesn't exist)  

  The loadBalancer section needs to have the marathon-lb image defined, as well as a serviceAccountSecret (if applicable).
  ```
  dcos:
    enabled: ${providers.dcos.enabled:false}
    clusters:
      - name: default
        dcosUrl: http://dcos.example.com
        loadBalancer:
          image: mesosphere/marathon-lb:v1.5.0
          serviceAccountSecret: marathon_lb
    accounts:
      - name: some-user-account
        environment: test
        clusters:  
          - name: default
            uid: dcosUserName
            password: ${DCOS_USER_PASSWORD}
      - name: some-service-account
        environment: test
        clusters:  
          - name: default
            uid: dcosServiceAcctName
            serviceKeyData: ${DCOS_SERVICE_ACCOUNT_KEY}
  ```
  
* Add any DCOS_USER_PASSWORD or DCOS_SERVICE_ACCOUNT_KEY values to `spinnaker/experimental/docker-compose/compose.env`

  In this example config, both a user account and service account are shown. For testing, the serviceKey can also be set as the Base64 encoded key directly, e.g.
  ```
  dcos:
    enabled: ${providers.dcos.enabled:false}
    accounts:
      - name: some-service-account
        environment: test
        clusters:  
          - name: default
            uid: dcosServiceAcctName
            serviceKeyData: |
                -----BEGIN PRIVATE KEY-----
                MIIEvQIBADANBgkqhki ...
                -----END PRIVATE KEY-----
  ```

  Using an environment variable instead comes in handy if deploying Spinnaker on DC/OS Enterprise, where a secret could be used to populate the environment variable.

* If the DC/OS cluster has a strict or permissive [security mode](https://docs.mesosphere.com/1.8/administration/installing/custom/configuration-parameters/#security) set, the [root certificate](https://docs.mesosphere.com/1.8/administration/tls-ssl/get-cert/) from the DC/OS CA will have to be supplied in the `clouddriver-local.yml` config to allow Spinnaker to talk to DC/OS.
  ```
  dcos:
    enabled: ${providers.dcos.enabled:false}
    clusters:
      - name: default
        dcosUrl: http://dcos-url.com
        caCertFile: /path/to/root/certificate
        loadBalancer:
          image: mesosphere/marathon-lb:v1.5.0
          serviceAccountSecret: marathon_lb
  ```

  `caCertData` can also be supplied instead of `caCertFile` - it is expected to be the Base64 encoded certificate data (NOT including the -----BEGIN CERTIFICATE----- and -----END CERTIFICATE----- markers).

* Add `dcos` to the providers section in `spinnaker-local.yml` (create if it doesn't exist):
  ```
  providers:
    dcos:
      enabled: true
      primaryCredentials:
        name: some-service-account
  ```
  
### DC/OS Permissions

The user or service account should have the following permissions in DC/OS Enterprise (where `SPINNAKER_ACCOUNT_NAME` is replaced with the `dcos.accounts.name` value configured above):

| Resource       | Actions        |
| :------------- | :------------- |
| dcos:mesos:agent:executor:app_id:/ | read |
| dcos:mesos:agent:executor:app_id:/test | read |
| dcos:mesos:agent:framework:role:* | read |
| dcos:mesos:agent:sandbox:app_id | read |
| dcos:mesos:agent:sandbox:app_id:/test | read |
| dcos:mesos:agent:task:app_id:/test | read |
| dcos:mesos:agent:task:app_id:test | read |
| dcos:mesos:master:framework:role:* | read |
| dcos:mesos:master:task:app_id:/ | read |
| dcos:mesos:master:task:app_id:/test | create, read |
| dcos:secrets:list:default:/ | read |
| dcos:service:marathon:marathon:services:/       | read |
| dcos:service:marathon:marathon:services:/`SPINNAKER_ACCOUNT_NAME` | create,delete,read,update |
| dcos:service:metronome:metronome:jobs:/ | create,delete,read,update | 

**NOTE:** There is currently a marathon permissions bug that prevents instance details from being retrieved.  Until that is resolved, the only workaround we've found so far is to give the user/service account superuser permissions.  Without superuser permissions, Spinnaker is still usable but you are unable to view instance details through the Spinnaker UI.


### Docker Registry configuration

In order to deploy Docker images to DC/OS a Docker registry account must be configured in Spinnaker. This can be configured following the same steps as in the Kubernetes deployment documentation: http://www.spinnaker.io/v1.0/docs/target-deployment-configuration#section-docker-registry

## Running

Now you should be able to run ```DOCKER_IP=`docker-machine ip default` docker-compose up -d``` (or if using Docker for Mac, ```DOCKER_IP=localhost docker-compose up -d```). After all the Spinnaker services are fully started, you should be able to access deck at `http://localhost:9000`
