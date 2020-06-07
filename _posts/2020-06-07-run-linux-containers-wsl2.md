---
title: Run Linux containers on Windows 10 using WSL 2
date: 2020-06-07 00:00:00 +0000
description: How to run Linux containers in Windows 10 using Windows Subsystem for Linux and Docker Desktop
categories: [Windows Subsystem for Linux]
tags: [WSL2,Docker]
header:
 teaser: "/assets/img/posts/teasers/docker.png"
permalink: /wsl2/run-linux-containers/
excerpt: The Windows Subsystem for Linux lets you run a Linux environment on Windows, without creating a virtual machine. WSL 2 is the latest version of the Windows Subsystem for Linux, generally available in Windows 10, version 2004 (May 2020 update), which uses a full Linux kernel. To run the containers under WSL 2, you need to install Docker Desktop for Windows. 
---
### What is Windows Subsystem for Linux (WSL)?
The Windows Subsystem for Linux lets you run a Linux environment on Windows, without creating a virtual machine. 
WSL 2 is the latest version of the Windows Subsystem for Linux, generally available in Windows 10, version 2004 (May 2020). One of the main improvements over version 1 is the usage of full Linux kernel, specially tuned for WSL 2, optimizing for size and performance to provide a fantastic Linux experience on Windows. More information of WSL is available in the Microsoft [docs](https://docs.microsoft.com/en-us/windows/wsl/)

### Windows Terminal
Windows Terminal is an excellent terminal application that you can use to access, for example, command prompt, Powershell, Cloud Shell, and WSL. You can use multiple tabs, panes, set a background, customize colors, create your shortcut keys, and much more. Visit the official documentation [page](https://docs.microsoft.com/en-us/windows/terminal/) to learn more about this very powerful tool. 

### Install WSL 2 and Docker Desktop
To be able to run containers under WSL 2, you need to install a Docker Desktop. There are four pre-requirements before installing the Docker Desktop:
1. You need Windows 10, version 2004 (May 2020 update)
2. Enabled WSL 2 feature in Windows 10
3. The Linux kernel upgrade package for WSL 2
4. Linux distribution up and running under WSL 2

In the Microsoft official [documentation](https://docs.microsoft.com/en-us/windows/wsl/install-win10), you can find instructions on how to enable the WSL feature, upgrade WSL version 1 to version 2 and how to install your favorite distro under WSL.
After satisfying the pre-requirements, download the latest Docker Desktop installation [package](https://hub.docker.com/editions/community/docker-ce-desktop-windows/) and start the installation process. When prompted, choose to use the WSL 2 based engine. 

### Run a container
Using Windows Terminal or the Linux distribution shortcut from the Start menu, open the console connection to your Linux distribution running under WSL. 
1. Check if docker command is accessible via the Linux distribution
    ```shell
    docker --version
    ```
    If the above command displays the docker version, proceed with step 2. Otherwise, restart the computer if you didn't restart it after the Docker Desktop installation.
2. Start a Nginx container
    ```shell
    docker run --name nginx-demo -p 8080:80 -d nginx
    ```
    This command will download the latest Nginx container from the public repository, run the container and map the localhost port 8080 with the container port 80.
3. Open your favorite web browser and navigate to http://localhost:8080. You will see the Wellcome to nginx! page
    ![Desktop View]({{ "/assets/img/posts/wsl2/nginxContainer.png" | relative_url }})

### Cleanup the resources
To delete the running container and downloaded Nginx image, follow these steps:
1. Stop the running Nginx container
    ```shell
    docker stop nginx-demo
    ```
2. Delete the container
    ```shell
    docker rm nginx-demo
    ```
3. Delete the Nginx image
    ```shell
    docker rmi nginx
    ```