### Pushing Application Images to External Registry

OpenShift Container Platform comes with an internal registry. By default when you create an application the build configuration is set up to push the images  into the internal registry and the deployment configuration is set up to pull images from this internal registry.

Some people may be interested in OpenShift pushing the resultant application docker image ,after it is built, into a separate docker registry that they run outside OpenShift.

Here we will walk through a step by step example to accomplish the same:

####Step 1: Create a new project:
 Run the following commmand to create a new project
 
 ```
 oc new-project ext-image-push
 ```
 
####Step 2: Setup Docker Credentials as a Secret
Here I am using DockerHub Registry as an example to push to. But you can use any Docker Registry of your choice. 

**Note** You will not be able to push RHEL based application images to DockerHub

We will supply **.docker/config.json** file with valid Docker Registry credentials in order to push the output image into a private Docker Registry or pull the builder image from the private Docker Registry that requires authentication. 

The **.docker/config.json** file is found in your home directory by default and has the following type of content (you may have json instead of yaml):

```
auths:
  https://index.docker.io/v1/: 
    auth: "YWRfbGzhcGU6R2labnRib21ifTE=" 
    email: "user@example.com" 
```


If you connected to DockerHub before, you will already have this in your home directory. If you are using a different internal registry you will have the respective registry config here (eg: `myregistry.mycompany.io` instead of `index.docker.io`) 

Run the following command to add a secret to your project that pulls the `auth` details from the docker config. We are naming the secret as `dockerhub`. The output of the command shows `secret/dockerhub` is created.

```
$ oc secrets new dockerhub ~/.docker/config.json
secret/dockerhub
```

You can also verify the list of secrets by running `oc get secrets` as shown below. This also lists the secret `dockerhub` that we just created.

```
$ oc get secrets
NAME                       TYPE                                  DATA      AGE
builder-dockercfg-3073d    kubernetes.io/dockercfg               1         11m
builder-token-gmhiw        kubernetes.io/service-account-token   3         11m
builder-token-n7ajm        kubernetes.io/service-account-token   3         11m
default-dockercfg-35ywk    kubernetes.io/dockercfg               1         11m
default-token-3zq09        kubernetes.io/service-account-token   3         11m
default-token-6sgod        kubernetes.io/service-account-token   3         11m
deployer-dockercfg-gm2fy   kubernetes.io/dockercfg               1         11m
deployer-token-3and7       kubernetes.io/service-account-token   3         11m
deployer-token-fgblq       kubernetes.io/service-account-token   3         11m
dockerhub                  Opaque                                1         2m
```


Alternately, (if you have not logged into DockerHub from your workstation before,)  you can use to create the secret by passing the credentials directly

```
oc secrets new-dockercfg dockerhub --docker-server=DOCKER_REGISTRY_SERVER --docker-username=DOCKER_USER --docker-password=DOCKER_PASSWORD --docker-email=DOCKER_EMAIL
```

####Step 3: Add secret to your builder service account

Each build in your project runs with using the builder service account. To get a list of service accounts in your project run and it shows the builder service account as one of the service accounts that are automatically created for you when you created a new project. **Note**  `sa` is short form for service account.

```
$ oc get sa
NAME       SECRETS   AGE
builder    2         34m
default    2         34m
deployer   2         34m

```

We will now edit the builder SA and add the secret in there. Run the following command and it brings up the builder service account configuration in your default editor

```
$ oc edit sa builder
```


Edit the file to add the `dockerhub` secret to the secrets list as shown below. Be careful about the indentation.


