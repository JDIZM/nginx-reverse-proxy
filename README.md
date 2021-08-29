# nginx reverse proxy

- reverse proxy
- load balancing
- logging

## setup

Requires a Docker host.

https://docs.docker.com/get-docker/

Servers are located in their own folder inside `/servers` to keep each config separate.

## commands


```
# simple reverse proxy setup
docker-compose up --build
docker-compose down

# load balancer with multiple servers
docker-compose -f stack.loadbalancer.yaml up --build
docker-compose -f stack.loadbalancer.yaml down
```

Note: --build in the command will force the image to be rebuilt incase you made any changes.

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

Spin up the containers and visit `http://localhost/` in an icognito browser to test the load balancing between servers. It should round robin between servers and show whether you are on server A or server B.

You can access each server individually at their location urls
- http://localhost/servera/
- http://localhost/serverb/

## Logging

The logs from Nginx are being written to the /logs folder.

The official nginx image creates a symbolic link from /var/log/nginx/access.log to /dev/stdout, and creates another symbolic link from /var/log/nginx/error.log to /dev/stderr, overwriting the log files and causing logs to be sent to the relevant special device instead. See the [Dockerfile](https://github.com/nginxinc/docker-nginx/blob/8921999083def7ba43a06fabd5f80e4406651353/mainline/jessie/Dockerfile#L21-L23).

## Resources

- https://www.docker.com/blog/how-to-use-the-official-nginx-docker-image/
- https://docs.docker.com/config/containers/logging/
- http://nginx.org/en/docs/beginners_guide.html#proxy
- https://docs.nginx.com/nginx/admin-guide/load-balancer/http-load-balancer/
- https://www.digitalocean.com/community/tutorials/understanding-nginx-http-proxying-load-balancing-buffering-and-caching
- https://docs.nginx.com/nginx/admin-guide/security-controls/securing-http-traffic-upstream/




