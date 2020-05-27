# scalyr-loadgen
Configuration and instructions for running log ingestion load tests on Scalyr

## Overview

This repository contains Kubernetes manifest files for running log ingestion load tests on Scalyr.  The load generation
consists of pods writing HTTP access logs in Apache combined format.  Each pod consists of two main pods, one
generating the access logs and one running the Scalyr Agent to upload the logs to Scalyr.  A pod can be configured to 
generate up to 5MB/s of access logs.  The pods are grouped into Kubernetes depoloyments, one for each Scalyr account
being load tested (you can configured up to 10).  You can scale the overall load generation by increasing the number
of pods run for each deployment, as well as the number of deployments.

To aid verifying the load test results, a series of telemetry log files are uploaded to Scalyr in additional to the web
access logs.  These telemetry logs come from two sources: the HTTP access log generator and the Scalyr Agent.  The HTTP access
log generator emits a line for each second, recording the amount generated both in number of bytes and lines.  The Scalyr
Agent logs are collected from the agents running inside the log generation pods and are useful for troubleshooting any
issues.  The telemetry logs can be uploaded to a separate Scalyr account other than the ones being load tested.

## Running a load test

To run a load test, you will need a Kubernetes cluster running version v1.14 or higher that is not already uploading
logs to Scalyr.  We recommend not using a cluster already uploading logs to Scalyr to avoid accidentially uploading the
load generation logs twice.

Execute these steps to run your load test:

