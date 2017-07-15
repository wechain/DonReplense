# DonReplense

#### Run as many dockerized web projects as you want, all on https, on a single server, all neatly packed in their own compose stack

## What exactly does this do?

DonReplense acts as a single reverse proxy with SSL offloading for any number of dockerized web projects on a single server, without  you having to worry to much about obtaining and maintaining SSL certificates, while still having a nice and tidy layer of abstraction between the different projects.

Any other Docker Web Container or Docker-Compose stack with a few environment variables, that lands in the same bridge network, will automatically be provided with a SSL certificate from LetsEncrypt.

The reverse proxy will be the only container that actually needs to expose any ports to the Docker host, all any other container needs to do is expose a port within the Docker hosts internal bridge network.

## How does this work?

DonReplense is nothing new actually, just a fancy name for an implementation of other projects that have been around for a while now.

**Do**cker **n**ginx **Re**verse **p**roxy **l**et's **en**crypt **se**parated would simply have been too long. SonReplense is easy to remember.

It utilizes too awesome projects: 
* [jwilder's docker-gen](https://github.com/jwilder/docker-gen)
and
* [JrCs's docker-letsencrypt-nginx-proxy-companion](https://github.com/JrCs/docker-letsencrypt-nginx-proxy-companion)

Let's go through everything step by step. The repository consists of two docker-compose stacks:
* the main stack
    * this stack contains the general setup. It consists of 4 services:
        * proxy
            * base image is just a regular nginx
            * the only container that actually exposes anything to the outside world, both port 80 and 443
            * connects to both this stacks network, as well as an external network called "ambassador"
            * gains read-only access to ./data/proxy/certs
        * docker-gen
            * the base image is [jwilder's docker-gen](https://github.com/jwilder/docker-gen)
            * has read-only access to the docker host's docker.sock
                * this is how the docker host will notify this container about other containers starting or stopping
            * will watch for any new container with an environment variable called "VIRTUAL_HOST"
                * if any such starting container is found, it will update it's nginx.tmpl file, create a new server directive for this new container and push the changes to the proxy container
                * similarly, if any such container is stopped, it will remove the corresponding entry from the server directives and push the updated config to the proxy container
        * letsencrypt-companion
            * will handle obtaining and renewing certificates from letsEncrypt
            * also has read-only access to the docker host's docker.sock
                * just like the docker-gen container, this container will be notified about any starting or stopping containers
            * will watch for any new container with environment properties called "LETSENCRYPT_HOST" and "LETSENCRYPT_EMAIL"
                * will check if there are certificate files for the domain name set in the "LETSENCRYPT_HOST" property of any such started container
                    * if none are found, it will try to obtain new certificate files for this domain, also using the "LETSENCRYPT_EMAIL" address
                    * if certificate files for this domain already exists, it will try to renew any certificate that would expire within 30 days every 3600 seconds
        * sample-app1
            * a simple hello-world app
            * sits in the same network as the main stack
            * has the 3 environment variables "VIRTUAL_HOST", "LETSENCRYPT_HOST" and "LETSENCRYPT_EMAIL" set via an .env file
            
* the sample2 stack:
    * this stack consists of only one container, app1
    * the app1 container sits in its own default, "private" network, as well as the external "ambassador" network
    * also has the 3 environment variables "VIRTUAL_HOST", "LETSENCRYPT_HOST" and "LETSENCRYPT_EMAIL" set via an .env file
    
## FAQ
Q: Why do the main stacks proxy and docker-gen container have their names set explicitly?

A: The letsencrypt-companion service needs to be aware of these exact containers. There are two ways to accomplish that:
First, take a look at the environment variables of the letsencrypt-companion service:<br>
The container names are passed to this service as NGINX_DOCKER_GEN_CONTAINER and NGINX_PROXY_CONTAINER. <br>
Usually, in any docker-compose setting, docker-compose generates dynamic names for any service in a given stack by the following scheme:
* $PWD or the name of the current folder the docker-compose.yml is residing
* an underscore _
* the service name
* another underscore _
* an integer representing the services instance number

If $PWD where to "dockerIsAwesome", the name for the proxy service generated would be "dockerisawesome_proxy_1"<br>
Unfortunately, referencing another service via a DNS entry is not enough in this case, we need to know the exact names of these two containers.

If you, for any reason, wouldn't like to set the names of these two containers explicitly, there is an alternative way: Setting labels. Just uncomment the label directives for the proxy and docker-gen service. In that case, you can comment-out the environment directives for the letsencrypt-companion service. 
            
                
