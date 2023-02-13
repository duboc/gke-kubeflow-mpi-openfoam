# Kubeflow and the MPI Operator on GKE

This repository is a fork from the OpenShift one and provides an example of using
[Kubeflow](https://www.kubeflow.org/) and its [MPI
operator](https://github.com/kubeflow/mpi-operator) on top of
[GKE](https://cloud.google.com/kubernetes-engine).
The examples specifically target running [OpenFOAM](https://openfoam.org/) CFD
simulations with CPUs and GPUs. It uses Google Cloud [Filestore](https://cloud.google.com/filestore)
for providing RWX storage.

## Quickstart (GKE)

[![Open in Cloud Shell](https://gstatic.com/cloudssh/images/open-btn.svg)](https://ssh.cloud.google.com/cloudshell/editor?cloudshell_git_repo=https%3A%2F%2Fgithub.com%2FGoogleCloudPlatform%2Fmicroservices-demo&shellonly=true&cloudshell_image=gcr.io/ds-artifacts-cloudshell/deploystack_custom_image)

1. **[Create a Google Cloud Platform project](https://cloud.google.com/resource-manager/docs/creating-managing-projects#creating_a_project)** or use an existing project. Set the `PROJECT_ID` environment variable and ensure the Google Kubernetes Engine and Cloud Operations APIs are enabled.

```
PROJECT_ID="<your-project-id>"
gcloud services enable container.googleapis.com --project ${PROJECT_ID}
```

2. **Clone this repository.**

```
git clone https://github.com/duboc/gke-kubeflow-mpi-openfoam
cd gke-kubeflow-mpi-openfoam
```

3. **Create a GKE cluster.**

- GKE Standard mode:

```
ZONE=us-central1-b
gcloud container clusters create gke-openfoam \
    --project=${PROJECT_ID} --zone=${ZONE} \
    --machine-type=e2-standard-2 --num-nodes=3
```


### Filestore

To-do

### Cluster Autoscaling
Whether or not you want to auto-scale your cluster is up to you. It is trivial


### Kubeflow MPI Operator
The Kubeflow MPI Operator needs to be installed. The installation process also
creates a `CustomResourceDefinition` for an `mpijob` object, which is how you
will define the MPI job that you want the cluster to run.

```bash
kubectl apply -f https://raw.githubusercontent.com/kubeflow/mpi-operator/v0.3.0/deploy/v2beta1/mpi-operator.yaml
```

This will create a namespace with all of the required elements to deploy the
operator. Wait for the pod for the MPI Operator to be deployed and ready before
continuing.

## MPI Job
The following sections detail getting the MPI data into the cluster and then
running the example MPI job. It is recommended that you deploy the following
assets into their own namespace. For this
example we will refer to the `cfd` namespace.

### CFD Project
Create a new namespace called `cfd`. 

```bash
kubectl create ns cfd
```


### Persistent Volume Claim
You will need some storage to attach the file manager and the CFD workers to. Be
sure to create the following file in the namespace you created:

    kubectl create -f filestore/filemanager-pvc.yaml -n cfd

Check the status of the PVC to make sure that it is successfully bound.

### Tiny File Manager
There are supporting manifests in the `manifests` folder for deploying a
PHP-based file manager program, [Tiny File
Manager](https://tinyfilemanager.github.io/), and a corresponding volume claim.
The `PersistentVolumeClaim` will grab a small amount of RWX storage, and you can
then upload the `damBreak` example from the `examples` folder in order to do
your pre-processing.

A [separate
repository](https://github.com/OpenShiftDemos/tinyfilemanager-php-ubi) has a
`Containerfile` which can be used to build the Tiny File Manager into a RHEL8
UBI-based Apache and PHP S2I image. Although Source-to-Image (S2I) was not used
to build the resulting container, the RHEL8 UBI Apache S2I image already has the
proper security modifications in order to easily be used in an OpenShift
environment. You can find more information about the [S2I image
here](https://github.com/sclorg/s2i-php-container). The image is also hosted on
Quay.io:
[https://quay.io/repository/openshiftdemos/tinyfilemanager-php-ubi](https://quay.io/repository/openshiftdemos/tinyfilemanager-php-ubi)

You can deploy the file manager with the following:

    kubectl create -f filestore/filemanager-assets.yaml -n cfd

This will also create a Service type LoadBalancer  so that you can access the file
manager outside the cluster. 

The default username and password for Tiny File Manager is used:

    admin / admin@123

* In the Tiny File Manager, click into the `storage` folder
* Click the `Upload` button at the top right.
* Drag the local `drambreak-conent\damBreak` folder (from this repository)
  into the file manager window to upload it recursively
  
That's it!

### OpenFOAM MPI Launcher/Worker Container Image

The MPI operator uses a concept of a Launcher pod (which is where `mpirun`
originates) and Worker pods, which are the targets of the `mpirun` command.
There is no reason that both the Launcher and the Worker cannot be the same
container image. This tutorial uses the same container image for both.

OpenFOAM's CFD analysis process involves several steps. You can actually perform
some of the pre- and post-processing steps inside a running container using
`rsh` or `exec`. However, you can also include the steps in your pre- and
post-processing in a script, and then mount that script into the Launcher pod
and make that script be the entry execution point of the Launcher. We have
staged precisely that scenario for you in the subsequent steps.

### OpenFOAM MPI Script ConfigMap
An easy way to mount files into a container in Kubernetes is using a
`ConfigMap`. The sequence of steps in performing the _damBreak_ MPI job is
contained in a bash script in the `dambreak-configmap.yaml` file. Go ahead and
create this `ConfigMap`:

    kubectl create -f dambreak-content/dambreak-configmap.yaml -n cfd

### OpenFOAM MPI Job
At this point you are ready to run your OpenFOAM MPI job. Take a look at the
`mpijob-dambreak-example.yaml` manifest to see the structure of an `mpijob`. The
MPI Operator repository provides more details. To paraphrase, our `mpijob` has 4
worker Replicas, and our `mpirun` command specifies 4 processors. Both the
Launcher and the Worker are both using our OpenFOAM image, and the rest of the
arguments to `mpirun` come from the OpenFOAM project. Go ahead and create the
`mpijob` manifest now:

    kubectl create -f dambreak-content/mpijob-dambreak-example.yaml -n cfd


Eventually the launcher should get into `Running` state. If you check its logs,
you will see a lot of this:

```
smoothSolver:  Solving for alpha.water, Initial residual = 0.00193684, Final residual = 3.40587e-09, No Iterations 3
Phase-1 volume fraction = 0.124752  Min(alpha.water) = -2.76751e-09  Max(alpha.water) = 1
MULES: Correcting alpha.water
MULES: Correcting alpha.water
Phase-1 volume fraction = 0.124752  Min(alpha.water) = -2.76751e-09  Max(alpha.water) = 1
DICPCG:  Solving for p_rgh, Initial residual = 0.0343767, Final residual = 0.0016133, No Iterations 5
time step continuity errors : sum local = 0.000480806, global = -4.75907e-08, cumulative = 7.68955e-05
DICPCG:  Solving for p_rgh, Initial residual = 0.00181566, Final residual = 8.38192e-05, No Iterations 33
time step continuity errors : sum local = 2.42908e-05, global = 3.04705e-06, cumulative = 7.99426e-05
DICPCG:  Solving for p_rgh, Initial residual = 0.000251337, Final residual = 8.59604e-08, No Iterations 109
time step continuity errors : sum local = 2.49879e-08, global = -9.43014e-10, cumulative = 7.99416e-05
ExecutionTime = 22.2 s  ClockTime = 23 s

Courant Number mean: 0.076648 max: 1.01265
Interface Courant Number mean: 0.00417282 max: 0.936663
deltaT = 0.0010575
Time = 0.11616
```


## Slots per Worker
In our example we used a replica count of 2 on the Workers, but specified `-np
4` for `mpirun`. How does this work? In the MPI job we have specified
`slotsPerWorker: 2` which causes the MPI operator to configure the MPI `hosts`
file to specify that each worker has 2 _slots_, or processors. The MPI job
further includes a limit/request for 2 CPUs for each Worker pod. If you were to
`rsh` or `exec sh` into one of the worker pods and execute `top`, you would see
that two cores are being used:

```
...
    152 openfoam  20   0  312992  98708  80856 R  96.7   0.2   3:09.63 interFoam                                                                                                                                                              
    151 openfoam  20   0  313084  98752  80780 R  96.3   0.2   3:08.58 interFoam      
...
```

Depending on the nature of your environment, you may wish to run more
`slotsPerWorker` in order to reduce the total number of Pods that get scheduled
by the MPI operator. There are varying support limits for the number of
pods-per-node depending on your Kubernetes distribution.

Also, attempting to schedule very large numbers of pods simultaneously can
result in system instability. On the flip side, trying to fit larger Pods that
require more cores can also be challenging if your MPI job is running in an
otherwise busy Kubernetes cluster. Lastly, each Pod is making a connection to
the storage system, which results in higher throughput (on the network) and more
disk access. Too many Pods will eventually decrease performance as the storage
subsystem starts to become a limiting factor. Finding a good balance of slots
versus Pod size versus total Pods will be dependent on your environment.