---
title: Node.js with App Service on Linux
tags: [azure, nodejs, docker, linux]
image:
  feature: header/h02.svg
---
This is a follow up to my [first post](/nodejs-in-azure-pt1) on running Node.js in Azure App Service, this time I want to cover [the newly GA'ed App Service on Linux](https://azure.microsoft.com/en-us/blog/general-availability-of-app-service-on-linux-and-web-app-for-containers/). 
In most regards *App Service on Linux* works the same way as the regular (Windows) *App Service*, is has the same core features and operate mostly the same.  

The main difference (aside from the blindingly obvious; the host OS is Linux!) is *App Service on Linux* uses Docker containers to host & run your webapp. This provides a heap of advantages when it comes to deploying your app, and IMO containers also make an ideal way to run Node based apps. There's [many lightweight and robust Docker images](https://hub.docker.com/_/node/) you can use, the Alpine Linux image notably weighing in at a tiny 23MB.  

You have two fundamental ways of working with *App Service on Linux*:
- Use one of the provided application stack images  
Referred to as **"App Service on Linux"** or **"Linux Web Apps"**
- Run a custom image, either one of your own Docker images or one pulled from Dockerhub.  
Referred to as [**"Web App for Containers"**](https://azure.microsoft.com/en-us/services/app-service/containers/)

We'll cover using both approaches to host and run Node apps

<!--more-->

# App Service on Linux - Web Apps
Microsoft provides a range of app stacks for several languages; Node.js, PHP, .NET Core & Ruby. A full list of the versions is [detailed in the documentation](https://docs.microsoft.com/en-gb/azure/app-service-web/app-service-linux-intro), and range of Node versions is supported from v4 through to v8.

When using these provided stacks, a Linux Web App acts very similar to its Windows brethren. They host the runtime (thankfully without lumpy old IIS or messy workarounds like iisnode) and you just provide the code. You can deploy your code via Kudu or FTP and the container is configured to run this via a mapped volume. So that the directory and all the files in `/home/site/wwwroot` on the Web App gets mounted into the container (into the same path `/home/site/wwwroot`)

Kudu deployment options are limited to local Git, Github & Bitbucket (so no OneDrive or Dropbox, but those are pretty crappy deployment mechanisms anyhow). When deploying through Git, Kudu once again steps in and does all the nice things things like run `npm install` your behalf. Fab.

Another [neat feature is SSH access](https://docs.microsoft.com/en-us/azure/app-service/containers/app-service-linux-ssh-support), which gives you a interactive web shell into the running container, this lets you poke about to fix (and break!) things 

{% include img src="linux-web-app.png" alt="Illustration of how Linux Web Apps works with Node" caption=true %}

### Starting your Node app in App Service on Linux
The provided containers for Node do some clever stuff, in order to figure out how to start your app. The first choice will using the `npm start` script inside your `package.json` assuming it is provided. The next fallback is a "guess", so if your startup file is named: `bin/www`, `server.js`, `app.js`, `index.js` or `hostingstart.js` you don't need to do anything, the container will find it, and start Node running it.
The source & Dockerfiles are all on Github https://github.com/Azure-App-Service/node so you can poke about and take a look what is going on

Should this auto detection fail or you simply want to just tell the Web App how to start your Node app you can do so, via the portal

{% include img src="builtin-startup.png" %}

This "Startup File" can be the JavaScript file which starts your app (e.g. something like `server.js` or `index.js`) or more interestingly it can be a PM2 JSON file (typically called `process.json`), which can describe how to run your app and several other things such as watching for file changes and restart the Node process. More info on the startup file & PM2 is [documented here](https://docs.microsoft.com/en-gb/azure/app-service-web/app-service-linux-using-nodejs-pm2)

**Note.** If you are deploying via FTP rather than Kudu and don't upload the node_modules directory your app will obviously not start (just like with Windows App Service) however the container will also not start, meaning you will not be able to SSH into the container. Seeing HTTP 503 errors when accessing your site is a clue the container didn't start, and missing node_modules is a common cause for a Node app

<a name="wafc"></a>  

# Web App for Containers - Intro
With the GA launch of *App Service on Linux* they've tried to make a clearer distinction between running a custom image container and using the provided stacks, but this has also introduced some confusion.  
The new name for this mode of working is called *Web App for Containers* and it has been listed in the marketplace and product pages as new separate service, however this is a little misleading, the technical difference is minimal - it's still the same App Service on Linux service.  
So to be clear, both "App Service on Linux" and "Web App for Containers" are the same thing, just used slightly differently 

Phew, with that out of the way, *Web App for Containers* is where this service gets really interesting. You can bring **ANY** containerized web app and run it in Azure, and get all the nice supporting rich PaaS features that App Services brings.  
Want to run a Python Flask app? no problem (and here's a [little sample demo app](https://hub.docker.com/r/bencuk/python-demoapp/) to try if you want), want to use Java with WildFly or Tomcat? No problem. Want to run your own weird custom webserver or exe? No problem!

Cool eh? There is one caveat - the traffic to and from your container must be HTTP(S), so it much be a web app of some sort. No running something like MySQL or Redis, even if you fudge the service to listen on port 80 it still will not work, I've tried!

### Web App for Containers with Node
OK back to Node. Running a your app in custom Node.js container is simple, use one of the [standard official Node runtime images](https://hub.docker.com/_/node/) as a base and create a Dockerfile which bundles your app code on top, have `npm install` in there, and set the command/entrypoint to `npm start`

{% include img src="web-app-containers.png" alt="Illustration of how Web Apps for Containers works with Node" caption=true %}

A simple, minimal working Dockerfile for a Node Express app would look as follows

```dockerfile
FROM node:6-alpine
WORKDIR /usr/src/app

COPY package.json .
RUN npm install --production --silent

COPY . .

EXPOSE 3000
CMD [ "npm", "start" ]
```
Pretty simple!  
You can then push your image to either Dockerhub, Azure Container Registry or your own private registry then you simply point your Web App at it. You set this either during creation or you can specify it later from the portal & the "Docker Container" blade

{% include img src="wafc-image.png" alt="Selecting a custom image in Web App for Containers" caption=true %}

**Note.** The App Service will do a pretty good job of inspecting your image and auto detecting which port to use, the public web address provided will always be running on 80/443, the port mapping to the container is handled internally. Should you need specify the port you can do so with the `WEBSITES_PORT` application setting.

You don't have to use the portal, you can also deploy the App Service running your custom container using an ARM template of course. [This is an example template of mine on GitHub](https://github.com/benc-uk/azure-arm/tree/master/paas-web/linux-acr-existing) for doing just that. Note the use of various app settings all beginning with `DOCKER_` which control what image is used and how it is fetched

### Adding SSH to your Custom Container
If you want to add SSH support to your own container like the built in images have, it is pretty simple, but the specifics will vary depending on your base image and the distro of Linux it uses. The summary is:
- Install OpenSSH `apk add openssh` or `apt install -y openssh`
- Generate server keys `ssh-keygen -A`
- Set the root password to `Docker!` using `chpasswd`
- Configure OpenSSH to allow login from root and run on port 2222
- Start sshd as you start your container, via a shell script which then also starts your Node app

### Sample App
A working example and simple [reference containerized Node app can be found on my GitHub](https://github.com/benc-uk/nodejs-demoapp). This is based on the standard Alpine Node image, and runs the Express web framework. SSH support was added as described above.  
This app is also on Dockerhub as: [bencuk/nodejs-demoapp](https://hub.docker.com/r/bencuk/nodejs-demoapp/)

### Gotchas & Troubleshooting
Let's wrap up with a few gotchas. 
- The container is not started until the first HTTP request is made to the site, this can result in quite a delay on first load. Be patient! 
- If your container doesn't start you will see HTTP 503 errors. The logs in `/home/LogFiles` (check via FTP or Kudu) which reference docker should provide some clues. Note. you might need to [turn on logging](https://docs.microsoft.com/en-us/azure/app-service/containers/app-service-linux-intro#limitations)
- Your container will not automatically refresh/update you push an updated image to the registry.  
Your options are to simply restart the Web App, or add/change a dummy app setting which forces a pull & restart. You can also enable CI via webhooks, this can be switched on in the portal (see above screenshot) or from the Azure CLI ([more info](https://docs.microsoft.com/en-us/azure/app-service/containers/app-service-linux-ci-cd))
- If deploying with ARM you can point to a target image that doesn't exist (yet) this introduces some CD scenarios where both the image, registry and App Service are are created dynamically via automation 
- It's also well worth checkout the FAQ  https://docs.microsoft.com/en-us/azure/app-service/containers/app-service-linux-faq 

Hope this overview of both the new Linux App Service and using it with Node has been useful