---
layout: page
title: Production Setup
permalink: /production_setup/
---

In this setup we use the same ERPNext image as we use in trial setup
and config it to run production
and instead of running all service in single container we separate some and put it into 8 container,
and most important thing is it separate data volumes from container to docker volumes.

1. frappe
2. mariadb
3. nginx-frappe
4. redis cache
5. redis queue
6. redis socketio
7. nginx-proxy
8. nginx-letsencrypt

Note that every time you want to update app or install new app on frappe image
you have to create new frappe image and that image must have blank bench/logs and bench/sites folder.

Because if unfortunate unknown event that might cause frappe service to go down
docker swarm will try to maintain that service by create new container using image that define in compose file.
If you update app or install new app and don't create new image as soon as service fail
your site will fail as well.

### Usage

* Init swarm

    `docker swarm init`

* Clone repository

    `git clone https://github.com/pipech/erpnext-docker-debian.git`

* Change work directory

    `cd erpnext-docker-debian/production_setup`

* Change Environment in production_setup/env/ file

    ```
    - frappe_app.env
    - frappe_nginx.env
    - nginx_proxy.env
    ```

    note: Now you can't access website through localhost, 
    you should've set domain point to machine ip address
    and access through that domain.

* Deploy stack using prd.yml as prd1 stack (In production folder where prd.yml is)

    `docker stack deploy -c prd.yml <stack_name>`

* Find frappe container id

    `docker ps -a`

* Call bash in frappe container

    `docker exec -it <frappe_container_id> bash`

* Run init.sh

    `cd .. && . init.sh`

* Exit from container

    `exit`

* Config mysql

    `docker exec -it <mysql_container_id> bash`

    `mysql -u "root" "-p123" < "/home/init.sql"`

* Exit from container

    `exit`

* Restart frappe container

    `docker service update --force <stack_name>_frappe`

* Remove all exited container

    `docker rm $(docker ps -a -q)`

* Go to web browser and access ERPNext via your domain

    `http://yourdomain.com`

### Finishing up

* Change administrator password by login in to http://yourdomain.com

    ```
    ERPNext
    User : Administrator
    Pass : admin
    ```

* Change MySQL password

    `docker exec -it <mysql_container_id> mysql -u "root" "-p123"`
    
    `ALTER USER 'root'@'localhost' IDENTIFIED BY 'MyNewPass';`

### Health-check

* All 8 services should running.

    `docker service ls`
    ```
        ID                  NAME                    MODE                REPLICAS            IMAGE                                            PORTS
    ywe1xryvsfun        p06_frappe              replicated          1/1                 pipech/erpnext-docker-debian-production:stable   *:6787->6787/tcp,*:8000->8000/tcp,*:9000->9000/tcp
    v93gdseu5agy        p06_mariadb             replicated          1/1                 mariadb:10.2.12                                  *:3307->3306/tcp
    8wnwrnijgash        p06_nginx-frappe        replicated          1/1                 nginx:1.12.2                                     *:8080->80/tcp
    4dzi4g0x4d9x        p06_nginx-letsencrypt   replicated          1/1                 jrcs/letsencrypt-nginx-proxy-companion:latest
    ops1qx3f3cxs        p06_nginx-proxy         replicated          1/1                 jwilder/nginx-proxy:latest                       *:80->80/tcp,*:443->443/tcp
    x6er9y432gjt        p06_redis-cache         replicated          1/1                 redis:alpine
    2r1zu913egm7        p06_redis-queue         replicated          1/1                 redis:alpine
    zjfdf8i64j42        p06_redis-socketio      replicated          1/1                 redis:alpine
    ```

* Check service in frappe container, all 6 services should run with success

    `docker logs <frappe_container_id>`

    ```
    2018-02-09 07:10:53,185 CRIT Supervisor running as root (no user in config file)
    2018-02-09 07:10:53,192 INFO supervisord started with pid 5
    2018-02-09 07:10:54,196 INFO spawned: 'bench-frappe-schedule' with pid 8
    2018-02-09 07:10:54,197 INFO spawned: 'bench-frappe-default-worker-0' with pid 9
    2018-02-09 07:10:54,198 INFO spawned: 'bench-frappe-long-worker-0' with pid 10
    2018-02-09 07:10:54,200 INFO spawned: 'bench-frappe-short-worker-0' with pid 11
    2018-02-09 07:10:54,207 INFO spawned: 'bench-node-socketio' with pid 12
    2018-02-09 07:10:54,217 INFO spawned: 'bench-frappe-web' with pid 13
    2018-02-09 07:10:55,232 INFO success: bench-frappe-schedule entered RUNNING state, process has stayed up for > than 1 seconds (startsecs)
    2018-02-09 07:10:55,232 INFO success: bench-frappe-default-worker-0 entered RUNNING state, process has stayed up for > than 1 seconds (startsecs)
    2018-02-09 07:10:55,232 INFO success: bench-frappe-long-worker-0 entered RUNNING state, process has stayed up for > than 1 seconds (startsecs)
    2018-02-09 07:10:55,232 INFO success: bench-frappe-short-worker-0 entered RUNNING state, process has stayed up for > than 1 seconds (startsecs)
    2018-02-09 07:10:55,233 INFO success: bench-node-socketio entered RUNNING state, process has stayed up for > than 1 seconds (startsecs)
    2018-02-09 07:10:55,233 INFO success: bench-frappe-web entered RUNNING state, process has stayed up for > than 1 seconds (startsecs)
    ```

### Update image

You can update any service image simply by run

    docker service update --image <image>:<tag> <stack_name>_<service>

Or edit prd.yml file and run

    docker stack deploy -c prd.yml <stack_name>
    
### Adding new site

* Call bash in frappe container

    `docker exec -it <frappe_container_id> bash`

* Create new site

    `bench new-site <site_domain>`
    
* Install app

    `  bench --site <site_domain> install-app <app_name>`
    
* Add domain to production_setup/env

    ```
    - env/nginx_proxy.env
    - env/frappe_nginx.env
    ```