# Red Hat Advanced Cluster Security Network Policy Generator Task

Allow users to generate network policies using [`roxctl netpol`](https://docs.openshift.com/acs/4.4/cli/command-reference/roxctl-netpol.html)

For the build-time network policy generation feature, `roxctl` CLI does not need to communicate with RHACS Central so you can use it in any development environment.

## Prerequisites

1. The build-time network policy generator recursively scans the directory you specify when you run the command. Therefore, before you run the command, you must already have service manifests, config maps, and workload manifests such as `Pod`, `Deployment`, `ReplicaSet`, `Job`, `DaemonSet`, and `StatefulSet` as YAML files in the specified directory.

2. Verify that you can apply these YAML files as-is using the `kubectl apply -f` command. The build-time network policy generator does not work with files that use Helm-style templating.

3. Verify that the service network addresses are not hardcoded. Every workload that needs to connect to a service must specify the service network address as a variable. You can specify this variable by using the workload’s resource environment variable or in a config map.
   
   [Example 1: using an environment variable](https://github.com/np-guard/cluster-topology-analyzer/blob/c79907c5af22acab35bb034ed0da622311fcf7e8/tests/k8s_guestbook/frontend-deployment.yaml#L25:L28)
   
   [Example 2: using a config map](https://github.com/np-guard/cluster-topology-analyzer/blob/c79907c5af22acab35bb034ed0da622311fcf7e8/tests/onlineboutique/kubernetes-manifests.yaml#L105:L109)
   
   [Example 3: using a config map](https://github.com/np-guard/cluster-topology-analyzer/blob/c79907c5af22acab35bb034ed0da622311fcf7e8/tests/onlineboutique/kubernetes-manifests.yaml#L269:L271)

5. Service network addresses must match the following official regular expression pattern:

   `(http(s)?://)?<svc>(.<ns>(.svc.cluster.local)?)?(:<portNum>)?`

   Where:
   
   `<svc>` is the service name.
   
   `<ns>` is the namespace where you defined the service.
   
   `<portNum>` is the exposed service port number.

Following are some examples that match the pattern:

`wordpress-mysql:3306`

`redis-follower.redis.svc.cluster.local:6379`

`redis-leader.redis`

`http://rating-service.`

## Install the Task

```bash
kubectl apply -f https://api.hub.tekton.dev/v1/resource/tekton/task/rhacs-netpol-generate/0.1/raw
```

## Parameters

- **`manifest_path`**: Directory path where the deployment manifests are stored. (required)
- **`netpol_out`**: Generated network policy output file name. Default: "netpol.yaml"

## Workspaces

- **manifest_dir**: An [Workspaces in Pipelines and PipelineRuns](https://github.com/tektoncd/pipeline/blob/main/docs/workspaces.md#workspaces-in-pipelines-and-pipelineruns)
which stores files across the pipeline runs.

## Usage

Check the [documentation](https://docs.openshift.com/acs/operating/manage-user-access/configure-short-lived-access.html#configure-short-lived-access_configure-short-lived-access)
to configure the trust with the OIDC token issuer. This
[example](../../rhacs-m2m-authenticate/0.1/samples/configure-m2m.md) describes
a possible RHACS machine-to-machine integration configuration.

The `roxctl` [documentation](https://docs.openshift.com/acs/cli/command-reference/roxctl.html)
describes the available commands and their options.

**Example task uses:**

Declarative configuration preparation:
```yaml
    - name: create-access-scope
      taskRef:
        name: rhacs-generic
        kind: Task
      params:
        - name: insecure-skip-tls-verify
          value: "true"
        - name: rox_endpoint
          value: $(params.rox_central_endpoint)
        - name: rox_image
          value: $(params.rox_image)
        - name: rox_arguments
          value:
            - declarative-config
            - create
            - access-scope
            - --name=testScope
            - --description=test access scope
            - --included=testCluster=stackrox
```

Deployment check:
```yaml
  tasks:
    - name: check-deployment
      taskRef:
        name: rhacs-generic
        kind: Task
      params:
        - name: insecure-skip-tls-verify
          value: "true"
        - name: rox_endpoint
          value: central.stackrox.svc:443
        - name: rox_arguments
          value:
            - deployment
            - check
            - --output=table
            - --file=/workspace/data/$(params.deployment)
      workspaces:
        - name: data
          workspace: shared-workspace
```

Image scan:
```yaml
  tasks:
    - name: scan-image
      taskRef:
        name: rhacs-generic
        kind: Task
      params:
        - name: insecure-skip-tls-verify
          value: "true"
        - name: rox_endpoint
          value: central.stackrox.svc:443
        - name: rox_arguments
          value:
            - image
            - scan
            - --output=table
            - --image=$(params.IMAGE)@$(tasks.build-image.results.IMAGE_DIGEST)
      runAfter:
        - build-image

```

**Samples:**

* [pipeline.yaml](samples/pipeline.yaml) demonstrates use in a pipeline.
* [pipelinerun.yaml](samples/pipelinerun.yaml) demonstrates use
in a pipelinerun.

# Known Issues

