# Kpack with Local Registry and Kind

## Kind setup

Here's a script that includes configuration for insecure registry. The script also starts the registry in docker and exposes it on port 5000 locally. That works for building on the host and pushing and deploying in cluster:

```
$ ./kind-setup.sh
```

The `kind` config contains `containerdConfigPatches` for `registry.local` as a hostname for an insecure registry:

```
kind: Cluster 
apiVersion: kind.x-k8s.io/v1alpha4
containerdConfigPatches: 
- |-
  [plugins."io.containerd.grpc.v1.cri".registry.mirrors."${reg_name}:${reg_port}"]
    endpoint = ["http://${reg_name}:${reg_port}"]
...
```

So kind can pull images from `registry.local`. The hostname resolution is handled by docker if you're inside a container and it knows that it's insecure because of containerd.

## Registry Host

The host `registry.local` is a special name that is treated as insecure by `kpack` (or rather the library that it uses). You need to be able to resolve the host from the `kpack` worker pods, and `kpack` is smart enough to use containerd to resolve the registry.

You might also want to be able to docker push to the local registry from the host - it's not necessary for `kpack` to work, but useful for other reasons. We can do that by hacking `/etc/hosts`:

```
$ docker inspect -f '{{.NetworkSettings.IPAddress}} registry.local' registry.local | (sudo tee -a /etc/hosts)
```

## Kpack

Then set up `kpack` per the [tutorial](https://github.com/pivotal/kpack/blob/master/docs/tutorial.md) (it's using release 0.2.2):

```
$ kubectl apply -f kpack
```

(do it twice if you get errors). Also set up a dummy secret for registry authentication:

```
$ kubectl create secret docker-registry tutorial-registry-credentials --docker-username=user --docker-password=password --docker-server=registry.local:5000 --namespace default
```

Then build the petclinic image:

```
$ kubectl apply -f demo
```

The build should eventually succeed, like in the tutorial. If you don't use `.local` in the hostname it will fail because it will assume the registry is secure. If you don't hack `/etc/hosts` it will fail because it can't locate the registry host.

You can check the build by looking at the status of the pod:

```
$ kubectl get pods 
NAME                                     READY   STATUS     RESTARTS   AGE
tutorial-image-build-1-n6qzp-build-pod   0/1     Init:4/6   0          8m25s
```

It's quite likely it will get stuck on `Init:4/6` while it downloads all the build dependencies. That's the "build" container, so you can trace it with `kubectl`:

```
$ kubectl logs tutorial-image-build-1-n6qzp-build-pod build
...
[INFO] Downloading from spring-snapshots: https://repo.spring.io/snapshot/org/objenesis/objenesis-parent/2.6/objenesis-parent-2.6.pom
...
```

Eventually the build will finish and the image will be available:

```
$ kubectl get builds
NAME                           IMAGE                                                                                                        SUCCEEDED
tutorial-image-build-1-n6qzp   registry.local:5000/apps/petclinic@sha256:f307eb60627e8277ad4d448a1a9505aaf1ae5df10b7255ada8fb547167853e9c   True
```