```
# Please edit the object below. Lines beginning with a '#' will be ignored,
# and an empty file will abort the edit. If an error occurs while saving this file will be
# reopened with the relevant failures.
#
apiVersion: v1
imagePullSecrets:
- name: builder-dockercfg-3073d
kind: ServiceAccount
metadata:
  creationTimestamp: 2016-08-18T23:34:00Z
  name: builder
  namespace: ext-image-push
  resourceVersion: "636656"
  selfLink: /api/v1/namespaces/ext-image-push/serviceaccounts/builder
  uid: 3db20d64-659c-11e6-9178-fa163e69a197
secrets:
- name: builder-token-gmhiw
- name: builder-dockercfg-3073d
- name: dockerhub
```
Now your builder can use the newly created secret.

####Step 4: Create a new build in your project

I am using a Dockerfile in a git repository as an example here. Since I am pushing to DockerHub in this example, I am using a non-RHEL base image (busybox). 

```
$ oc new-build https://github.com/VeerMuchandi/time --context-dir=busybox
--> Found Docker image b05baf0 (8 weeks old) from Docker Hub for "busybox"

    * An image stream will be created as "busybox:latest" that will track the source image
    * A Docker build using source code from https://github.com/VeerMuchandi/time will be created
      * The resulting image will be pushed to image stream "time:latest"
      * Every time "busybox:latest" changes a new build will be triggered

--> Creating resources with label build=time ...
    imagestream "busybox" created
    imagestream "time" created
    buildconfig "time" created
--> Success
    Build configuration "time" created and build triggered.
    Run 'oc logs -f bc/time' to stream the build progress.
```

Kubernetes creates an ImageStream for `busybox` within this project and downloads the `busybox` container image (from DockerHub in this case, as I am using it from DockerHub).

It also creates an ImageStream `time` to push the resultant application image to, once the build is complete.

We also have a BuildConfiguration with name `time` that has specification on how the build is done.

The following command lists all the buildconfigs in your project. **Note** `bc` is the short form for buildconfig.

```
$ oc get bc
NAME      TYPE      FROM      LATEST
time      Docker    Git       1
```

####Step 5: Edit BuildConfig to push to your Docker Registry

Now let us edit the BuildConfig to push the docker registry of your choice (in this case I am pushing to DockerHub).

Run the following command to bring up the buildConfig in the editor. 

```
$ oc edit bc time
```

Alternately, you can also do this from the WebConsole.  

*  Get into your Project 
*  **Browse** -> **Builds**-> Select `time`
*  On the right top Select **Actions**-> **Edit YAML**


Locate the following code in your editor:

```
spec:
  output:
    to:
      kind: ImageStreamTag
      name: time:latest
```
Note that this is telling the builder to push the output to the ImageStream `time` with ImageStreamTag of `time:latest`

Now, we will change this to point to your Docker Registry as follows. You can use the Docker registry of your choice (in which case you will change the `docker.io` to let us say `myregistry.mycompany.io`)

```
spec:
  output:
    to:
      kind: DockerImage   
      name: docker.io/veermuchandi/mytime:latest
    pushSecret:
      name: dockerhub
```

In addition we also add the `pushSecret` here to let the builder use that particular secret while pushing into this registry.

**Be extra cautious** with indentation

---
**Temporary Change**
Currently we are running an issue as DockerHub suddenly switched to a new version and stopped supporting older manifests. We need to do a quick workaround until we take care of this incompatibility. Make one more change to the buildConfig to use dockerimage for busybox, instead of pulling from ImageStream.

This code needs to be changed 

```
  strategy:
    dockerStrategy:
      from:
        kind: ImageStreamTag
        name: busybox:latest
    type: Docker

```

to

```
  strategy:
    dockerStrategy:
      from:
        kind: DockerImage
        name: busybox:latest
    type: Docker
```
---


####Step 5: Run Build

Now we are done with the changes to the build configuration and ready to run the build. This invokes dockerbuild within OpenShift and pushes the resultant image to external docker registry.

Start a new build with the command below:

