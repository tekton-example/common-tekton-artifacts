# How to Setup a Tekton CI/CD Environment on OpenShift 4

*A complete step-by-step guide to create a Tekton (v0.19.0) CI/CD solution on OpenShift 4 or Kubernetes 1.15+.*

## Overview

The goal of this guide is to show how Tekton is used to automate an application's entire CI/CD lifecycle by providing an in-depth walkthrough example. [Tekton](https://github.com/tektoncd) is a cloud-based pipeline tool that was released to beta earlier this year. This guide will show you how to use Tekton, from installation to using it to build and run your applications. By the end of this guide, you will have a complete Tekton-based CI/CD environment that can be modularized and expanded to solve all your development's CI/CD needs.

The pipeline yaml files provided in this guide are used to build and deploy Java applications packaged with Maven, but the guide aims to be thorough enough to build your own custom pipelines. You can swap out the specific pipeline files to suit your specific runtimes or build steps (See [Tekton Hub](https://hub-preview.tekton.dev/) for a catalog of premade resources). This pipeline demonstrates the use of shared Tekton Workspaces as well as Task Results, features released in the latest Tekton beta.

The pipeline will perform the following automated tasks:

1. Pull application and dependency source code from git repositories
2. Install maven dependencies from the dependency repository
3. Build, package, containerize, and push your java application to an external image registry
4. Deploy an application using its manifest files and the application image

## Prerequisites

* Kubernetes-based cluster using v1.16+, such as OpenShift 4+, with admin access. 

  If you don't have one yet, you can follow along with a sandbox kube cluster using [kind](https://kind.sigs.k8s.io/docs/user/quick-start/), or [minikube](https://minikube.sigs.k8s.io/docs/start/). If you need a production cluster, you can make a cloud-managed OpenShift cluster on your favorite cloud provider, such as [IBM Cloud](https://cloud.ibm.com/kubernetes/catalog/create?platformType=openshift), [AWS](https://aws.amazon.com/quickstart/architecture/openshift/), [Azure](https://azure.microsoft.com/en-us/services/openshift/), or [GCP](https://cloud.google.com/solutions/partners/openshift-on-gcp). OpenShift is great with Tekton because it provides integrated UI support to visualize your pipelines.

Feel free to follow along with this [example project](https://github.com/tekton-example) that includes all the resources created in this guide. 

## Step 1. Install Tekton

First, install Tekton on your cluster and the TektonCLI on your local machine. You do not need the CLI tool to use Tekton but it can be useful to interact with the pipelines. Installing Tekton is as simple as logging in to your cluster and running 1-2 commands depending on your platform. I recommend following the official steps below.

1. [TektonCD Getting Started](https://tekton.dev/docs/getting-started/) or [TektonCD Install](https://github.com/tektoncd/pipeline/blob/master/docs/install.md)
2. [TektonCD CLI Install](https://github.com/tektoncd/cli)

OpenShift TektonCD Install Example:

![tekton-openshift-install-example](/Users/sdesmond/projects/2020/git/demos/tekton-example/images/tekton-openshift-install-example.png)

You can also install Tekton using the [Operator](https://operatorhub.io/operator/tektoncd-operator). As of this writing, it uses an older version (0.15.2 vs. 0.19.0) but it comes with a Tekton Dashboard and it might maintain some of pipeline artifacts for you in the future.

## Step 2. Configure Tekton

In this section we will create a ServiceAccount for our pipeline and assign it credentials to handle four things: Run containers as root (for Buildah/Docker), access our external image registry, access our git repositories, and control to modify our application's Deployment yaml file.

### Create and Assign a ServiceAccount

To use the Buildah Task you will need to grant **privileged** access to the Pod running the Task. By default, Tekton uses the cluster's `default` ServiceAccount. We will grant access by creating our own `pipeline` ServiceAccount, assigning it Privileged access and applying it to the PipelineRun.

You will see this as a part of the steps in the `build-maven-image` Task:

```yaml
    securityContext:
      privileged: true
```

To create this SA and apply the *privileged* SecurityContextConstraint with OpenShift, run the following:

`oc create sa pipeline `
`oc adm policy add-scc-to-user privileged -z pipeline`

To use this SA, we will apply this line later in our PipelineRun under *spec:*

`serviceAccountName: pipeline`

However, we will also change our default to use the `pipeline` SA:

```bash
kubectl create configmap config-defaults \
--from-literal=default-service-account=pipeline \
-o yaml -n tekton-pipelines \
--dry-run=client  | kubectl replace -f -
```

Next, we will need to give this ServiceAccount access to resources in the namespace we are using by creating a ClusterRoleBinding. This is needed in our example to reapply the Image tag in the `update-deployment` Task. In this case, I will give the `pipeline` SA the *cluster-admin* role, but you may decide to create your own ClusterRole and scope the pipeline's access further. This SA, the pipeline artifacts, and the application artifacts will be deployed in the `java-maven-demo` namespace.

```bash
oc create clusterrolebinding pipeline --clusterrole=cluster-admin --serviceaccount=java-maven-demo:pipeline
```

### Accessing the External Registry

We need to grant access to our external image registry. To do this, create a secret and link it to the ServiceAccount created before. Here is an example:

```bash
oc create secret docker-registry srd-docker-registry \
--docker-server=docker.io \
--docker-username=<username> \
--docker-password=<api_token> \
--docker-email=<email>
```

Link the secret to your pipeline ServiceAccount like so:

`oc secrets link pipeline srd-docker-registry`

If you need to pull dependencies from an image registry in your pipeline, you will need to grant it this permission in addition to the previous command.

`oc secrets link pipeline srd-docker-registry --for=pull`

### Accessing the Git Repositories

Similar to the external registry, we will create our git secret and link it to the ServiceAccount. Here is a yaml file example with an annotation needed by Tekton.

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: gitsecret
  annotations:
    tekton.dev/git-0: https://github.com
type: kubernetes.io/basic-auth
stringData:
  username: user@email.com
  password: password_token
```

Save this as a yaml file and create it with:

`oc create -f <file_name>`

### Create Pipeline Storage

Lastly, our Pipeline will need a place to clone source code and share cached build artifacts. We will create a 5Gi PersistentVolumeClaim, which will later be given as a parameter in the PipelineRun to use when executing the Pipeline.

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pipeline-pvc
spec:
  resources:
    requests:
      storage: 5Gi
  volumeMode: Filesystem
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain 
```

That is all for Tekton configuration. For more information on authentication with Tekton check out this [document](https://github.com/tektoncd/pipeline/blob/master/docs/auth.md).

## 2. Manually Deploy our Application

With Tekton ready to use on our cluster, let's first walk through the deployment steps for our application so we can translate the process to Tekton for automation.

I have generated an example Java application [here](https://github.com/tekton-example/java-app-example). This app was generated with Quarkus and it comes with a maven `pom.xml` file and a `Dockerfile` under the path `src/main/docker/Dockerfie.jvm`. I modified the `pom.xml` file to require [this](https://github.com/tekton-example/common-java-dependencies) second repository as a build dependency.

To build this application ourselves, we would need to perform the following steps:

1. Pull the dependency library and install into the workspace with `mvn install`
2. Pull the application's source code repository
3. Package the application into a jar file with `mvn package`
4. Containerize the application using the Dockerfile and push the image to an external registry
5. Apply the application's latest manifest files to the cluster stored under `k8s/` directory
6. Update the deployment to use the latest image from the registry

By the end of this guide, we will instead commit our code and execute a PipelineRun, which will deploy the latest code using an image that is tagged with the latest commit's sha string from git.

## 3. Setup Tekton Artifacts

It is recommended to use version control for your artifacts as a backup for easy reference, modification or collaboration. Depending on the size of the project, some users decide to store their tekton artifacts alongside other kubernetes-related yaml files. I will create a stand-alone repository on [GitHub](https://github.com/tekton-example/common-tekton-artifacts) to store tekton-related artifacts called `common-tekton-artifacts`.

The repository contains folders for the artifacts like so:

*resources/*
*tasks/*
*pipelines/*

As our project grows and our CI/CD needs expand, we can use this framework to easily manage our pipelines and resources. Next, we will create our Pipeline's Tasks.

## Create Tasks

A task contains a series of steps, where each step uses an image provided to create a container and run commands or scripts specified. Tekton creates Pods to execute Tasks. 

We will need to create Tasks to complete each manual step of the deployment and then we will link them together with a Pipeline. Our Tasks will be able to share resources with each other by using a shared Workspace, which is a Tekton property representing a PersistentVolumeClaim. We will create Tasks that can be modularized for use with other Pipelines.

#### Task git-clone: Clone the Source Code repository

The first step in our CI/CD solution is to clone the application's source code and its dependency's source code into a workspace. Here is a standalone task from the Tekton Catalog we can use:

```yaml
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
  name: git-clone
  labels:
    app.kubernetes.io/version: "0.1"
  annotations:
    tekton.dev/pipelines.minVersion: "0.12.1"
    tekton.dev/tags: git
    tekton.dev/displayName: "git clone"
spec:
  description: >-
    These Tasks are Git tasks to work with repositories used by other tasks
    in your Pipeline.
    The git-clone Task will clone a repo from the provided url into the
    output Workspace. By default the repo will be cloned into the root of
    your Workspace. You can clone into a subdirectory by setting this Task's
    subdirectory param.
  workspaces:
    - name: output
      description: The git repo will be cloned onto the volume backing this workspace
  params:
    - name: url
      description: git url to clone
      type: string
    - name: revision
      description: git revision to checkout (branch, tag, sha, ref…)
      type: string
      default: master
    - name: refspec
      description: (optional) git refspec to fetch before checking out revision
      default: ""
    - name: submodules
      description: defines if the resource should initialize and fetch the submodules
      type: string
      default: "true"
    - name: depth
      description: performs a shallow clone where only the most recent commit(s) will be fetched
      type: string
      default: "1"
    - name: sslVerify
      description: defines if http.sslVerify should be set to true or false in the global git config
      type: string
      default: "true"
    - name: subdirectory
      description: subdirectory inside the "output" workspace to clone the git repo into
      type: string
      default: ""
    - name: deleteExisting
      description: clean out the contents of the repo's destination directory (if it already exists) before trying to clone the repo there
      type: string
      default: "true"
    - name: httpProxy
      description: git HTTP proxy server for non-SSL requests
      type: string
      default: ""
    - name: httpsProxy
      description: git HTTPS proxy server for SSL requests
      type: string
      default: ""
    - name: noProxy
      description: git no proxy - opt out of proxying HTTP/HTTPS requests
      type: string
      default: ""
    - name: gitInitImage
      description: The image used where the git-init binary is.
      default: "gcr.io/tekton-releases/github.com/tektoncd/pipeline/cmd/git-init:v0.15.2"
      type: string
  results:
    - name: commit
      description: The precise commit SHA that was fetched by this Task
  steps:
    - name: clone
      image: $(params.gitInitImage)
      script: |
        CHECKOUT_DIR="$(workspaces.output.path)/$(params.subdirectory)"
        cleandir() {
          # Delete any existing contents of the repo directory if it exists.
          #
          # We don't just "rm -rf $CHECKOUT_DIR" because $CHECKOUT_DIR might be "/"
          # or the root of a mounted volume.
          if [[ -d "$CHECKOUT_DIR" ]] ; then
            # Delete non-hidden files and directories
            rm -rf "$CHECKOUT_DIR"/*
            # Delete files and directories starting with . but excluding ..
            rm -rf "$CHECKOUT_DIR"/.[!.]*
            # Delete files and directories starting with .. plus any other character
            rm -rf "$CHECKOUT_DIR"/..?*
          fi
        }
        if [[ "$(params.deleteExisting)" == "true" ]] ; then
          cleandir
        fi
        test -z "$(params.httpProxy)" || export HTTP_PROXY=$(params.httpProxy)
        test -z "$(params.httpsProxy)" || export HTTPS_PROXY=$(params.httpsProxy)
        test -z "$(params.noProxy)" || export NO_PROXY=$(params.noProxy)
        /ko-app/git-init \
          -url "$(params.url)" \
          -revision "$(params.revision)" \
          -refspec "$(params.refspec)" \
          -path "$CHECKOUT_DIR" \
          -sslVerify="$(params.sslVerify)" \
          -submodules="$(params.submodules)" \
          -depth "$(params.depth)"
        cd "$CHECKOUT_DIR"
        RESULT_SHA="$(git rev-parse HEAD | tr -d '\n')"
        EXIT_CODE="$?"
        if [ "$EXIT_CODE" != 0 ]
        then
          exit $EXIT_CODE
        fi
        # Make sure we don't add a trailing newline to the result!
        echo -n "$RESULT_SHA" > $(results.commit.path)
```

The only required parameter for the `git-clone` Task is the *url* to our source code, i.e. `https://github.com/tekton-example/java-app-example.git`

We can specify additional parameters such as the branch to clone or a proxy to use. In our case, we will be using the `subdirectory` parameter to clone the source code into a folder in the workspace rather than at the root of the directory. We will also use the `commit` result later to tag our image. will use this Task twice in our pipeline to clone our dependency and application repositories.

#### Task maven-build: Install Dependencies

The following Task uses an image with maven to run a `mvn install` by first generating or using an existing settings.xml file in the workspace and then installing the package into the `source` workspace. The workspace will be given to the *Pipeline* by a param in the *PipelineRun*.

```yaml
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
  name: maven
  labels:
    app.kubernetes.io/version: "0.2"
  annotations:
    tekton.dev/pipelines.minVersion: "0.12.1"
    tekton.dev/tags: build-tool
spec:
  description: >-
    This Task can be used to run a Maven build.
  workspaces:
    - name: source
      description: The workspace consisting of the maven project and custom maven settings.
  params:
    - name: MAVEN_IMAGE
      type: string
      description: Maven base image
      default: gcr.io/cloud-builders/mvn@sha256:57523fc43394d6d9d2414ee8d1c85ed7a13460cbb268c3cd16d28cfb3859e641 #tag: latest
    - name: GOALS
      description: maven goals to run
      type: array
      default:
        - "package"
    - name: MAVEN_MIRROR_URL
      description: The Maven repository mirror url
      type: string
      default: ""
    - name: SERVER_USER
      description: The username for the server
      type: string
      default: ""
    - name: SERVER_PASSWORD
      description: The password for the server
      type: string
      default: ""
    - name: PROXY_USER
      description: The username for the proxy server
      type: string
      default: ""
    - name: PROXY_PASSWORD
      description: The password for the proxy server
      type: string
      default: ""
    - name: PROXY_PORT
      description: Port number for the proxy server
      type: string
      default: ""
    - name: PROXY_HOST
      description: Proxy server Host
      type: string
      default: ""
    - name: PROXY_NON_PROXY_HOSTS
      description: Non proxy server host
      type: string
      default: ""
    - name: PROXY_PROTOCOL
      description: Protocol for the proxy ie http or https
      type: string
      default: "http"
    - name: CONTEXT_DIR
      type: string
      description: >-
        The context directory within the repository for sources on
        which we want to execute maven goals.
      default: "."
  steps:
    - name: mvn-settings
      image: registry.access.redhat.com/ubi8/ubi-minimal:8.2
      script: |
        #!/usr/bin/env bash
        [[ -f $(workspaces.source.path)/settings.xml ]] && \
        echo 'using existing $(workspaces.source.path)/settings.xml' && exit 0
        cat > $(workspaces.source.path)/settings.xml <<EOF
        <settings>
          <servers>
            <!-- The servers added here are generated from environment variables. Don't change. -->
            <!-- ### SERVER's USER INFO from ENV ### -->
          </servers>
          <mirrors>
            <!-- The mirrors added here are generated from environment variables. Don't change. -->
            <!-- ### mirrors from ENV ### -->
          </mirrors>
          <proxies>
            <!-- The proxies added here are generated from environment variables. Don't change. -->
            <!-- ### HTTP proxy from ENV ### -->
          </proxies>
        </settings>
        EOF
        xml=""
        if [ -n "$(params.PROXY_HOST)" -a -n "$(params.PROXY_PORT)" ]; then
          xml="<proxy>\
            <id>genproxy</id>\
            <active>true</active>\
            <protocol>$(params.PROXY_PROTOCOL)</protocol>\
            <host>$(params.PROXY_HOST)</host>\
            <port>$(params.PROXY_PORT)</port>"
          if [ -n "$(params.PROXY_USER)" -a -n "$(params.PROXY_PASSWORD)" ]; then
            xml="$xml\
                <username>$(params.PROXY_USER)</username>\
                <password>$(params.PROXY_PASSWORD)</password>"
          fi
          if [ -n "$(params.PROXY_NON_PROXY_HOSTS)" ]; then
            xml="$xml\
                <nonProxyHosts>$(params.PROXY_NON_PROXY_HOSTS)</nonProxyHosts>"
          fi
          xml="$xml\
              </proxy>"
          sed -i "s|<!-- ### HTTP proxy from ENV ### -->|$xml|" $(workspaces.source.path)/settings.xml
        fi
        if [ -n "$(params.SERVER_USER)" -a -n "$(params.SERVER_PASSWORD)" ]; then
          xml="<server>\
            <id>serverid</id>"
          xml="$xml\
                <username>$(params.SERVER_USER)</username>\
                <password>$(params.SERVER_PASSWORD)</password>"
          xml="$xml\
              </server>"
          sed -i "s|<!-- ### SERVER's USER INFO from ENV ### -->|$xml|" $(workspaces.source.path)/settings.xml
        fi
        if [ -n "$(params.MAVEN_MIRROR_URL)" ]; then
          xml="    <mirror>\
            <id>mirror.default</id>\
            <url>$(params.MAVEN_MIRROR_URL)</url>\
            <mirrorOf>central</mirrorOf>\
          </mirror>"
          sed -i "s|<!-- ### mirrors from ENV ### -->|$xml|" $(workspaces.source.path)/settings.xml
        fi
    - name: mvn-goals
      image: $(params.MAVEN_IMAGE)
      workingDir: $(workspaces.source.path)/$(params.CONTEXT_DIR)
      command: ["/usr/bin/mvn"]
      args:
        - -s
        - $(workspaces.source.path)/settings.xml
        - "$(params.GOALS)"
```

The task will run `mvn package` by default and it can be passed a **GOALS** parameter to specify other arguments. We will use this Task to run `mvn install` on our dependency repository. While we could use this Task to package our application as well, but we will instead include this step in the next Task for our application source code since we will have similar steps to containerize and push the application to an image registry.

#### Task build-maven-image: Build, Containerize, and Push the Application Image

This Task will use the application source code to package, containerize, and push our application image to an external registry. If we were doing these steps manually using my own docker registry, it would look like this:

1. `mvn package`
2. `docker build -f src/main/docker/Dockerfile.jvm -t docker.io/sdesmond6/java-app-example:latest .`
3. `docker push docker.io/sdesmond6/java-app-example:latest`

```yaml
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
  name: build-maven-image
  labels:
    app.kubernetes.io/version: "0.1"
  annotations:
    tekton.dev/pipelines.minVersion: "0.12.1"
    tekton.dev/tags: image-build
spec:
  description: >-
    Packages source with maven builds and into a container image,
    then pushes it to a container registry.

    Builds source into a container image using Project Atomic's
    Buildah build tool. It uses Buildah's support for building from Dockerfiles,
    using its buildah bud command.This command executes the directives in the
    Dockerfile to assemble a container image, then pushes that image to a
    container registry.

  params:
  - name: GOALS
    description: Maven goals to execute
    default: ["install"]
  - name: IMAGE
    description: Reference of the image buildah will produce.
  - name: BUILDER_IMAGE
    description: The location of the buildah builder image.
    default: quay.io/buildah/stable:v1.14.8
  - name: STORAGE_DRIVER
    description: Set buildah storage driver
    default: overlay
  - name: DOCKERFILE
    description: Path to the Dockerfile to build.
    default: ./Dockerfile
  - name: CONTEXT
    description: Path to the directory to use as context.
    default: .
  - name: TLSVERIFY
    description: Verify the TLS on the registry endpoint (for push/pull to a non-TLS registry)
    default: "false"
  - name: FORMAT
    description: The format of the built container, oci or docker
    default: "oci"
  - name: BUILD_EXTRA_ARGS
    description: Extra parameters passed for the build command when building images.
    default: ""
  - name: PUSH_EXTRA_ARGS
    description: Extra parameters passed for the push command when pushing images.
    type: string
    default: ""
  workspaces:
  - name: source

  results:
  - name: IMAGE_DIGEST
    description: Digest of the image just built.

  steps:
  - name: package-maven
    image: gcr.io/cloud-builders/mvn
    workingDir: $(workspaces.source.path)/$(params.CONTEXT)
    command: ["/usr/bin/mvn"]
    args:
      - -Dmaven.repo.local=$(workspaces.source.path)
      - "$(inputs.params.GOALS)"

  - name: build-image
    image: $(params.BUILDER_IMAGE)
    workingDir: $(workspaces.source.path)
    script: |
      buildah --storage-driver=$(params.STORAGE_DRIVER) bud \
        $(params.BUILD_EXTRA_ARGS) --format=$(params.FORMAT) \
        --tls-verify=$(params.TLSVERIFY) --no-cache \
        -f $(params.DOCKERFILE) -t $(params.IMAGE) $(params.CONTEXT)
    volumeMounts:
    - name: varlibcontainers
      mountPath: /var/lib/containers
    securityContext:
      privileged: true

  - name: push
    image: $(params.BUILDER_IMAGE)
    workingDir: $(workspaces.source.path)
    script: |
      buildah --storage-driver=$(params.STORAGE_DRIVER) push \
        $(params.PUSH_EXTRA_ARGS) --tls-verify=$(params.TLSVERIFY) \
        --digestfile $(workspaces.source.path)/image-digest $(params.IMAGE) \
        docker://$(params.IMAGE)
    volumeMounts:
    - name: varlibcontainers
      mountPath: /var/lib/containers
    securityContext:
      privileged: true

  - name: digest-to-results
    image: $(params.BUILDER_IMAGE)
    script: cat $(workspaces.source.path)/image-digest | tee /tekton/results/IMAGE_DIGEST

  volumes:
  - name: varlibcontainers
    emptyDir: {}
```

The Task uses the **buildah** tool to build the source code from the Dockerfile and push it to the registry image name provided. This Task will be ran with the ServiceAccount created previously that has the authorization to run buildah and push to the registry.

#### Task apply-manifests: Apply Kubernetes Manifest Files

In this example, we follow a common standard and store our application's manifest files alongside our source code under a folder called `k8s/`. These define how we expect the application to run inside our cluster. In this example application, this includes a Deployment, Service, and Route as it will be deployed on OpenShift. You can create your own manifest files for your application and store them in a `k8s/` folder. An alternative approach may be to store all of your applications' manifest files in a single repository (i.e. if you use ArgoCD for your CD needs), but we are planning to use this pipeline as a complete solution.

```yaml
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
  name: apply-manifests
spec:
  workspaces:
  - name: source
  params:
    - name: manifest_dir
      description: The directory in source that contains yaml manifests
      type: string
      default: "k8s"
  steps:
    - name: apply
      image: quay.io/openshift/origin-cli:latest
      workingDir: /workspace/source
      command: ["/bin/bash", "-c"]
      args:
        - |-
          echo Applying manifests in $(inputs.params.manifest_dir) directory
          oc apply -f $(inputs.params.manifest_dir)
          echo -----------------------------------
```

This Task uses `oc` CLI with the image `origin-cli`. This image also contains `kubectl`. Additionally, it runs an `apply` command, meaning that the manifests must be created already in the cluster before the first execution, i.e. `kubectl create -f k8s/`.

#### Task update-deployment: Update Deployment Manifest to Use Latest Image

The last step in our Pipeline will be to patch our Deployment file with the latest image. This can be customized in a number of ways. We will be using the sha commit result of our *git-clone* Task to tag the image with our latest git commit. In the case of OpenShift, you could use a DeploymentConfig with an ImageChange trigger and an ImageStream, allowing OpenShift to handle this Task.

```yaml
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
  name: update-deployment
spec:
  params:
    - name: deployment
      description: The name of the deployment patch the image
      type: string
    - name: IMAGE
      description: Location of image to be patched with
      type: string
  steps:
    - name: patch
      image: quay.io/openshift/origin-cli:latest
      command: ["/bin/bash", "-c"]
      args:
        - |-
          oc patch deployment $(inputs.params.deployment) --patch='{"spec":{"template":{"spec":{
            "containers":[{
              "name": "$(inputs.params.deployment)",
              "image":"$(inputs.params.IMAGE)"
            }]
          }}}}'
```

We will provide the Task with the image name and a name for the application. The patch will update the Deployment and trigger the application to redeploy using the image.

### Task Recap

With our task files finished, we can add them all to our cluster: `oc create -f tasks/`.

At this point, you can test each Task directly by creating TaskRun files, or by using the Tekton CLI to generate them, i.e.

```bash
tkn task start apply-manifests --param manifest_dir=./app-git-clone/k8s --workspace name=local-maven-repo,claimName=maven-repo-pvc  --showlog
```

Next, we will create a Pipeline to combine these Tasks to finish our CI/CD solution.

## Create the Pipeline

The Pipeline will represent a list of Tasks to execute, passing the necessary parameters to each Task. The pipeline can provide default arguments to Tasks and will also accept parameters via a PipelineRun. This means we can use the Pipeline to deploy any number of applications by creating a PipelineRun specific to each application.

```yaml
apiVersion: tekton.dev/v1beta1
kind: Pipeline
metadata:
  name: maven-pipeline
spec:
  params:
  - name: dependency-git-url
  - name: dependency-git-revision
    default: main
  - name: dependency-folder-name
    default: dependency-source
  - name: application-git-url
  - name: application-git-revision
    default: main
  - name: application-folder-name
    default: application-source
  - name: dockerfile-path
    default: "."
  - name: application-name
  - name: image-name
  workspaces:
  - name: source
  tasks:
  - name: dependency-git-clone
    taskRef:
      name: git-clone
    params:
    - name: url
      value: $(params.dependency-git-url)
    - name: revision
      value: $(params.dependency-git-revision)
    - name: subdirectory
      value: $(params.dependency-folder-name)
    workspaces:
    - name: output
      workspace: source
  - name: application-git-clone
    runAfter:
      - dependency-git-clone
    taskRef:
      name: git-clone
    params:
    - name: url
      value: $(params.application-git-url)
    - name: revision
      value: $(params.application-git-revision)
    - name: subdirectory
      value: $(params.application-folder-name)
    workspaces:
    - name: output
      workspace: source
  - name: maven-install-dependencies
    runAfter:
      - application-git-clone
    taskRef:
      name: maven
    params:
    - name: GOALS
      value: ["-DskipTests","clean","install"]
    - name: CONTEXT_DIR
      value: $(params.dependency-folder-name)
    workspaces:
    - name: source
      workspace: source
  - name: build-application-image
    runAfter:
      - maven-install-dependencies
    taskRef:
      name: build-maven-image
    params:
    - name: GOALS
      value: ["-DskipTests", "clean", "package"]
    - name: IMAGE
      value: $(params.image-name)":"$(tasks.application-git-clone.results.commit)
    - name: CONTEXT
      value: $(params.application-folder-name)
    - name: DOCKERFILE
      value: $(params.dockerfile-path)
    workspaces:
    - name: source
      workspace: source
  - name: apply-manifests
    runAfter:
    - build-application-image
    taskRef:
      name: apply-manifests
    params:
      - name: manifest_dir
        value: $(params.application-folder-name)/k8s
    workspaces:
    - name: source
      workspace: source
  - name: update-deployment
    runAfter:
    - apply-manifests
    taskRef:
      name: update-deployment
    params:
    - name: deployment
      value: $(params.application-name)
    - name: IMAGE
      value: $(params.image-name)":"$(tasks.application-git-clone.results.commit)
```

Once we've wired our Pipeline up we can add it to the cluster: `oc create -f <file_name>`.

### Note on Task Concurrency

You can build your pipeline to run tasks in parallel. If you do, keep in mind that there can be issues with trying to mount Workspaces concurrently if they are backed by the same PVC and it is ReadWriteOnce. While Tekton is in beta, I would suggest only running Tasks in parallel if your build process allows it and the Tasks don't need a shared Workspace. You can do so by removing the `runAfter` property in the Pipeline under the Task.

## Create a PipelineRun

With our Tasks and our Pipeline created, we are now ready to define an execution of the pipeline for our [example application](https://github.com/tekton-example/java-app-example). The PipelineRun will need to supply the required parameters for the Pipeline, which ultimately get used by the Tasks.

```yaml
apiVersion: tekton.dev/v1beta1
kind: PipelineRun
metadata:
  generateName: java-app-example-pl-
spec:
  pipelineRef:
    name: maven-pipeline
  serviceAccountName: pipeline
  params:
  - name: application-name
    value: java-app-example
  - name: dependency-git-url
    value: https://github.com/tekton-example/common-java-dependencies.git
  - name: application-git-url
    value: https://github.com/tekton-example/java-app-example.git
  - name: dockerfile-path
    value: src/main/docker/Dockerfile.jvm
  - name: image-name
    value: docker.io/sdesmond6/java-app-example
  workspaces:
  - name: source
    persistentVolumeClaim:
      claimName: pipeline-pvc
```

The PipelineRun ties everything together for this application. We provide the `pipeline-pvc` created previously, and use the ServiceAccount which has access to our git repositories and the image registry. If we needed, we could also configure our dependencies or application source code to use a specific branch, but that can be left as an exercise.

## Execute the Pipeline

With the pipeline in place, running it is simple. First commit your code to the default `master` branch, and then create the PipelineRun:

`oc create -f java-app-example-pipelinerun.yaml`.

You can monitor the progress of the pipeline with the Tekton CLI:

`tkn pr logs -Lf`

Each PipelineRun created represents one execution of the Pipeline. You can look at all your runs with `oc get pr`, and describe each with `oc describe pr`.

## Summary

That concludes this guide. We covered how to install, configure, and organize a Tekton CI/CD environment for your cluster, how to create and organize your artifacts, and then we created a complete Pipeline example for a Java application that utilizes Tekton's latest workspaces and results features. If you completed this guide, I hope you have noticed how modular and extensible Tekton can be. 

If you would like to contribute to the example project space please send a message. If you found this guide helpful, please let me know what you liked or what can be improved so that I can improve future content.

## Next Steps

This guide provides a CI/CD framework to get started in your own environment. Once you have a running pipeline for your applications, you can configure a webhook to trigger your pipeline automatically via a Pull Request with [triggers](https://github.com/tektoncd/triggers). There are many useful Tasks in the [Tekton Catalog](https://hub-preview.tekton.dev/) created by the community, such as one for a [sonarqube](https://hub-preview.tekton.dev/detail/88) security scanner, or one that creates a [GitHub release](https://hub-preview.tekton.dev/detail/21).