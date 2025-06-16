---
reviewers:
- eparis
- pmorie
title: Configuring Redis using a ConfigMap
content_type: tutorial
weight: 30
---

<!-- overview -->


In this tutorial, you'll learn how to configure a Redis {{< glossary_tooltip text="Pod" term_id="pod" >}} using a {{< glossary_tooltip text="ConfigMap" term_id="configmap" >}} by following the steps from a real-world example.

This tutorial builds upon the [Configure a Pod to Use a ConfigMap](/docs/tasks/configure-pod-container/configure-pod-configmap/) task. 



## {{% heading "objectives" %}}


* Create a ConfigMap with Redis configuration values
* Create a Redis Pod that mounts and uses the created ConfigMap
* Verify that the configuration was correctly applied



## {{% heading "prerequisites" %}}


* You should understand how to [Configure a Pod to Use a ConfigMap](/docs/tasks/configure-pod-container/configure-pod-configmap/).
* You must be using `kubectl` 1.14 and above. 
  * {{< version-check >}}
* {{< include "task-tutorial-prereqs.md" >}} 




<!-- lessoncontent -->


## Real-World Example

In this real-world example, you'll configure a Redis cache using data stored in a ConfigMap. This exercise includes the following steps:

1. Create and apply a ConfigMap with a Redis pod manifest
2. Verify the Pod and ConfigMap configuration
3. Add configuration values to your ConfigMap
4. Delete and recreate the Pod

Once you've completed the tutorial, you can clean up your work by deleting the resources created throughout this exercise.

### Step 1: Create and apply a ConfigMap with a Redis pod manifest

1. Create a ConfigMap with an empty configuration block. Name it `example-redis-config`.

```shell
cat <<EOF >./example-redis-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: example-redis-config
data:
  redis-config: ""
EOF
```

2. Apply the ConfigMap you just created along with a Redis pod manifest.

```shell
kubectl apply -f example-redis-config.yaml
kubectl apply -f https://raw.githubusercontent.com/kubernetes/website/main/content/en/examples/pods/config/redis-pod.yaml
```

3. Examine the contents of the Redis pod manifest. You should see the following:

{{% code_sample file="pods/config/redis-pod.yaml" %}}

**Key components to note:**

* A volume named `config` is created by `spec.volumes[1]`
* The `key` and `path` under `spec.volumes[1].configMap.items[0]` exposes the `redis-config` key from the `example-redis-config` ConfigMap as a file named `redis.conf` on the `config` volume
* The `config` volume is then mounted at `/redis-master` by `spec.containers[0].volumeMounts[1]`

This has the net effect of exposing the data in `data.redis-config` from the `example-redis-config` ConfigMap you created as `/redis-master/redis.conf` inside the Pod.

### Step 2: Verify the Pod and ConfigMap configuration

1. Examine the Redis Pod and ConfigMap objects.

```shell
kubectl get pod/redis configmap/example-redis-config 
```

You should see the following output:

```
NAME        READY   STATUS    RESTARTS   AGE
pod/redis   1/1     Running   0          8s

NAME                             DATA   AGE
configmap/example-redis-config   1      14s
```

2. Describe the ConfigMap to view its contents.

```shell
kubectl describe configmap/example-redis-config
```

Note that the `redis-config` key is empty since no value was provided when creating the object. 

```shell
Name:         example-redis-config
Namespace:    default
Labels:       <none>
Annotations:  <none>

Data
====
redis-config:
```

3. Use `kubectl exec` to enter the pod and run the `redis-cli` tool to check the current configuration.

```shell
kubectl exec -it redis -- redis-cli
```

4. Check `maxmemory`.

```shell
127.0.0.1:6379> CONFIG GET maxmemory
```

You should see the default value of 0.

```shell
1) "maxmemory"
2) "0"
```

5. Check `maxmemory-policy`.

```shell
127.0.0.1:6379> CONFIG GET maxmemory-policy
```

You should see the default value of `noeviction`.

```shell
1) "maxmemory-policy"
2) "noeviction"
```

### Step 3: Add configuration values to your ConfigMap

1. Add the following configuration values to the `example-redis-config` key in the ConfigMap: `maxmemory 2mb` and `maxmemory-policy allkeys-lru`.

**Example:**

{{% code_sample file="pods/config/example-redis-config.yaml" %}}

2. Apply the updated ConfigMap.

```shell
kubectl apply -f example-redis-config.yaml
```

3. Describe the ConfigMap to confirm it updated correctly.

```shell
kubectl describe configmap/example-redis-config
```

You should see the configuration values you just added:

```shell
Name:         example-redis-config
Namespace:    default
Labels:       <none>
Annotations:  <none>

Data
====
redis-config:
----
maxmemory 2mb
maxmemory-policy allkeys-lru
```

{{</* note */>}}
At this point, if you check the Redis Pod again using `redis-cli` via `kubectl exec` you'll see the configuration values have not changed. This is because the Pod needs to be restarted to grab updated values from associated ConfigMaps.
{{</* /note */>}}

#### Check the Pod configuration

For the sake of this tutorial, check the Pod configuration using `redis-cli` via `kubectl exec` and confirm the default values are still present.

```shell
kubectl exec -it redis -- redis-cli
```

Then, check `maxmemory`.

```shell
127.0.0.1:6379> CONFIG GET maxmemory
```

Note that it remains at the default value of 0.

```shell
1) "maxmemory"
2) "0"
```

Similarly, when you check `maxmemory-policy`:

```shell
127.0.0.1:6379> CONFIG GET maxmemory-policy
```

Note that it returns the `noeviction` default setting.

```shell
1) "maxmemory-policy"
2) "noeviction"
```

### Step 4: Delete and recreate the Pod

1. Delete and recreate the Pod.

```shell
kubectl delete pod redis
kubectl apply -f https://raw.githubusercontent.com/kubernetes/website/main/content/en/examples/pods/config/redis-pod.yaml
```

2. Re-check the configuration values one last time.

```shell
kubectl exec -it redis -- redis-cli
```

3. Check `maxmemory`.

```shell
127.0.0.1:6379> CONFIG GET maxmemory
```

You should now see the updated value of 2097152.

```shell
1) "maxmemory"
2) "2097152"
```

4. Check `maxmemory-policy`.

```shell
127.0.0.1:6379> CONFIG GET maxmemory-policy
```

You should now see the updated value of `allkeys-lru`:

```shell
1) "maxmemory-policy"
2) "allkeys-lru"
```

Congratulations! You have successfully configured a Redis cache using data stored in a ConfigMap.

### Step 5: Clean up

You can clean up the work you did in this tutorial by deleting the resources you created.

```shell
kubectl delete pod/redis configmap/example-redis-config
```

## {{% heading "whatsnext" %}}


* Learn more about [ConfigMaps](/docs/tasks/configure-pod-container/configure-pod-configmap/).
* Take the [Updating configuration via a ConfigMap](/docs/tutorials/configuration/updating-configuration-via-a-configmap/) tutorial.
