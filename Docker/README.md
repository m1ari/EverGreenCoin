# Docker Image creation for EverGreenCoin

This will build a container based on Debian 10 (Buster) that will run the EverGreenCoin daemon using the current git release (v1.9.1.0). This is built against Berkley DB 4.8 to maintain compatability with other standard wallets.

The Wallet and blockchain are stored in a dedicated volume allowing for easy image upgrades.

# Building
## Build Stages
This Dockerfile uses three stages
* base
builds on debian:10 and installs a couple of things to help in all other stages

* builder
Sets up a build environment and compiles and installs Berkley DB 4.8 as well as EverGreenCoin

* evergreencoin
Sets up the runtime image.

## Build Steps

Before building you need to add the Berkley DB source tarball to the current directory. This can be done with
```
wget http://download.oracle.com/berkeley-db/db-4.8.30.NC.tar.gz
```

Released images are tagged with the EGC version (v1.9.1.0) and an incrementing release number which can be linked to a version of these files. Additionally the latest stable release will also be tagged as latest.


Building can be achieved with
```
docker build -t evergreencoin:v1.9.1.0-01 .
```

## Pushing release Images

If this is good extra tags can be added and pushed to dockerhub
```
docker tag evergreencoin:v1.9.1.0-01 m1ari/evergreencoin:v1.9.1.0-01 
docker tag evergreencoin:v1.9.1.0-01 m1ari/evergreencoin:latest
docker push m1ari/evergreencoin:v1.9.1.0-01 
docker push m1ari/evergreencoin:latest
```

# Running
## Quickstart

You can just run
```
docker run -dt --name=evergreencoin -p 5757:5757 -v evergreencoin:/var/lib/evergreencoin evergreencoin
```
Which will create a volume called evergreencoin if it doesn't already exist, expose the network port and run the daemon.
The blockchain and wallet will be stored within the volume meaning the container can be easily upgraded. It is the users responsibilty to ensure any important data within the volume (the wallet file) is backed up as required.

## Bootstrap with a snapshot
First you will need to download a recent snapshot to your local machine.

You can optionally create the volume manually, however this should be done upon starting the first container.
```
docker volume create evergreencoin
```
This process is a bit more involved as we need to copy the snapshot into the image and extract it before starting the evergreencoin daemon.

Firstly we will create a container and start it running bash in a detatched state. We then copy in the snapshot file and attach to the running bash session
```
docker run -dt --rm --name=tmpegc -v evergreencoin:/var/lib/evergreencoin --entrypoint=bash evergreencoin
docker container cp <snapshot> tmpegc:/tmp
docker container attach tmpegc
```

Inside the docker container we install some extra bits and extract the snapshot. Anything outside /var/lib/evergreencoin will be lost once we exit the bash container.
```
apt update
apt install xz-utils
cd /var/lib/evergreencoin
tar xJvf /tmp/<snapshot>.tar.xz
chown -R 999:999 blk0001.dat database/ txleveldb/
exit
```
Before exiting you can additionally make any extra changes you might wish to the evergreencoin.conf file

Finally you can start up a new production container
```
docker run -dt -p 5757:5757 --name=evergreencoin -v evergreencoin:/var/lib/evergreencoin evergreencoin:latest

```

# TODO
[ ] Use a script as the entry point (this should be able to accept parameters to mimic evergreencoid if it's already running)
[ ] Create random RPC password for each new install `openssl rand -base64 12`
[ ] Auto download a blockchain snapshot
[ ] Setup cron for logrotate
[ ] Add a healthcheck command

# Changelog
v1.9.1.0-01	20210728	Initial Release


# Other Notes

The existing Dockerfile uses three scripts to build an image based on Ubuntu 18.04
* https://gist.github.com/xbrooks/44e375d582ffd4c1b7d7dbf61ad39435
Builds the sourcecode against the packaged libdb (5.3?)
All other steps are replicated in this Dockerfile
Several steps in this are based on building for a non Docker environment so are redundant here

* https://gist.github.com/xbrooks/f142b7ed83d8b0adab9b27d71145b94c
Sets up the runtine environment
This is replicated within this Dockerfile

* https://gist.github.com/xbrooks/05392718769be46086514f839e57c137
This is used as the Entrypoint
Some of the steps are handled by this Dockerfile (using the USER directive to switch which user runs the daemon)


