---
layout: post
title:  "Continuous deployment to Kubernetes with Concourse CI"
date:   2018-02-15 11:57:05 -0400
categories: docker concourse kubernetes
---

![]({{ "/assets/continuous_deploy_concourse.png" | absolute_url }})

I've been using Concourse CI in conjunction with Kubernetes for a year now and I believe it could provides crucial benefits to every company that is working towards a container transition. 

You have dockerized your application, created Kubernetes templates, why would you stop there and continue using a classic CI. A poorly implemented CI based on plugins, or hosted solutions that does not integrate well with Docker, and where you have no control upon will break your workflow and will force you to do workarounds. With Concourse _everything_ is a container. If you designed your containers well, you won't have to change the way you work going from your local environment to Concourse or Kubernetes.

Another massive argument is the scale at which Concourse can operate. As long as you have resources to schedule your containers and volumes, the system will scale. Where I work, we have Concourse workers on an autoscaling group in Google Cloud. At peak traffic we can have dozens of concurrent builds, spinning hundreds of containers and sometimes thousands of volumes.

## Concourse CI notions

- *Resources* are external assets, a Docker image on a registry, a github repository, a slack hook, you name it. They will almost all have checks to trigger jobs, inputs to retrieve and outputs to update/push the resource.
- *Jobs* contain tasks, these tasks will be executed together in parallel or in sequence on the same node (similar to Kubernetes pods). You will be able to pass inputs and outputs from one task to another.
- *Tasks* are like functions, with inputs and outputs. Since it's a container, you can choose which image it will use to fit your needs.

The *pipeline* contains all the jobs necessary to fulfill a purpose. When a resource gets updated it will trigger a *build* which is the logical execution of the job's tasks.

## Build images

### Docker image resource definition:
```
- name: foobar-app-image
  type: docker-image
  source:
    repository: gcr.io/foobar/foobar_app
    username: _json_key
    password: {{gcloud_service_key}}
```

### Build and push the image:
```
- put: foobar-app-image
  params:
    build: artifact-app/
    cache: true
    cache_tag: latest
    tag: metadata/short-commit-hash
    tag_as_latest: true
```
Note: 
- the artifact-app/ volume contains a Dockerfile and dependencies to build the image 
- the `latest` tag will be cached on disk, if we have a lot of builds this can save time!
- The metadata is a simple task that uses a `git` Docker image to retrieve the git short commit hash:
    ```
    ---
    platform: linux

    image_resource:
      type: docker-image
      source: {repository: greenwall/git}

    params:
      TERM: xterm-256color

    inputs:
      - name: code-base

    outputs:
      - name: metadata

    run:
      path: /bin/sh
      args:
      - -exc
      - |
        GIT_SHORT_HASH=`cd code-base && git log --pretty=format:'%h' -n 1`
        echo $GIT_SHORT_HASH > metadata/short-commit-hash
    ```

## Run unit tests on pre-built container
We could easily reuse that image for unit testing, tests that should not depend on anything else than the code itself. Here's an example on how to do it with php-unit:

```
- task: php-unit-tests
  config:
    platform: linux
    image_resource:
      type: docker-image
      source:
        repository: gcr.io/foobar/foobar_app
        username: _json_key
        password: {{gcloud_service_key}}
    run:
      path: sh
      args:
      - -exc
      - |
        /var/www/vendor/phpunit/phpunit/phpunit --testsuite=unit --bootstrap /var/www/includes/tests/bootstrap.php --configuration /var/www/includes/tests/phpunit-conf.xml
  on_failure:
    put: git-pr
    params:
      context: php-unit-tests
      path: git-pr
      status: failure
- put: git-pr
  params:
    context: php-unit-tests
    path: git-pr
    status: success
```


