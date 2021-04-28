# Kpack with Local Registry and Kind

Kind setup includes configuration for insecure registry:

```
$ ./kind-setup.sh
```

includes `containerdConfigPatches` for `registry.local` as a hostname for an insecure registry. Also starts the registry in docker and exposes it on port 5000 locally. That works for building on the host and pushing and deploying in cluster.

The host `registry.local` is a special name that is treated as insecure by `kpack` (or rather the library that it uses). You need to be able to resolve the host from the `kpack` worker pods. So we can do that by hacking `/etc/hosts`:

```
$ docker inspect -f '{{.NetworkSettings.IPAddress}} registry.local' registry.local | (sudo tee -a /etc/hosts)
```

Then set up `kpack` per the [tutorial](https://github.com/pivotal/kpack/blob/master/docs/tutorial.md) (it's using release 0.2.2):

```
$ kubectl apply -f kpack
```

and build the petclinic

```
$ kubectl apply -f demo
```

The build should eventually succeed, like in the tutorial. If you don't use `.local` in the hostname it will fail because it will assume the registry is secure. If you don't hack `/etc/hosts` it will fail because it can't locate the registry host.