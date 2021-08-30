# nginx reverse proxy

- reverse proxy
- load balancing
- logging to persistent volume 
- ssl with letsencrypt/certbot

This is an example server configuration to see how the reverse proxy and load balancing works.

Docker Swarm comes with it's own load balancing between nodes.

https://docs.docker.com/engine/swarm/key-concepts/#load-balancing

## Setup

Requires a Docker host.

https://docs.docker.com/get-docker/

Servers are located in their own folder inside `/servers` to keep each config separate.

## Commands

### Single host deployment with docker-compose

```
# simple reverse proxy setup
docker-compose up --build
docker-compose down

# load balancer with multiple servers
docker-compose -f stack.yaml up --build
docker-compose -f stack.yaml down
```
Note: --build in the command will force the image to be rebuilt incase you made any changes.

### With docker swarm and multiple hosts.

The images will be pulled from Docker Hub instead of using a local Dockerfile.
```
# deploy stack
docker swarm init
docker stack deploy -c stack.yaml stackname

# list stack and services to see them running
docker service ls
docker stack ls
docker stack services stackname

# to update the stack you can run the same command again
docker stack deploy -c stack.yaml stackname

# remove stack
docker stack rm stackname

```
## index.html

Each server has it's own index.html that's copied when the image is built.

## Reverse Proxy

You don't need to expose any ports on the internal docker containers only the client facing Nginx server needs to expose port 80/443, allow Nginx to route the traffic internally.

Note that we use the upstream group name when specifying the proxy_pass for the reverse proxy location.

You can use any other domain, ip, upstream group or docker container name.

Also note the trailing slash on the location and proxy_pass.

This will rewrite the url parameters so it doesn't pass /servera with the request.

A request made to `http://localhost/backend/1` will translate to `/1`

## Load Balancing

Spin up the containers and visit `http://localhost/` in an incognito browser to test the load balancing between servers. It should round robin between servers and show whether you are on server A or server B.

You can access each server individually at their location urls
- http://localhost/servera/
- http://localhost/serverb/

When using Docker Swarm the load will be distributed across all nodes so you can view the same setup from multiple ip addresses within the swarm and Nginx will load balance between the internal Nginx servers to show you which server is responding.

You can use an external load balancer as well as the routing mesh provided by docker swarm.

https://docs.docker.com/engine/swarm/how-swarm-mode-works/services/#services-tasks-and-containers
https://docs.docker.com/engine/swarm/key-concepts/#load-balancing
https://docs.docker.com/engine/swarm/ingress/#using-the-routing-mesh
https://docs.docker.com/engine/swarm/ingress/#configure-an-external-load-balancer

## Logging

The logs from Nginx are being written to the /logs folder in the single host reverse proxy setup. This won't work in swarm mode and it's not advised to write to the host volume when using swarm mode.

