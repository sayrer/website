+++
title = "Customizing Installation"
description = "Modify the Linkerd Installation"
+++

Instead of forking the Linkerd install and upgrade process,
[Kustomize](https://kustomize.io/) can be used to patch the output of `linkerd
install`  in a consistent way. This allows customization of the install to add
functionality specific to installations.

To get started, save the output of `install` to a YAML file. This will be the
base resource that Kustomize uses to patch and generate what is added to your
cluster.

```bash
linkerd install > linkerd.yaml
```

{{< note >}}
When upgrading, make sure you populate this file with the content from `linkerd
upgrade`. Using the latest `kustomize` releases, it would be possible to
automate this with an [exec
plugin](https://github.com/kubernetes-sigs/kustomize/tree/master/docs/plugins#exec-plugins).
{{< /note >}}

Next, create a `kustomization.yaml` file. This file will contain the
instructions for Kustomze listing the base resources and the transformations to
do on those resources. Right now, this looks pretty empty:

```yaml
resources:
- linkerd.yaml
```

Now, let's look at how to do some example customizations.

{{< note >}}
Kustomize allows as many patches, transforms and generators as you'd like. These
examples show modifications one at a time but it is possible to do as many as
required in a single `kustomization.yaml` file.
{{< /note >}}

## Add PriorityClass

There are a couple components in the control plane that can benefit from being
associated with a critical `PriorityClass`. While this configuration isn't
currently supported as a flag to `linkerd install`, it is not hard to add by
using Kustomize.

First, create a file named `priority-class.yaml` that will contain the overlay.
This overlay will explain what needs to be modified.

```yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: linkerd-identity
spec:
  template:
    spec:
      priorityClassName: system-cluster-critical
---
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: linkerd-controller
spec:
  template:
    spec:
      priorityClassName: system-cluster-critical
```

Then, add this as a strategic merge option to `kustomization.yaml`:

```yaml
resources:
- linkerd.yaml
patchesStrategicMerge:
- priorityclass.yaml
```

Applying this to your cluster requires taking the output of `kustomize build`
and piping it to `kubectl apply`. For example you can run:

```bash
kubectl kustomize build . | kubectl apply -f -
```

## Modify Grafana Configuration

Interested in enabling authentication for Grafana? It is possible to
modify the `ConfigMap` as a one off to do this. Unfortunately, the changes will
end up being reverted every time `linkerd upgrade` happens. Instead, create a
file named `grafana.yaml` and add your modifications:

```yaml
kind: ConfigMap
apiVersion: v1
metadata:
  name: linkerd-grafana-config
data:
  grafana.ini: |-
    instance_name = linkerd-grafana

    [server]
    root_url = %(protocol)s://%(domain)s:/grafana/

    [analytics]
    check_for_updates = false
```

Then, add this as a strategic merge option to `kustomization.yaml`:

```yaml
resources:
- linkerd.yaml
patchesStrategicMerge:
- grafana.yaml
```

Finally, apply this to your cluster by generating YAML with `kustomize build`
and piping the output to `kubectl apply`.

```bash
kubectl kustomize build . | kubectl apply -f -
```
