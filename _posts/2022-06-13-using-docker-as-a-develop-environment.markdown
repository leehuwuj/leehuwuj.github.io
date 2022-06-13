---
layout: post
title:  "Using docker container as a develop environment"
date:   2022-06-13 13:00:00 +0700
categories: engineering
tag: [devops,docker]
---

You are using a Macbook with Apple silicon chip (M1, M2) which currently have tons of error in architecture compatible or you have to use your company Windows laptop which is not comfortable to development. Yes, there’s a simple approach to resolve it: The Docker.

Hmm, you already know Docker and are using it to containerize the application right? But have you ever using it as a development environment? Believe me, it’s works like a champ.

Let’s start with a simple demonstration: A python project.

## Context:

I’m using the Apple M1 Pro chip which running Mac OS Montery. It has a trouble with a python library called [python-ldap][python-ldap-pip] which could not integrate with LDAP though SSL protocol, i tested the same code in a Linux machine and linux container but it works well, so there is a trouble with the OS environment.
## Develop your code and run it by Docker:

In any development, there’s 3 main things that you would like to aware: The IDE, terminal and network. 

If you are familiar with docker i think there’s no issue with network but in some special case, you will need port-forwarding to integrating with other networks.

The IDE and terminal are things that we most interact with. You often store the source code in your host machine, build the docker image then run with to docker daemon right? But, it’s not interactively which we need to rebuild the image every time changed code.

There are 2 ways to make you comfortable with docker when development on it. Using VSCode or mounted volume.

### Using remote Docker mode in VSCode:

Visual studio code is a great IDE which are being used in various of development languages. It’s lightweight but enough functions to help you in almost the case.

With Docker extensions, you can edit your container code remotely without rebuild the image, it’s also support the IDE extensions remotely so everything should work fine.

You can easily manage files or make it powerful by using Attach Visual Studio Code with docker container:

<img src="/images/docker-container-develop-environment/container-attach.png" width="200" height="200" class="align-center"/>

In Docker remote mode, you can also install the vscode extensions like Python, Vim, Debugger,… everything should works like your host.

![left-aligned-image](/images/docker-container-develop-environment/vscode-docker-remote.png){: .align-center}

## Using mounted volume:

Files in docker container are internally so everything that edited in docker container remotely will be clean when you remove it. To resolve this trouble, you can easily mount your source code in the host to container. Example:

```bash
docker run \
  -d -t \
  -v <your_source_code_path>:<your_container_workdir_path> \
  <your_image> \
  bash
```

The arguments `-d` `-t` to attach the shell in background that allows the container still running without existed.

With this approach, you can using any IDE to edit your code in your host machine then just attach a shell to run/debug your code:

```bash
docker exec -it container_name bash
```

*Note*: the `bash` cli may not available in your container then you may try the `sh` or `shell` cli.

[python-ldap-pip]: https://pypi.org/project/python-ldap/