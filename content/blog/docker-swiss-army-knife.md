---
date: "2017-10-30T20:36:35-05:00"
title: "Docker the Swiss Army Knife"
description: "Using Docker in creative ways to solve developer problems."
tags: [ "Docker", "Software Development" ]
categories: [ "Docker", "How To" ]
type: "post"
---

Docker is a very powerful container technology and is a joy to work with.

However, there are some less obvious uses that can be taken advantage of using Docker's interactive mode.

Check out these interesting Docker usages.

Lets say you want to interact with a database but don't want to install client libraries for this database technology.

For example, maybe you want to query a remote MongoDB database. You can quickly achieve this without installing any 
MongoDB dependencies.

```bash
docker pull mongo:3.4.0
docker run -it --rm mongo:3.4.0 /bin/bash
mongo --host <IP-Address> -u <some-username> -p
```

Our run command needs to be a interactive process, so we specify the -i and -t options together in order to allocate
a tty for the container process.

We also specify the --rm option to tell Docker to remove the Docker container that was created on exit.

Here is an example of importing some data into the remote database using our container,

```bash
docker run -v <path-to-mongo-dump-directory>:/dump -it --rm mongo:2.6.7 /bin/bash
mongorestore --host <IP-Address> -u <some-username> dump
```

You can apply the same pattern to work with a MySQL or PostgreSQL database.

The advantage of using docker is you can specify whatever version of the client database software you want to use
with the added benefit of easy cleanup, simply removing the docker image when you are done with it.

Other interesting things that can be accomplished with docker interactive include trying out new languages or specific 
versions of languages.

```bash
docker pull haskell:8.2.1
docker run -it --rm haskell:8.2.1 /opt/ghc/8.2.1/bin/ghci
Prelude> putStrLn "Hello, World!"
Hello, World!
Prelude> :quit
```

```bash
docker pull node:8.8.1
docker run -it --rm node:8.8.1 /usr/local/bin/node
> console.log('hello world');
hello world
undefined
> .exit
```

Maybe you are stuck using a Windows machine and want to use your favorite Debian command line util,

```bash
docker pull debian:stretch-slim
docker run -it --rm debian:stretch-slim /bin/bash
apt-get moo
                 (__) 
                 (oo) 
           /------\/ 
          / |    ||   
         *  /\---/\ 
            ~~   ~~   
..."Have you mooed today?"...
```
The last example I will show is if you are attempting to debug a running Docker container. There is of course the docker logs
and inspect commands that can help when debugging issues with a running Docker container,

```bash
docker inspect <container-id>
docker logs <container-id>
```

Or if you need more information you can quickly attach to the running container with a shell process and debug within
the running container, 

**at your own risk of course :)**

```bash
docker exec -it <container-id> /bin/bash
ls -al <path-to-interesting-directory>
cat <path-to-interesting-file>
```

I hope these creative Docker examples get your interest peaked for using Docker in unorthodox ways. Of course 
you can always use Docker for deploying software which it does a phenomenal job of as well!

Feel free to let me know if you have any questions or other clever Docker tricks I may not have covered.

Cheers!

Aaron