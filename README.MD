## Elastic Filebeat for AWS ECS

Using this simple docker container, we are able to ship all the docker logs coming out of a ECS (EC2) instance, from the instance to an Elastic cluster
We are able as well to customize which container logs get picked up by the filebeat using docker labels.

The target containers should obviously produces logs and write them to the file system.
In the case of ECS, we should make sure they are configured to use the `json-file` log driver.

### Content

#### Dockerfile

This is a simple dockerfile based on the [elastic filebeat container](https://www.docker.elastic.co/r/beats/filebeat:7.15.2) container
We build a new version of the image and integrate our custom configuration.

#### Filebeat configuration.
The configuration is found in `filebeat.yml`
The complete filebeat reference can be found [here](https://www.elastic.co/guide/en/beats/filebeat/current/filebeat-reference-yml.html?baymax=rec&rogue=rec-1&elektra=guide)

The major changes that was introduced into the filebeat.yml was related to filter logs picked by filebeat on the ECS instance.
```filebeat.autodiscover:
   providers:
     - type: docker
       templates:
         - condition:
             contains:
               docker.container.labels.filebeat.enable: 'true'
           config:
             - type: container
               paths:
                 - /var/lib/docker/containers/${data.docker.container.id}/*.log
```
We declare a provider of type `docker` and we setup some conditions.

Filebeat will gather logs from other containers only if the matches the set of conditions we specify here.
In our case, using this config, all containers with the label `filebeat.enable` with value `true` will be used.
We also specify the path for the logs on the ECS instance, so filebeat knows where to retrieve those logs.
More configurations are available and can be found [here](https://www.elastic.co/guide/en/beats/filebeat/7.15/configuration-autodiscover.html)

#### Task definition

You can also find in the `.aws` folder an example of a target task definition template for our filebeat image.
In order for filebeat to access the los properly, you should make sure of the following:

##### Volumes

We create two volumes 
In our case, we called them logs and sock and assign them to the following path
- Volume logs: /var/lib/docker/containers
- Volume sock: /var/run/docker.sock
Those volumes corresponds to some path on the ECS instance.

We then mount those volumes inside our filebeat container.
This is done in our task configuration (or through the AWS task definition builder)
```
      "mountPoints": [
        {
          "readOnly": true,
          "containerPath": "/var/run/docker.sock",
          "sourceVolume": "sock"
        },
        {
          "readOnly": null,
          "containerPath": "/var/lib/docker/containers",
          "sourceVolume": "logs"
        }
      ],
```

##### Access
The filebeat container was launched with user `root` as well as setting the `privileged` attributes to `true`


### How to use
Using this Dockerfile, we can create our own image and further customize it.

The following parameters should be set:

If we are using the cloud version of elastic
- the cloud id 
- the elastic password 

Otherwise
- The elastic search host
- The elastic search username
- The elastic search password
Once done, we can create our image with the following command
e.g. 
The following command create a new multiplatform image and push it to a docker repository
```
$ docker buildx build --platform linux/arm64/v8,linux/amd64 -t {{inser_your_docker_cloud_name}}/ecs-elastic-filebeat:v1.0.0 --push  .
```

### Gotchas

It could happen that your filebeat container does not have access to `/var/run/docker.sock` on the ECS instance. 
To fix that, you can assign the proper role to your task.
