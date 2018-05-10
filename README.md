### Peer Mountain Infrastructure

Hello there, how are you doing? This document explains a little bit how the infrastructure is configured today.

## Services Information
* [Docker](#docker)
* [Elasticsearch](#elasticsearch)
* [GitLab](#gitlab)
* [Hadoop Infrastructure](#hadoop-infrastructure)
* [RabbitMQ](#rabbitmq)
* [Writer and Reader](#writer-and-reader)

## Deploying and Developing
* [Developing Locally](#developing-locally)
* [Deploying in Production](#deploying-in-production)

## Docker
Docker Swarm is running in all hosts.

You can visualizar the Docker Swarm status with Portainer. To access it, create the following SSH tunnel and access it through the web interface at [http://localhost:9000](). The password is at Maecenas TeamPass.

```bash
$ ssh -f -L 9000:localhost:9000 root@peer-mountain01 -N
```

## Elasticsearch
This services has some details:

* It uses the folder `/data/elasticsearch` on each node to store its data, so *just one container* must run on each node. If you take a look at `elasticsearch/docker-compose.yml` you will see that each service has a `replica=1` and a label constraint.
* It uses its own HA feature to estabilish the cluster.
* It has a reverse TCP HAProxy to distribute the load, `elasticsearch-proxy`, so any services accessing it should access it.

## GitLab
The `peer-mountain01` has a GitLab instance installed locally, where this and every code is hosted.

Since we cannot open a lot of port externally, we need a SSH config do pull/push code, you can configure yours like this:

```
$ cat ~/.ssh/config
...

Host peer-mountain01
    HostName 94.130.38.47
    User git
    IdentityFile /path/to/your/key
    IdentitiesOnly yes
```

To access the GitLab web interface, you need a SSH tunnel. To run that, you need another SSH configuration (this time with a root user – you can add the two configurations, don't worry):

```
$ cat ~/.ssh/config
...
Host peer-mountain01
   Hostname 94.130.38.47
   User root
   IdentityFile /path/to/your/key
```

And run the command:
```bash
$ ssh -f -L 9352:localhost:9352 root@peer-mountain01 -N
```

The pipeline from this project runs on `peer-mountain01`, where Ansible is installed. To run as root, we gave root permissions to the `gitlab-runner` on the host:

```bash
# add the user to the sudo group
$ sudo usermod -a -G sudo gitlab-runner

# add this line to /etc/sudoers
gitlab-runner ALL=(ALL) NOPASSWD: ALL
```

## Hadoop Infrastructure
The Hadoop infrastructure is managed by the [Ambari Project](https://ambari.apache.org/). The services used are HDFS, HBase, ZooKeeper and Ambari Metrics.

You can access the administration dashboard on `peer-mountain01` with a SSH tunnel. First, run the command below:

```bash
$ ssh -f -L 8080:localhost:8080 root@peer-mountain01 -N
```

And access your browser at `http://localhost:8080`.

## RabbitMQ
This services has some details:

* It uses the folder `/data/rabbitmq` on each node to store its data, so *just one container* must run on each node. If you take a look at `rabbitmq/docker-compose.yml` you will see that each service has a `replica=1` and a label constraint.
* It uses Consul to establish its cluster. Note that Consul is deployed *just* for that.
* It has a reverse TCP HAProxy to distribute the load, `rabbitmq-proxy`, so any services accessing it should access it.

## Writer and Reader
These two services need access to the HBase, but its master service only runs in `peer-mountain01`, so `writer` and `reader` *must* run on this node. This is accomplished with this deployment configuration:

```yaml
# Snippet from {writer/reader}/docker-compose.yml

deploy:
  replicas: 1
  placement:
    # Since only peer-moutain01 is the manager, it works
    constraints: [node.role == manager]
```

---

## Developing Locally
Each service has its own `docker-compose-development.yml` file to be used locally for development. The benefits of using it are:

* The code from the service is mounted as a volume inside the container, so every change on the code will be reflected automatically.
* *Some* services have code watch and autoreload to restart the services when the code is updated. If the service hasn't, just restart the container.
* All the services dependencies are bundled in the file, like databases and queues.
* All the services are in a network called `mvp`, so you can connect all of them locally. 

To start, create the `mvp` network needed:
```bash
$ docker network create mvp --driver=bridge
```

For example, to run the `writer` service locally, you must execute the following commands (you may run them in different consoles, to see their logs):

```bash
$ docker-compose up -f docker-compose-development.yml up hbase
$ docker-compose up -f docker-compose-development.yml up elasticsearch
$ docker-compose up -f docker-compose-development.yml up rabbitmq
$ docker-compose up -f docker-compose-development.yml up writer
```

Keep in mind that some services *share* some dependencies, like `writer` and `reader` shares HBase, Elasticsearch and RabbitMQ. So if you want to get the whole infrastructure locally, you must initalize them just once.

Here's a script to initalize the whole infrastructure locally. Is assumes you have the repositories flatly (all of them in the same folder):

```bash
$ tree -L 1
.
├── authorizer
├── consul
├── elasticsearch
├── himalaya-models
├── infrastructure
├── portainer
├── rabbitmq
├── reader
├── teleferic
└── writer
```

```bash
#!/bin/bash

echo "Initializing HBase..."
cd writer && git submodule init && git submodule update && docker-compose -f docker-compose-development.yml up -d hbase && cd ..

echo "Initializing ES..."
cd writer && docker-compose -f docker-compose-development.yml up -d elasticsearch && cd ..

echo "Initializing RabbitMQ..."
cd writer && docker-compose -f docker-compose-development.yml up -d rabbitmq && cd ..

echo "Initializing Writer..."
cd writer && docker-compose -f docker-compose-development.yml up -d writer && cd ..

echo "Initializing Reader..."
cd reader && git submodule init && git submodule update && docker-compose -f docker-compose-development.yml up -d reader && cd ..

echo "Initializing Authorizer..."
cd authorizer && docker-compose -f docker-compose-development.yml up -d authorizer && cd ..

echo "Initializing Teleferic..."
cd teleferic && docker-compose -f docker-compose-development.yml up -d teleferic && cd ..

```

After every service is up, you neeed to *bootstrap* the environment:

Initialize the HBase tables:


```bash
# Logs into the Writer container
$ docker exec -ti $(docker ps -aqf 'name=writer_writer_1') bash

# Open the Python interpreter
$ ipython

# Run the following Python code
from himalaya_models.backends import utils
utils.create_hbase_message_table()
utils.create_hbase_persona_table()
```

From the same console, Initialize the *Genesis Persona*:

```python
from himalaya_models.models import Persona


pubkey = "genesis_persona_pubkey_here"
address = "genesis_persona_address_here"
nickname = "genesis_persona_nickname_here"

Persona(address=address, nickname=nickname, pubkey=pubkey).save()
```

Now you should be ready to use the platform locally.

## Deploying in Production
To deploy any service in production, just merge the code on the `master` branch and create a tag (you can do it directly in the GitLab) interface.
