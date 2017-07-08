---
layout: post
title:  "Vault funness"
date:   2017-07-08 11:57:05 -0400
categories: vault consul security secrets
---

I've had a long running project of migrating my company's secrets to vault for managing secrets at scale across application.

Our needs were:
* segmentation of secrets per application/component/team
* reducing attack surface by reducing the number of sources for secrets
* replacing unsecure secrets storage (like Chef symetrically encrypted databags)
* providing an API for developers to manage their secrets on their dev environment the same way they are managed in production.
* one source of truth: we know where the secret is and that there is no duplicate
* being able to easily rotate secrets (one source of truth)
* providing auditability on our system.
* delegating authentication so we don't have to manage tokens in any way.

Here is what I learned.

## you don't want to manage tokens.
With vault comes tokens for accessing it, you can generate them, they have a hierarchy and are linked somehow to root tokens and root tokens are risky! If you create an application it will require a token to access the vault, but then you will need to manage it and store it in a secure place... but where ? I thought vault would help me having a secure storage but now I need a secure storage to store my token for my secure storage?

Here comes authentication backend to the rescue. You can use them extensively to delegate authentication. We chose to use 2 of these:
* Github auth backend to authenticate humans
* AWS auth backend to authenticate machines

## 2-man rule is probably not for you
Once you understand that unseal keys are for unsealing and are not related to tokens you read that on dev environment you require 1 unseal key but on prod it would be good practice to have more. Yes it definetly is a good practice, but is it for you? Do you need to secure your secrets like a nuclear submarine?

Ideally yes but in practice it implies being able to store different keys in different KMS, accessible by different people. Like you could use 2 encryption modules and give keys to decrypt to different teams (eg. DevOps and Security). It would also implies adding security team on the pager. Imagine your cloud provider having issues and rebooting your instances and you need to unseal during the night.

## configuration file dafuq?
The configuration files are like having commandline options in a file and that's it. Most of the configuration is _executed_ when the vault is first initialized. Configuration file is only for starting a vault instance with minimum configuration for it to receive request and manage keys and tokens.
You want to add secrets backend? Add an audit backend? load policies? You will have to do it manually or scripted against the instance.

## HA is not scalable
Vault use leader election with an HA backend with failover policy. You can have multiple instance of vault but there will be only one _working_ for real. Whichever your HA strategy (see documentation), you will always have one leader encrypting, decrypting, storing or returning secrets. You can hit a standby but a standby would only redirect you to the leader or act as an interface.

## HA backend is statefulless
Since recent version of Vault you can segregate secret storage backend and HA "storage" backend. It's great because you can store the secrets on an easy to manage and consistent storage like a mysql database or an S3 bucket and leave HA to HA enabled backend (consul, etcd) which are more complicated to manage.

I realized that while doing that I when I intentionally destroy consul, vault goes down, when I recreate the cluster vault goes up again. Looks like a stateless behavior to me, but remember this: only Vault is stateless, it just _needs_ something to be able to talk with it's peers. And this something is statefull! For example loosing quorum on consul will cause issues to vault.

## healthcheck is a mess
There is this choice of architecture in Vault that you can poke him with a curl and the http status code will reflect it's state. But Vault stands apart and works in a unique way so they don't have matching http status code in the standards and they chose at some point not to respect these standards! You thought 400+ status code would be bad? wrong! http 429 you didn't get rate limited you just hit a standby that replied the _content_ you were expecting... this leads to problem of course when you need to integrate this with status checks in Kubernetes, or Newrelic for example. You have to understand every status and workaround them, create custom checks...

## secrets are imutables


##