1.  Install `kustomize` if necessary

    These manifest files rely on `kustomize` for templating.  Rather than rely on the old version
    packaged with `kubectl`, our instructions assume you have the standalone `kustomize` binary
    installed.  We have tested the instructions with `kustomize` version 3.5.

    To install, please refer to the [kustomize install instructions](https://github.com/kubernetes-sigs/kustomize/blob/master/docs/INSTALL.md).

2.  Clone this repository

    ```
    git clone git@github.com:scalyr/scalyr-loadgen.git
    ```

3.  Create a new directory to hold the configuration for your load test

    ```
    cd scalyr-loadgen
    cp -r example my-load-test
    cd my-load-test
    ```

4.  Edit the `kustomization.yaml` file to specify how many load generation accounts to use, as well as
    name of your load-test cluster and which Scalyr server to send the logs (`www.scalyr.com` vs `eu.scalyr.com`)
  
    Here is an example that load tests two Scalyr accounts, sending the logs to `www.scalyr.com` with a cluster
    name of `loadgen-cluster` for the load generation and `loadgen-telemetry-cluster` for the telemetry logs.
  
    ```
      bases:
      - ../k8s/base/cluster
      - ../k8s/base/accounts/load-generator-01
      - ../k8s/base/accounts/load-generator-02

      generatorOptions:
        disableNameSuffixHash: true

      secretGenerator:
      - name: loadgen-api-keys
        namespace: scalyr-loadgen
        behavior: replace
        env: api-keys.txt

     configMapGenerator:
     - name: loadgen-telemetry-configmap
       namespace: scalyr-loadgen
       behavior: replace
       literals:
       - SCALYR_K8S_CLUSTER_NAME=loadgen-telemetry-cluster
     - name: load-generator-configmap
       namespace: scalyr-loadgen
       behavior: replace
       literals:
       - SCALYR_K8S_CLUSTER_NAME=loadgen-cluster
       - LOG_MBS_PER_POD=5
    ```

  5.  Configure the Scalyr API keys.
  
      To authorize your pods to upload logs to your Scalyr accounts, you must create a configuration
      file holding a write logs API key for each account.
      
      The file must be named `api-keys.txt` and be placed in the same directory as your `kustomization.yaml` file.
      It must have an entry called `telemetry-write-logs-key` holding an API key for the telemetry account, and one
      entry for each load generation account, with a key called `load-generator-XX-write-logs-key` (where XX is
      replaced with 01, 02, etc).
      
      To retrieve a write logs key for an account, you must log into the account on Scalyr and visit
      the [API keys page](https://app.scalyr.com/keys) for accounts on www.scalyr.com or [API keys page in EU]
      (https://app.scalyr.com/keys) for accounts on eu.scalyr.com.  The log access keys are listed in the first
      section.  We suggest you create a new write key for your load test by using the "Add Key" button.
      
      In our example above, our `api-keys.txt` file may look like:
      
      ```
      telemetry-write-logs-key=06XAnxxGLzA7V1/2KQuulqClQNaw12345-
      load-generator-01-write-logs-key=06XAnxxGLzA7V1/212345yTVwaex12345-
      load-generator-02-write-logs-key=0QMn0r9ge0Xcl/iQbyr6D70NM1234567-
      ```
      

  
  6.  Configure how much load to generate per account.
  
      The load generated per account is controlled by two factors:  the number of pods run per account
      and the amount of logs generated per pod.  We recommend you scale the load by only adjusting the
      number of pods per account, leaving the default value of 5MB/s of logs generated per pod.
      
      To increase the number of pods per account, you need to modify the number of replicas listed
      in the `k8s/base/peraccount/load-generator.yaml` manifest file.  The default is 1.
      
      Here is an example where we adjust the replicas to 5:
      
      ```
      apiVersion: apps/v1
      kind: Deployment
      metadata:
        name: load-generator
        namespace: scalyr-loadgen
      spec:
        replicas: 5
      ...
      ```
      
      With 5 replicas, we would generate 25MB/s per load account, giving us a total of 50MB/s of load
      across the two accounts.
  
  7.  Start the load test
  
      To start the load test, execute the following command in your load test directory:
      
      ```
      kustomize build . | kubectl apply -f -
      ```
      
      You should check to make sure all of the pods started successfully by running the
      following command
      
      ```
      kubectl -n scalyr-loadgen get pods
      ```
      
      If some of the pods have not been scheduled, you may wish see if your cluster has enough
      CPU and memory resources to run the pods.
      
      Once the load test is running, you can log into each of the load generation accounts
      and verify the are receiving the logs.
 
  8.  End the load test
  
      To stop the load test, execute the following command in your load test directory:
      
      ```
      kustomize build . | kubectl delete -f -
      ```

## Details

This section contains more detailed information about each of the components for the interested reader.

### Load generation Deployment

Each pod in the load generation deployment runs four containers:

1.  A forked version of [flog](https://github.com/mingrammer/flog)
2.  A Scalyr Agent in sidecar mode
3.  A tail sidecar echoing the flog telemetry log to its stdout
4.  A tail sidecar echoing the agent log to its stdout.

We have our [own fork of flog](https://github.com/scalyr/flog) that adds in features to control the number of log
bytes written as well as produce the telemetry file described above.

We run a Scalyr Agent instance in each pod (as opposed to running it as a DaemonSet) in order to scale the
CPU for the Scalyr Agent linearly with the amount of logs generated.  In this setup, we run the Scalyr
Agent's Kubernetes integration in side car mode, meaning it will only collect K8s logs from the containers
running it its same pod, as opposed to all containers on the same node.

The final two containers are used to send the two internal logs (the flog telemetry and Scalyr Agent logs)
to a container's stdout so that the Scalyr Agent's Kubernetes integration can collect them.

Note, we took care in naming the containers in the Load generation pods.  The container whose stdout
has the log generation output must contain the string `log-load`.  The other containers whose stdout
has the telemetry logs must contain the string `-telemetry`.  We do so that the load generation
Scalyr Agent can only collect the load logs, leaving the telemetry logs to the telemetry DaemonSet.

### Telemetry DaemonSet

The telemetry DaemonSet's main purpose is to collect all of the telemetry logs for all load generation pods
and send them to the telemetry Scalyr Agent.  This is accomplished by running a Scalyr Agent per node in
the cluster, configured to upload logs only from the `scalyr-loadgen` namespace and only for containers
with the string `-telemetry` in it.