The official nginx image creates a symbolic link from /var/log/nginx/access.log to /dev/stdout, and creates another symbolic link from /var/log/nginx/error.log to /dev/stderr, overwriting the log files and causing logs to be sent to the relevant special device instead. See the [Dockerfile](https://github.com/nginxinc/docker-nginx/blob/8921999083def7ba43a06fabd5f80e4406651353/mainline/jessie/Dockerfile#L21-L23).

To save the logs to a volume there is an additional log being created in the nginx.conf files.

Now you can `cd /var/log/nginx` and cat the log files to see the contents. The logs should now persist on the volume when containers are removed.

## SSL

It's a little tricky to get ssl working properly. You first need certbot to issue a certificate for the domain, then you need to add the certs to the Nginx config.

Edit the command section of the ssl.yaml to include your email and the domain you want to register.
```
command: certonly --webroot --webroot-path=/var/www/certbot --email your-email@domain.com --agree-tos --no-eff-email -d domain.com -d www.domain.com
```

Set an A record for the domain to point to your server IP address.

Then load nginx with ssl config on the server
```
docker-compose -f ssl.yaml up
```

Let certbot generate certs.

After it generates the certs and saves the files to the host machine you can uncomment the lines that reference the ssl certs in the ssl.conf and spin up the containers again.

```
# edit the ssl.conf to uncomment the ssl configs and add your domain name
nano servers/ssl/ssl.conf
```

spin up the container again and test ssl works!
```
docker-compose -f ssl.yaml up
```

- [nginx ssl](https://www.nginx.com/blog/using-free-ssltls-certificates-from-lets-encrypt-with-nginx/)

## Deploying to docker swarm

Requires an image from a docker registry. You can create one locally by following the guide in https://docs.docker.com/engine/swarm/stack-deploy/ or push your own images to Docker Hub.

You need to pull the images from a registry in the docker-compose file. So instead of building an image with a local Dockerfile like you would with docker-compose the image is hosted online.

The docker images for this repo are available here:
https://hub.docker.com/r/jdizm/nginx-reverse-proxy

### Pushing a new image the repository

```
docker push jdizm/nginx-reverse-proxy:tagname
```
Each server image is tagged with its own name. 

To build the images again we can run docker-compose and then push them to the repo to update their tags.

an example workflow for updating all the images, then deploying the stack again.
```
docker-compose -f stack.yaml up --build
docker push jdizm/nginx-reverse-proxy:loadbalancer
docker push jdizm/nginx-reverse-proxy:servera
docker push jdizm/nginx-reverse-proxy:serverb
docker stack deploy -c stack.yaml stackname
```

## deploying a docker host with docker machine to digitalocean

see - https://docs.docker.com/machine/drivers/digital-ocean/

we can export our token and fingerprint to a bash variable
```
export DOTOKEN=<your generated token>
export DOFINGER=<ssh key fingerprint>
```

then we can create a new droplet and connect to it
```
# create a new droplet with digitalocean
docker-machine create --driver digitalocean --digitalocean-access-token $DOTOKEN --digitalocean-ssh-key-fingerprint $DOFINGER --digitalocean-size s-1vcpu-1gb --digitalocean-image ubuntu-20-04-x64 --digitalocean-region lon1 droplet-name

# ssh into the machine when it's been provisioned
docker-machine ssh droplet-name
```

If you encounter an "Unable to query docker version: Cannot connect to the docker engine endpoint" error. Then you can try stopping and starting the machine again and running the recommended steps to verify the docker env.
```
# get the machine name and ip address
docker-machine ls

# stop
docker-machine stop machine-name

# start
docker-machine start machine-name

# recommended steps to verify docker env
docker-machine env machine-name 
eval $(docker-machine env machine-name)

```

## creating a new manager

after gaining a shell on the host you want to become a manager
```
# init a new swarm and deploy the stack from github repo
docker swarm init --advertise-addr <ipaddress>
cd home
git clone https://github.com/JDIZM/nginx-reverse-proxy.git
cd nginx-reverse-proxy
docker stack deploy -c stack.yaml nrp

# show the running stack services
docker service ls
docker stack services nrp

# show the running tasks and their state
docker stack ps nrp
```
## Firewall ports

Note: make sure the firewall isn't blocking any of the ports https://docs.docker.com/engine/swarm/swarm-tutorial/#open-protocols-and-ports-between-the-hosts

- 2376 TCP
- 2377 TCP
- 7946 TCP/UDP
- 4789 UDP

## adding a node to the swarm

```
# get the join token from the manager
`docker swarm join-token worker`

# ssh into the machine you want to add and use the join token
docker-machine ssh machine-name

# then use the join token you received from the manager to add the node to the swarm.
docker swarm join --token <TOKEN> <MANAGER-IP>

# check the tasks running on the stack
docker stack ps stackname

# filter tasks that are running
docker stack ps stackname -f desired-state=running
```

The worker should join the node and services can be distributed across workers.

To scale the service to more workers you can do the following.
```
# get the service name
docker service ls

# scale the service to 3 workers
docker service scale service-name=3
```
- https://docs.docker.com/engine/swarm/swarm-tutorial/add-nodes/
- https://docs.docker.com/engine/swarm/how-swarm-mode-works/services/
- https://docs.docker.com/engine/swarm/ingress/#using-the-routing-mesh

## Resources

- http://nginx.org/en/docs/beginners_guide.html#proxy
- https://docs.nginx.com/nginx/admin-guide/load-balancer/http-load-balancer/
- https://docs.nginx.com/nginx/admin-guide/security-controls/securing-http-traffic-upstream/
- https://www.digitalocean.com/community/tutorials/understanding-nginx-http-proxying-load-balancing-buffering-and-caching
- https://medium.com/@simonestaffa/deploy-docker-containers-with-zero-downtime-ed06b0a0966d
- https://www.docker.com/blog/how-to-use-the-official-nginx-docker-image/
- https://docs.docker.com/docker-hub/repos/
- https://docs.docker.com/config/containers/logging/
- https://docs.docker.com/machine/
- https://docs.docker.com/machine/drivers/digital-ocean/
- https://docs.docker.com/engine/swarm/stack-deploy/
- https://docs.docker.com/engine/swarm/swarm-tutorial/add-nodes/
- https://docs.docker.com/engine/swarm/how-swarm-mode-works/services/
- https://docs.docker.com/engine/swarm/ingress/#using-the-routing-mesh
- https://docs.docker.com/engine/swarm/key-concepts/#load-balancing

