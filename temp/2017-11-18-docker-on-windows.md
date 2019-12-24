---
title: Running Docker on Windows 10
tags: [windows-10, docker]
image:
  feature: header/h11.svg
---
Running Docker, or specifically running Docker on your Windows 10 machine, sounds simple? Well yes in the most part it is, Docker provide [an installer and setup](https://store.docker.com/editions/community/docker-ce-desktop-windows) to do just that.  

So what's the issue? Well, 'Docker for Windows' is pretty flaky; sometimes won't start for no apparent reason and installing it often fails. Chatting to colleagues at work and on Twitter several people have had problems with it, usually just before they are running a live demo - the last thing you want to be worrying about before a demo is "will Docker fall over?" 

Now I'm not sure what the root of these problems is, but I thought I'd examine some of the alternatives. These alternatives all forgo installing the regular 'Docker for Windows' setup on your machine. I will also be ignoring the need to run Windows Containers, as we've already got enough to discuss.

<!--more-->

# Docker Tools and CLI

The first challenge is getting the docker command line tools on your machine. [Docker provide a nice installer for just the tools called 'Docker Toolbox'](https://www.docker.com/products/docker-toolbox).

The tools were are interested in are the following commands: `docker`, `docker-compose` and `docker-machine`

**NOTE** the default install options with 'Docker Toolbox' include a lot of rubbish you don't need, like VirtualBox (blergh) and Kitematic, be sure to deselect those options and just install 'Docker Client', 'Docker Compose' and 'Docker Machine' 

### Docker client tools for WSL bash
If you are using Windows Subsystem for Linux (WSL) and bash you will want to use the Docker tools from there too. There is no client only package that Docker provide for Linux, however the binaries are available. 

Running the following snippet in a WSL bash terminal will download and install the tools you need.

<pre class="command-line language-bash" data-user="bob" data-host="host" data-output="6,11"><code>
curl -L https://download.docker.com/linux/static/stable/x86_64/docker-17.09.0-ce.tgz -o /tmp/docker.tgz
tar -zxvf /tmp/docker.tgz docker/docker 
chmod +x docker/docker 
sudo mv docker/docker /usr/local/bin/docker
rmdir docker/

curl -L https://github.com/docker/machine/releases/download/v0.13.0/docker-machine-`uname -s`-`uname -m` -o /tmp/docker-machine
chmod +x /tmp/docker-machine
sudo cp /tmp/docker-machine /usr/local/bin/docker-machine

curl -L https://github.com/docker/compose/releases/download/1.17.1/docker-compose-`uname -s`-`uname -m` -o /tmp/docker-compose
chmod +x /tmp/docker-compose
sudo cp /tmp/docker-machine /usr/local/bin/docker-compose
</code></pre>


# Running a Docker host
OK with the client tools installed, we need some machine or VM running the Docker engine (I'll refer to this as a Docker host)

Your choices here fall into two options; run a VM locally or run a VM in the cloud, we'll discus both here.

Your next decision is how to build the VM and install Docker, as we're in DIY mode, the temptation is to create the Linux VM yourself (e.g. Ubuntu) and then install Docker on it. **DO NOT DO THIS!** This is way, way more pain than it worth, you will spend an inordinate amount of time with configuration; iptables, daemon.json, etc and at the end of it all have an insecure Docker instance.

The 'Docker Machine' tool was created for a good reason, so we should use it. Docker Machine will create the Docker host VM for us, install the Docker engine securely *and* give us a clean simple way to point our tools to the new VM

## Running Docker Machine
Docker Machine will let us create our host either in the cloud or locally. This is done via different drivers, there are a [wide range of drivers](https://docs.docker.com/machine/drivers/) we will look at two, Azure and Hyper-V

#### Creating Docker host in Azure
This is a simple process, just a single command `docker-machine create -d azure`. You will need your Azure subscription ID (easily found in the Azure portal) and that's about it. The following is a bare bones but functioning example
<pre><code class="language-bash">
docker-machine create -d azure --azure-subscription-id 12345678-aaaa-bbbb-cccc-abcdef123456 myNewHost
</code></pre>

The last parameter `mynewhost` is the name of the host & VM, so you can pick anything you like, but uppercase and hyphens tend to cause problems dock

You will likely want a bit more control over how the VM is created, and you can do this with additional parameters provided to the `create` command. Some parameters worthy of attention are:
- `--azure-location` - Azure region/location
- `--azure-resource-group` - Name of the resource group
- `--azure-size` - VM size, e.g. Standard_A4_v2 or Standard_F8s_v2
- `--azure-static-public-ip` - Assign static public IP
- `--azure-open-port` - List of ports to open on the NSG that is created
- `--azure-dns` - DNS prefix for the public IP 

A [full list is available here](https://docs.docker.com/machine/drivers/azure/)

<pre><code class="language-bash">
docker-machine create -d azure --azure-subscription-id 12345678-aaaa-bbbb-cccc-abcdef123456 \
--azure-resource-group Temp.Docker --azure-size Standard_F4s_v2 --azure-static-public-ip \
--azure-dns bensdocker --azure-location westeurope --azure-open-port 3000-8080 bensdocker
</pre></code>

<pre><code class="language-bash">
docker-machine create -d hyperv --hyperv-cpu-count 2 --hyperv-memory 1024 --hyperv-disk-size 8000 localdocker
</pre></code>

### Sharing Docker Machine config
If you are going to switching between PowerShell and WSL as I often do, then you might want the docker CLI to work in both against your remote Docker Machine created host.

Docker Machine stores its config in the `.docker/` folder in your home or user profile directory


