# docker deployer

This is some tools for deploying docker containers to a host which is
providing access to the docker with nginx.

I wanted something similar to heroku, rapid deployment of new things,
but without being bound to heroku and having to pay for an account to
scale and all those things.


## changelog

*2014 September* - make the deployment start a daemon on the remote
host to continually ping the docker's HTTP and restart it. The daemon
has a fifo that it reads for commands, you can get a list of commands
by sending it "help". Everything is stored in `/tmp/ddctrl/{name-of-image}`

*2014 August* - handle nginx configs with upstreams instead of direct
HTTP addresses in backend statements. The support isn't great but
without parsing the nginx config totally (and the best placed thing to
do that would be nginx) there's not a lot we can do better.


### requirements

You need:

* `bash`
* `curl`
* `jq`
* `docker` - duh
* an app that has a `Dockerfile`
* something that exports a port with Docker, probably a webapp because this uses nginx
* a live host to host your container
* an ssh key relationship with your remote host such that you can `ssh remotehost`
* an account on the docker registry

### how to

```shell-session
$ bash <(curl -L https://raw.github.com/nicferrier/docker-shell-deploy/master/deploy-make)
```

will download the script and ask a few questions:

```
docker image: nicferrier/elnode-gnudoc
nginx config: /etc/nginx/sites-enabled/gnudoc.conf
remote host name: po5.ferrier.me.uk
```

* `docker image` - the name of the docker image you want to build, this will be pushed to the docker registry
* `nginx config` - the nginx config on the live host which proxies the docker
* `remote host name` - the host name of the live host for the docker, the docker will be pulled here from the docker registry

If you have VOLUME export statements in your `Dockerfile` it will also
ask you where thet are mapped.

The script then creates a minimal `deploy` script which you can run to
deploy your docker app to the host you specified:

```ShellSession
$ bash deploy
```

You can safely check that into version control.

### what does the deploy script do?

Here's an example:

```bash
#!/bin/bash
# Docker deploy script generated by deploy-make

[ -f ./.deploy-test ] && source ./.deploy-test
[ -f ./.deploy ]     || curl https://raw.githubusercontent.com/nicferrier/docker-shell-deploy/master/deploy-helpers -o ./.deploy     || { echo "can't http the deployscript" ; exit 1; }
. ./.deploy
dockerImage=nicferrier/elnode-linky
dockerExPort=8005
nginxConfig=/etc/nginx/sites-enabled/linky
hostName=po5.ferrier.me.uk
dockerVolumes=;/home/nferrier/linky/db:/home/emacs/elnode-linky/db
deploy ${1:-"deploy"} nicferrier/elnode-linky 8005 /etc/nginx/sites-enabled/linky.conf po5.ferrier.me.uk /home/nferrier/linky/db:/home/emacs/elnode-linky/db
```

So what does it do?

* builds the current Dockerfile to `docker image`
* pushes the `docker image` to the [docker registry](https://registry.hub.docker.com/)
* pushes a function to the `remote host name` with ssh and executes it to:
** pull the new `docker image` from the docker registry
** start the pulled `docker image` as a container
** alter the `nginx config` to proxy the newly exported port
** restart nginx

### other stuff the deploy script supports

The `deploy` script that gets created actually supports a few more
tricks than just the deployment.

You can run just the build step of the deployment:

```ShellSession
$ bash deploy build
```

This will not push to the docker registry.

You can also run the build and push to docker registry, without doing
the deploy:

```ShellSession
$ bash deploy push
```

### stuff the deploy doesn't take care of

* creating your live environment outside the docker
** no nginx setup
** no creation of volume exported directories

## thoughts

This is very imperfect. It has a lot of assumptions:

* you're using a webapp
* you're exporting one port
* you're proxying with nginx
* you're using the docker registry
* ... and that's just for starters

However, it's a start and it is repeatable.

### is the download safe?

Perhaps you've heard that doing:

```ShellSession
$ curl http://something | bash -
```

is unsafe. Is:

```ShellSession
bash <(curl http://something)
```

any safer?

NO! Never use a curl and a bash together if you don't know what
they're doing. It's completely mad.

However, once you are confident of the script then you can do it.

If you're not confident of what the script does
then
[go look at it](https://github.com/nicferrier/docker-shell-deploy/blob/master/deploy-make).

### justification

Why not just use Heroku? or some other sexy PAAS?

Because this is just as usable and probably more scalable. By that I
mean that I can persuade it do new things easier than I can get Heroku
to do new things.

With Heroku I'm working with someone else's constraints. With this I'm
working with mine.
