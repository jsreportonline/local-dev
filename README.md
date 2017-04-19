# jsreportonline development

##Start local instance
1. download an install [docker](https://www.docker.com/community-edition#/download)
2. clone this repository
3. run `./clone-all.bat` (it should run on all platforms)
4. add the following to `/etc/hosts`
```sh
127.0.0.1 local.net 
127.0.0.1 test.local.net
```
5. run `docker-compose up` (can take ~10 minutes based on the network)
6. open [http://local.net](http://local.net) in browser and register tenant with subdomain `test`

##Technical notes

### Composition
jsreportonline service is split into several micro services defined in [docker-compose.yml](https://github.com/jsreportonline/local-dev/blob/master/docker-compose.yml) file. This mainly assures that the user code evaluation is properly isolated in dedicated container but also simplifies the deployment. The local `docker-compose.yml` is different from the production one. It includes less containers for user code evaluation and doesn't heave restart policy enabled. The production compose definition has for example 8 `tasks` and 6 `phantom-pdf` 
containers to minimize requirement for recycling explained in the next chapter. 

The users request flow is the following

1. `nginx` container working as the reverse proxy
2. `main` container which hosts the whole jsreport front-end and the main logic, only the user code evaluation is extracted and forwarded to the `workers` container
3. `workers` container is the manager of workers evaluating the user code
4. `tasks` the worker container evaluating templating engines and scripts
5. `phantom-pdf` the worker container running phantom html to pdf conversion...

### Main
This container contains the full `jsreport` and on top of it adds additional logic like: quota throttling, billing, multi-tenancy or custom authentication implementation. 

As mentioned previously, it doesn't run the user code like phantomjs based conversion or templating engines evaluation. This is extracted out from the jsreport in a little bit hack form. It for example changes the `script-manager` execution function to serialize the request and send it to the `workers`. In the similar way it modifies the `phantom-pdf`, `electron-pdf` and other recipes.

### Workers

The most interesting container is `workers` which is the hearth of the service. It is responsible for managing all containers used for user code evaluation and assures that the containers are properly recycled and that the each tenant runs in dedicated one.

####Dedicated containers
To assure that the each tenant runs in its own dedicated container we run a pool of the containers of each type and assign the tenants to them. It can never happen that one container has multiple tenants assigned. It can also never happen that a tenant leave a "bomb" in the container which is then reassigned to another tenant because we always recycle container before assigning a new tenant.

The workflow of the `workers` container is the following

1. Determine the container type (`tasks`, `phantom-pdf` ... ) and tenant from the request
2. Check the current pool of particular type and determine if the container is already assigned to the particular tenant. Forward the request to the container if found.
3. If there is not an already assigned container take the LRU unassigned one, assign it to the particular tenant and forward the request
4. If there is not a unassigned container, find the LRU assigned container with no running requests, reassign the tenant, restart the container and forward the request
5. If there is even no container which would not be running a request, queue the incoming request, the queue is flushed after each request finishes.

####Cluster

The production environment runs in the cluster of multiple servers and it is important to avoid assigning tenants to workers on the two servers at once. To overcome this we store in database the information to which server IP is the particular tenant currently assigned. This information is used at the beginning of the `workers` container processing to determine if the execution should continue or should be just resent to different server.

The `workers` container also periodically pings the database to make sure that others container doesn't resend the request to it when it is already turned off.

####Environments

The `workers` is correctly behaving also when multiple environments target the same database. Lets say we have production and test environments both using the same database. We don't want that the request submitted originally to the production environment is running on the `workers` container which is part of the test.

To overcome this the `workers` IP assignment information includes also the name of the environment which is stored as `stack`.

#### Windows workers

jsreportonline is able to run the `phantom-pdf` and `wkhtmltopdf` recipe also on the windows servers. Mainly because of back compatibility reasons.  This is the separate service which is running outside of the docker on separated machines. The `workers` container recognizes the request with windows target and forwards it to the LRU registered windows service.

### Networks
The `docker-compose.yml` defines dedicated network for every container evaluating user code. This brings networking isolation to these containers, because it should not be possible to ping from one worker container into the another. Only the `workers` container is listed in all these networks so it can distribute the tasks. 

### Mongo
The local instance uses mongodb running as yet another docker container. This is just convenient for the development, but the real db runs on the dedicated servers in 3-node replica set hosted in Mongo Atlas.



