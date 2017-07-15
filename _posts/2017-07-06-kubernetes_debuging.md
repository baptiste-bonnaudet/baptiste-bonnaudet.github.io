---
layout: post
title:  "bash helper for debugging Kubernetes services"
date:   2017-07-06 11:57:05 -0400
categories: kubernetes bash
---

When I started using Kubernetes and ramping up on containers I spent a lot of time debugging, mostly because I was heading in the wrong direction and lacking knowledge. But even now, creating new applications lead to a debug phase, from development environment to staging there is always a need to adapt - even with containers.

Deployments on Kubernetes can be a pain to debug. Mostly because you are always destroying and recreating pods, watching the states of your containers, diving into the logs... And of course the UI is not helping that much and you should not even trust it.

This is why I'm sharing my debugging bash functions that I use every day:

## k8watch - a global view on your deployments, pods and services on a specific namespace
```
# usage: k8watch <namespace>
function k8watch() {
 watch -n 5 "kubectl -n $1 get pods;echo '';kubectl -n $1 get services;echo '';kubectl -n $1 get ingress;echo '';kubectl -n $1 get deployments"
}
```

## k8shell - execute a shell on the first pod matching $2 in namespace $1, use $3 for specific container.
```
# usage: k8shell <namespace> <podname matching pattern> [<container>]
#        k8shell vault-consul-stack "^vault.*0ds" vault
function k8shell(){
 pod=`kubectl -n $1 get pods | egrep "${2}" | head -n 1 | awk '{ print $1 }'`
 if [ -z $3 ]; then
   kubectl -n $1 exec $pod -i -t -- /bin/sh
 else
   kubectl -n $1 exec $pod -c $3 -i -t -- /bin/sh
 fi
}
```

## k8logs - print the logs on the first pod matching $2 in namespace $1, use $3 for specific container.
```
# usage: k8logs <namespace> <podname matching pattern> [<container>]
#        k8logs vault-consul-stack "^vault.*0ds" vault
function k8logs(){
 pod=`kubectl -n $1 get pods | egrep "${2}" | head -n 1 | awk '{ print $1 }'`
 if [ -z $3 ]; then
   kubectl -n $1 logs -f $pod
 else
   kubectl -n $1 logs -f $pod -c $3
 fi
}
```
