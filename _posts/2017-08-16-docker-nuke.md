---
layout: post
title:  "Simple bash function to nuke you docker environment"
date:   2017-08-16 11:57:05 -0400
categories: docker environment yolo
---

This function will remove all docker containers, images, volumes and network from your environment. I use this while debugging and sometimes when the docker for mac VM decides to take a shit.

```
function docker-nuke(){
  docker ps -a -q | xargs -r docker rm -f;
  docker images -q | xargs -r docker rmi -f;
  docker volume ls -q | xargs -r docker volume rm;
  docker network ls -q -f type=custom | xargs -r docker network rm;
}
```
