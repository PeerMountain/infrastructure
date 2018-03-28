### Peer Mountain Infrastructure

Hello there, how are you doing? This document explains a little bit how the infrastructure is configured today.


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

## Docker
Docker Swarm is running in all hosts.

## Hadoop Infrastructure
The Hadoop infrastructure is managed by the [Ambari Project](https://ambari.apache.org/). The services used are HDFS, HBase, ZooKeeper and Ambari Metrics.

You can access the administration dashboard on `peer-mountain01` with a SSH tunnel. First, run the command below:

```bash
$ ssh -f -L 8080:localhost:8080 root@peer-mountain01 -N
```

And access your browser at `http://localhost:8080`.