```
$ oc start-build time
time-2
```
This started a new build with name `time-2' and spins up a pod with name `time-2-build`. The command `oc get pods` will show you the list of running pods.

To look at the logs run

```
$ oc logs time-2-build -f
I0818 20:53:19.434894       1 builder.go:57] Master version "v3.2.1.9-1-g2265530", Builder version "v3.2.1.9-1-g2265530"
I0818 20:53:19.567803       1 builder.go:145] Running build with cgroup limits: api.CGroupLimits{MemoryLimitBytes:92233720368547, CPUShares:2, CPUPeriod:100000, CPUQuota:-1, MemorySwap:92233720368547}
I0818 20:53:19.648243       1 source.go:197] Downloading "https://github.com/VeerMuchandi/time" ...
I0818 20:53:22.620006       1 source.go:208] Cloning source from https://github.com/VeerMuchandi/time
I0818 20:53:24.982265       1 docker.go:262] Checking for Docker config file for PULL_DOCKERCFG_PATH in path /var/run/secrets/openshift.io/pull
I0818 20:53:24.982411       1 docker.go:267] Using Docker config file /var/run/secrets/openshift.io/pull/.dockercfg
Step 1 : FROM busybox
 ---> 2b8fd9751c4c
Step 2 : MAINTAINER Veer Muchandi veer@redhat.com
 ---> Using cache
 ---> 3a33c3776759
Step 3 : ADD ./init.sh ./
 ---> Using cache
 ---> 7c96494a77bb
Step 4 : EXPOSE 8080
 ---> Using cache
 ---> 834eef421056
Step 5 : CMD ./init.sh
 ---> Using cache
 ---> 5217eade1002
Step 6 : ENV "OPENSHIFT_BUILD_NAME" "time-2" "OPENSHIFT_BUILD_NAMESPACE" "ext-image-push" "OPENSHIFT_BUILD_SOURCE" "https://github.com/VeerMuchandi/time" "OPENSHIFT_BUILD_COMMIT" "1e1ed2c579f98bc35de25c34952825419bb0a500"
 ---> Running in 494f09eb61d4
 ---> 74b746f15b90
Removing intermediate container 494f09eb61d4
Step 7 : LABEL "io.openshift.build.commit.date" "Tue Aug 16 20:28:41 2016 -0400" "io.openshift.build.commit.id" "1e1ed2c579f98bc35de25c34952825419bb0a500" "io.openshift.build.commit.ref" "master" "io.openshift.build.commit.message" "fixed init.sh" "io.openshift.build.source-location" "https://github.com/VeerMuchandi/time" "io.openshift.build.source-context-dir" "busybox" "io.openshift.build.commit.author" "VeerMuchandi \u003cveer.muchandi@gmail.com\u003e"
 ---> Running in ca858e5d2ccb
 ---> 87cb10858edd
Removing intermediate container ca858e5d2ccb
Successfully built 87cb10858edd
I0818 20:53:33.853275       1 docker.go:118] Pushing image docker.io/veermuchandi/mytime:latest ...
I0818 20:53:40.857715       1 docker.go:122] Push successful
```

Note that an image with tag `docker.io/veermuchandi/mytime:latest` is built and pushed into docker registry.


Now I can pull my image from Docker Registry (DockerHub in this case)
```
$ docker pull veermuchandi/mytime:latest
Trying to pull repository docker.io/veermuchandi/mytime ... 
latest: Pulling from docker.io/veermuchandi/mytime
8ddc19f16526: Pull complete 
914f5c514f33: Pull complete 
Digest: sha256:644a16ee433086d85797cebb29c0f9ff880d952ef57d55460cc2ff1b0eec742f
Status: Downloaded newer image for docker.io/veermuchandi/mytime:latest
```

and I can verify the docker image by running it

```
$ docker run -p 8080:8080 -d veermuchandi/mytime
$ curl http://localhost:8080
Fri Aug 19 17:01:19 UTC 2016
```

####Summary
We have learnt how to create an application image using the build process in OpenShift and configure the build configuration to push the resultant image to an external Docker Registry. Enjoy!!










 







 
 


