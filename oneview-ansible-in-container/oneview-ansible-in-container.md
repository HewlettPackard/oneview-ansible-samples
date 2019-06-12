# Running HPE oneview-ansible in a container

## Introduction

This how to guide and accompanying [GitHub repository](https://github.com/HewlettPackard/oneview-ansible-samples/tree/master/oneview-ansible-in-container) will show how to use the HPE [oneview-ansible](https://github.com/HewlettPackard/oneview-ansible) module from a containerized version on the [Docker Store](https://store.docker.com/community/images/hewlettpackardenterprise/oneview-ansible-debian) to manage and configure HPE infrastructure.

## HPE oneview-ansible in a container

Docker containers provide a low friction way to get developer environments and CI/CD pipelines up and running easily and consistently. But they do present challenges. They are isolated by design and are immutable. Somehow the oneview-ansible container needs to get access to the playbooks to run.

### Setting up the playbooks

Docker volumes are storage that can be shared among containers and persist beyond the lifecycle of containers. Begin by creating a Docker volume

```bash
docker volume create --name playbooks
```

Next run a container that can access the sample playbook and allow you to edit the playbooks.

```bash
docker run -it --rm -v playbooks:/playbooks --entrypoint /bin/ash alpine/git
```

This will run an Alpine Linux container with git installed. The arguments run the container in interactive mode (-it), remove the container on exit (--rm), mount the Docker volume `playbooks` at the mount point `/playbooks`, and override the default command (git) with the `ash` shell.

Use git to get the sample ansible playbooks:

```bash
cd /playbooks
git clone https://github.com/HewlettPackard/oneview-ansible-samples.git
```

Edit the configuration file `oneview-config.json' and provide the IP address and credentials for the OneView appliance or Synergy composer:

```bash
cd /playbooks/oneview-ansible-samples/oneview-ansible-in-container
vi oneview-config.json
```

*Editor's note: Let's pause while developers read their favorite [exiting vim humor](https://stackoverflow.blog/2017/05/23/stack-overflow-helping-one-million-developers-exit-vim/)*

### Running the playbook

At this point you are ready to run the sample playbook. Either exit the alpine/git container or open another command window and run the `oneview-ansible-debian` container:

```bash
docker run -it --rm -v playbooks:/playbooks hewlettpackardenterprise/oneview-ansible-debian /bin/bash
cd /playbooks/oneview-ansible-samples/oneview-ansible-in-container
```

You are now in a bash shell ready to run playbooks with the `oneview-ansible` modules. Enclosure Groups represent the top of the hierarchy of OneView resources. The following command will get information about all Enclosure Groups:

```bash
ansible-playbook -i hosts enclosure_group_facts.yml
```

### Cleanup

When you no longer need the shared volume it can be removed with:

```bash
docker volume rm playbooks
```

### Conclusion

You now have an environment to run ansible playbooks using the HPE OneView module in a container. You can switch to other directories in the cloned repository and try other samples. Additional samples can be found in the [oneview-ansible](https://github.com/HewlettPackard/oneview-ansible) repository under `examples`.

Copyright (2018) Hewlett Packard Enterprise Development LP