## Continous deployment on staging/production
### Tooling
To manage releases on Kubernetes we use [Helm](http://helm.sh). This tool also require kubectl to be properly configured and since we are using GKE we need the [gcloud](https://cloud.google.com/sdk/downloads) cli. These tools do not exist on Concourse, they are too specific, this is why I built an [alpine-based image containing all these tools](https://github.com/baptiste-bonnaudet/gcloud-kubectl-helm).

### Deploy and rollback on failure

The `deploy-staging` job in the full pipeline below takes a helm charts repository as input, use the gcloud-kubectl-helm Docker image as a task to deploy our application to staging. We use the helm test feature to test that our release was successful before failing. And in case of failure it will attempt a rollback. 

### The pipeline definition for build and deploy
```
resources:
- name: foobar-app-image
  type: docker-image
  source:
    repository: gcr.io/ls-docker/foobar_app
    username: _json_key
    password: {{gcloud_service_key}}

- name: code-base
  type: git
  source:
    uri: git@github.com:foobar/foobar.git
    branch: develop
    private_key: {{deploy_key_foobar}}

- name: charts 
  type: git
  source:
    uri: git@github.com:foobar/charts.git
    branch: master
    private_key: {{deploy_key_charts_git_repo}}


jobs:
- name: build-base-images
  max_in_flight: 1
  plan:
  - get: code-base
    params: {git.depth: 1}
    trigger: true
  - task: fetch-metadata
    file: code-base/ci/tasks/fetch-metadata.yml
  - task: prepare-artifact-app
    config:
      inputs:
      - name: code-base
      outputs:
      - name: artifact-app
      platform: linux
      image_resource:
        type: docker-image
        source: { repository: busybox }
      run:
        path: sh
        args:
        - -exc
        - |
          export ROOT_DIR=$PWD
          [ ... prepare artifact ... ]           
          cat $ROOT_DIR/artifact-app/Dockerfile
  - put: foobar-app-image
    params:
      build: artifact-app/
      cache: true
      cache_tag: latest
      tag: metadata/short-commit-hash
      tag_as_latest: true
      
- name: deploy-staging
  max_in_flight: 1
  plan:
  - get: code-base
    params: {git.depth: 1}
    passed: [build-base-images]
    trigger: true
  - task: fetch-metadata
    file: code-base/ci/tasks/fetch-metadata.yml
  - get: charts    
  - task: deploy
    config:
      platform: linux
      image_resource:
        type: docker-image
        source: {repository: greenwall/gcloud-kubectl-helm}
      params:
        gcloud_service_key: {{gcloud_service_key}}
        GCLOUD_PROJECT: foobar-stg
        ZONE: us-east1-c
        GKE_CLUSTER: foobar-use1-stg
        NAMESPACE: foobar
      inputs:
        - name: charts
        - name: metadata
      run:
        path: /bin/sh
        args:
        - -exc
        - |
          COMMIT_HASH=`cat metadata/short-commit-hash`
          RELEASE_NAME=staging
          # configure & authenticate, this will hide the key from the console
          { echo $gcloud_service_key > secret.json; } 2> /dev/null
          gcloud auth activate-service-account --key-file secret.json
          gcloud config set project $GCLOUD_PROJECT
          gcloud container clusters get-credentials $GKE_CLUSTER --zone $ZONE --project $GCLOUD_PROJECT
          # create helm release
          helm upgrade $RELEASE_NAME charts/foobar/foobar \
            --values charts/foobar/foobar/stg.yaml \
            --set app.image.tag=$COMMIT_HASH \
            --wait \
            --timeout 120
          # test helm release, 3 times before failing
          for i in $(seq 1 3); do
            if helm test $RELEASE_NAME --cleanup; then
              helm status $RELEASE_NAME
              helm history $RELEASE_NAME
              exit 0
            fi
            kubectl get pods --namespace $NAMESPACE | grep $RELEASE_NAME
            kubectl get jobs --namespace $NAMESPACE | grep $RELEASE_NAME
            sleep 60
          done
          echo "[ERROR] RELEASE WAS NOT SUCCESSFUL, PERFORMING ROLLBACK"
          exit 1
    on_failure:
      task: rollback
      config:
        platform: linux
        image_resource:
          type: docker-image
          source: {repository: greenwall/gcloud-kubectl-helm}
        params:
          TERM: xterm-256color
          gcloud_service_key: {{gcloud_service_key}}
          GCLOUD_PROJECT: foobar-stg
          ZONE: us-east1-c
          GKE_CLUSTER: foobar-use1-stg
          NAMESPACE: foobar
        inputs:
          - name: metadata
        run:
          path: /bin/sh
          args:
          - -exc
          - |
            RELEASE_NAME=staging
            # configure & authenticate
            { echo $gcloud_service_key > secret.json; } 2> /dev/null
            gcloud auth activate-service-account --key-file secret.json
            gcloud config set project $GCLOUD_PROJECT
            gcloud container clusters get-credentials $GKE_CLUSTER --zone $ZONE --project $GCLOUD_PROJECT
            
            PREVIOUS_RELEASE=`helm history $RELEASE_NAME | tail -n2 | head -n1 | awk '{ print $1 }'`
            helm rollback --wait --timeout 60 $RELEASE_NAME $PREVIOUS_RELEASE
            # test helm release, 3 times before failing
            for i in $(seq 1 3); do
              sleep 60
              if helm test $RELEASE_NAME --cleanup; then
                helm status $RELEASE_NAME
                helm history $RELEASE_NAME
                exit 0
              fi
              kubectl get pods --namespace $NAMESPACE | grep $RELEASE_NAME
              kubectl get jobs --namespace $NAMESPACE | grep $RELEASE_NAME
            done
            echo "[ERROR] ROLLBACK WAS NOT SUCCESSFUL"
            exit 1
```
### Shipping to production
We just saw how to build an image and automagically deploy it on staging, what is the next step for production?

#### Multiple branch tracking
You can decide to track `develop` for building an image and pushing to staging, and commits on `master` to trigger production update.

This solution would work only if you force merges on the `master` branch to be in sync.

#### External resource for manual go/nogo
Alternatively you can use [an external resource](https://concourse.ci/resource-types.html) tracking such as a Jira ticket or a github status/deployment for manual input.
