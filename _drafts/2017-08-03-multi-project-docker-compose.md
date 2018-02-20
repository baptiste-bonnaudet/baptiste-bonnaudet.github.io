---
layout: post
title:  "Create a multiple projects environment using docker-compose"
date:   2017-08-03 11:57:05 -0400
categories: docker docker-compose environment networking
---

Docker-compose for your dev environment is a bliss, it is so much simpler than kubernetes and minikube and recently I managed to make all my docker-compose project interact with each other easily through networking.

## Multiple projects, multiple docker-compose files
Using one file for all is not a good idea because it won't be versioned and it will be hard to keep in sync with all of your projects so for each of your project / git repository use at least one docker-compose file.

What I like to do also is for each project to have one compose file for the daemon services like web/app/db servers and one compose file for tools like db-migrations, building the assets, installing vendor dependencies...

## Networking

The key to make multiple docker projects work together is to manage networking like a boss.

There is multiple way of doing it and everything can be found in the docker-compose documentation. I found the easiest is to have one backbone or main project creating a specific network or just it's default (docker-compose will name default networks `<project_name>_default`) and use this network as the default network override on other projects.

```
networks:
  default:
    external:
      name: mainproject_default
```
This way all your services will be available to each other by default.

## Links

Now that you have a common networking for your containers to talk to each other you can call them by name.

## Bind IPs
