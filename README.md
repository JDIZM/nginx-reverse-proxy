# nginx reverse proxy

- reverse proxy
- load balancing
- logging

## setup

Requires a Docker host.

https://docs.docker.com/get-docker/

Servers are located in their own folder inside `/servers` to keep each config separate.

## commands

### Using docker-compose on a single host.

Use `docker-compose` for local deployment.
```
# simple reverse proxy setup
docker-compose up --build
docker-compose down

# load balancer with multiple servers
docker-compose -f stack.loadbalance.yaml up --build
docker-compose -f stack.loadbalance.yaml down
```
Note: --build in the command will force the image to be rebuilt incase you made any changes.
### With docker swarm and multiple hosts.

Use `docker stack deploy` for deployment to docker swarm.
```
# deploy stack
docker swarm init
docker stack deploy -c stack.loadbalance.yaml stackname

# list stack and services to see them running
docker service ls
docker stack ls
docker stack services stackname

# to update the stack you can run the same command again
docker stack deploy -c stack.loadbalance.yaml stackname

# remove stack
docker stack rm stackname

```
## index.html

Each server has it's own index.html that's copied when the image is built.

## Reverse Proxy

You don't need to expose any ports on the docker containers, allow Nginx to route the traffic.

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

## Logging

The logs from Nginx are being written to the /logs folder.

The official nginx image creates a symbolic link from /var/log/nginx/access.log to /dev/stdout, and creates another symbolic link from /var/log/nginx/error.log to /dev/stderr, overwriting the log files and causing logs to be sent to the relevant special device instead. See the [Dockerfile](https://github.com/nginxinc/docker-nginx/blob/8921999083def7ba43a06fabd5f80e4406651353/mainline/jessie/Dockerfile#L21-L23).

## Deploying to docker swarm

Requires an image from a docker registry. You can create one locally by following the guide in https://docs.docker.com/engine/swarm/stack-deploy/ or push your own images to Docker Hub.

You need to pull the images from a registry in the docker-compose file. So instead of building an image with a local Dockerfile like you would with docker-compose the image is hosted online.

The docker images for this repo are available here:
https://hub.docker.com/r/jdizm/nginx-reverse-proxy

### Pushing a new image the repository

`docker push jdizm/nginx-reverse-proxy:tagname`

Each server image is tagged with its own name. 

To build the images again we can run docker-compose and then push them to the repo to update their tags.

an example workflow for updating all the images, then deploying the stack again.
```
docker-compose -f stack.loadbalance.yaml up --build
docker push jdizm/nginx-reverse-proxy:loadbalancer
docker push jdizm/nginx-reverse-proxy:servera
docker push jdizm/nginx-reverse-proxy:serverb
docker stack deploy -c stack.loadbalance.yaml stackname
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
docker-machine create --driver digitalocean --digitalocean-access-token $DOTOKEN --digitalocean-ssh-key-fingerprint $DOFINGER --digitalocean-size s-1vcpu-1gb --digitalocean-image ubuntu-18-04-x64 droplet-name

# when it's been provisioned
# ssh into the machine
docker-machine ssh droplet-name
```

If you encounter an "Unable to query docker version: Cannot connect to the docker engine endpoint" error. Then you can try stopping and starting the machine again.
```
# get the machine name and ip address
docker-machine ls

# stop
docker-machine stop machine-name

# start
docker-machine start machine-name

```


## creating a new manager
```
docker swarm init --advertise-addr <ipaddress>
cd home
git clone https://github.com/JDIZM/nginx-reverse-proxy.git
cd nginx-reverse-proxy
docker stack deploy -c stack.loadbalance.yaml nrp
# show the running stack services
docker service ls
```

make sure the firewall isn't blocking ports https://docs.docker.com/engine/swarm/swarm-tutorial/#open-protocols-and-ports-between-the-hosts

## adding an node to the swarm

get the join token from the manager.
`docker swarm join-token worker`

ssh into the machine you want to add and
`docker swarm init`

then use the join token you received from the manager to add the node tot he swarm.

- https://docs.docker.com/engine/swarm/swarm-tutorial/add-nodes/
## Resources

- https://www.docker.com/blog/how-to-use-the-official-nginx-docker-image/
- https://docs.docker.com/config/containers/logging/
- http://nginx.org/en/docs/beginners_guide.html#proxy
- https://docs.nginx.com/nginx/admin-guide/load-balancer/http-load-balancer/
- https://www.digitalocean.com/community/tutorials/understanding-nginx-http-proxying-load-balancing-buffering-and-caching
- https://docs.nginx.com/nginx/admin-guide/security-controls/securing-http-traffic-upstream/
- https://medium.com/@simonestaffa/deploy-docker-containers-with-zero-downtime-ed06b0a0966d
- https://docs.docker.com/engine/swarm/stack-deploy/
- https://docs.docker.com/engine/swarm/swarm-tutorial/add-nodes/
- https://docs.docker.com/machine/
- https://docs.docker.com/machine/drivers/digital-ocean/


