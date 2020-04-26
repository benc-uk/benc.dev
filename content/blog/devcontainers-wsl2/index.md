---
title: VS Code Devcontainers with WSL2
date: 2020-04-26
tags: [linux, wsl, docker]
description: This is a super quick post on get VS Code Devcontainers (aka 'VS Code Remote Containers') working entirely with just WSL2, without the need to install Docker for Windows
---
This is a super quick post on get VS Code Devcontainers (aka 'VS Code Remote Containers') working entirely with just WSL2 ***without the need to install Docker for Windows***

I've wanted to give VS Code Devcontainers a go for a while, however I was put off by the need to run VS Code in Windows and the need for Docker for Windows. I've switched to a 100% WSL environment for all my dev work, so I have no dev tools (e.g. git) installed under Windows at all, I use VS Code WSL Remote for *everything*. 

Inspired by Stuart Leeks [post on using Devcontainers with WSL2](https://stuartleeks.com/posts/vscode-devcontainers-wsl/) which solves 80% of the problem, I wanted to see if I could remove the final annoying part - the requirement to have Docker for Windows. I'm pleased to say I've cracked it.


# Pre Reqs
- WSL2 up and running and working
- Docker installed in WSL2, daemon/engine + client. Use [this script if you're in a hurry](https://github.com/benc-uk/ubuntu-tools-install/blob/master/docker-engine.sh)
- Follow the steps outlined in [Stuart's post on Devcontainers and WSL2](https://stuartleeks.com/posts/vscode-devcontainers-wsl/), e.g. installing the VS Code Remote Extension Pack, and setting the `"remote.containers.experimentalWSL": true` flag in your VSCode settings. **IGNORE THE PARTS ABOUT DOCKER FOR WINDOWS 😁**

## Start/Stop Docker Aliases 
Docker in WSL2 will not auto start (as there's no init system in WSL), however starting it manually is no hardship. To this end, I suggest creating two aliases
```
start-docker='sudo /etc/init.d/docker start'
stop-docker='sudo /etc/init.d/docker stop'
```

# Part 1 - Docker over TCP
We'll switch the Docker daemon to listen on TCP, rather than just a local Unix socket

Edit the daemon config file
```bash
sudo nano /etc/docker/daemon.json
```

Paste in these contents and save 
```json
{
  "tls": false,
  "hosts": ["tcp://0.0.0.0:2375", "unix:///var/run/docker.sock"]
}
```
**Note.** It's important you leave the ability to connect via the socket (`/var/run/docker.sock`), the VS Code Remote containers extension does some strange things, including some pre-checks that launch `/bin/sh -c docker` in non-interactive mode, this won't read any profile scripts, effectively making it impossible to set env variables for. Just trust me on this one, it caught me out big time.  

Stop the daemon if it's running, and restart it.

If you run a docker command now, e.g. `docker ps` you should get an error. To tell the docker client to connect using TCP, we need to set the `DOCKER_HOST` env variable

```bash
export DOCKER_HOST=tcp://localhost:2375
docker ps
```
To make this change permanent place the export command in whatever shell profile script you use (`.bashrc` or `.zshrc` etc)

# Part 2 - Docker Windows client
Next get the Docker *client* installed into Windows. This is effectively all we need Windows side. Not the 400Mb beast that is Docker For Windows.

- Download the latest zip from here: https://download.docker.com/win/static/stable/x86_64/
- Extract `docker.exe` into a directory that is on your system path, or create a new directory and add it to your path.
- Create an new env variable called `DOCKER_HOST` and set the value to `tcp://localhost:2375`
- If you have any open PowerShell or Terminal windows you need to close them for the new env variable to take effect. Good 'old Windows eh?
- Open a PowerShell Core / PowerShell classic session (you could use CMD but you'll be very harshly judged) and run `docker ps`
- Huzzah!

We can stop there, and jump in a start using Devcontainers. Making sure you open the project folder in VS Code using the `\\$wsl\blah\blah` path, it should just work.

OK, this configuration isn't secure. Anyone could theoretically connect to your Docker host and do Bad Things(TM). I actually think this risk is extremely low, given that WSL2 isn't always running, and neither is Docker. But the risk is there

Lets look at a locking this down. Beware this part is not for the feint of heart! Skip it if you're happy with the less secure config currently in place

# Part 3 - TLS and client auth for Docker
Run these steps from within WSL2
- Create temporary directory to work out of.
- Carry out every step in this guide https://docs.docker.com/engine/security/https/ down to the steps where you chmod the files.  
**VERY IMPORTANT NOTE!** Anytime a command in the guide references `$HOST` use the value `localhost` or you'll get connection problems later.
- You might get an error about a missing `.rnd` file as I did. To create one simply run `openssl rand -out ~/.rnd`
- Copy the server pem files and CA somewhere, I put them in `/etc/docker`
  ```bash
  sudo cp ca.pem server-cert.pem server-key.pem /etc/docker
  ``` 
- Edit the daemon config file as before, e.g. `sudo nano /etc/docker/daemon.json`, new contents are:
  ```json
  {
    "tls": true,
    "tlscert": "/etc/docker/server-cert.pem",
    "tlskey": "/etc/docker/server-key.pem",
    "hosts": ["tcp://0.0.0.0:2376", "unix:///var/run/docker.sock"]
  }
  ```
- Copy the client certs and key to your WSL user `.docker` directory

  ```bash
  mkdir -pv ~/.docker
  cp -v {ca,cert,key}.pem ~/.docker
  ```
  Do the same to copy to the Windows side. NOTE. You will need to change the path to match your system/username
    ```bash
  mkdir -pv /mnt/c/Users/YOUR_USERNAME/.docker
  cp -v {ca,cert,key}.pem /mnt/c/Users/YOUR_USERNAME/.docker
  ```
- Enable TLS on the Docker client (as before, place in .bashrc/.zhrc or other profile script to make permanent)
  ```bash
  export DOCKER_HOST=tcp://localhost:2376 DOCKER_TLS_VERIFY=1
  ```
- Restart the Docker daemon and test with `docker ps`

Steps on Windows client side
- Modify the `DOCKER_HOST` env variable created in part 2, new value is `tcp://localhost:2376`
- Add a second env variable `DOCKER_TLS_VERIFY` value is `1`
- Close all terminals and restart, and test with `docker ps` command

Phew! 

# Summary
This hopefully will be of use to people that are working in a similar fashion (all in on WSL) and want to use the powerful new Devcontainers feature of VS Code