
# Manual steps to reproduce `permission denied` error when accessing Kubernetes secret volume or service account token

This repository contains a minimal example to reproduce the Kubernetes issue [#114461](https://github.com/kubernetes/kubernetes/issues/114461) and to verify the that the pull request [#114464](https://github.com/kubernetes/kubernetes/pull/114464) fixes the issue.

As described in the issue, the bug is a race condition that occurs with following conditions:

- Container runs as non-root with GID!=0.
- `defaultMode` removes read permission from other users, such as `0440`.
- fsGroup is used to set the group ownership to allow the process to read the files with group read permissions.

The problem applies to:
- secrets
- service account tokens
- configmaps

This example is known to work on Kind with Kubernetes 1.25.3 image `kindest/node:v1.25.3` but the issue is present in other Kubernetes versions too.


## Preparation

Create Kind cluster

```bash
kind create cluster --name permission-race-condition
```

Build the test app and upload it to the cluster:

```bash
docker build --tag watcher:latest docker/watcher/
kind load docker-image watcher:latest --name permission-race-condition
```

The test application follows the reproduction steps described in [the issue](https://github.com/kubernetes/kubernetes/issues/114461).
For details, see [`main.go`](docker/watcher/main.go).

## Reproducing the fault

### Failing to read a file on secret volume

Create a secret and a pod that runs the test application

```bash
kubectl apply -f manifests/secret.yaml
kubectl apply -f manifests/watcher.yaml
```

Note that the pod runs as non-root with GID!=0 (see [`Dockerfile`](docker/watcher/Dockerfile)).
The group ownership for files on the secret volume is set by `fsGroup: 10000` and read permissions are removed from other users by setting `defaultMode: 0440`, see [watcher.yaml](manifests/watcher.yaml).

The race condition is triggered by updating the secret.
Update the secret periodically by running:

```bash
while true; do kubectl create secret generic mysecret --from-file=password=/proc/sys/kernel/random/uuid --dry-run=client -o yaml | kubectl apply -f -; sleep 30; done
```

Kubelet will write the update on disk once in every 2 minutes.
Observe the test application logs when it gets notified of the update by `IN_MOVED_TO` inotify event

```bash
kubectl logs -f watcher
```

The event is sent when kubelet publishes the update by calling `rename("..data_tmp", "..data")`.
This makes the new file visible to the application, which then attempts to open it and print success/failure.
The test application is likely to fail with a `permission denied` error at every update.
It is faster to call `open()` than kubelet is to set the group read permissions after it had published the update:

```bash
2022/12/23 20:18:03 Starting watcher as uid: 405 gid: 100
2022/12/23 20:18:03 Watching paths: /secret/, /var/run/secrets/kubernetes.io/serviceaccount/
2022/12/23 20:21:01 Error : open /secret/password: permission denied
2022/12/23 20:21:01 Open: succeeded: 0, failed: 1
2022/12/23 20:22:03 Error : open /secret/password: permission denied
2022/12/23 20:22:03 Open: succeeded: 0, failed: 2
2022/12/23 20:23:12 Error : open /secret/password: permission denied
2022/12/23 20:23:12 Open: succeeded: 0, failed: 3
2022/12/23 20:24:39 Error : open /secret/password: permission denied
2022/12/23 20:24:39 Open: succeeded: 0, failed: 4
2022/12/23 20:26:07 Error : open /secret/password: permission denied
2022/12/23 20:26:07 Open: succeeded: 0, failed: 5
```

The reason for the error is that immediately after the update, the file is owned by group root for a short while, instead of the group `10000` set by `fsGroup`, as expected.

### Failing to read token on projected volume

Note that to reproduce this fault, the user must be set with `USER` in the [`Dockerfile`](docker/watcher/Dockerfile), instead of `RunAsUser` in the pod spec.
In this case group read permissions set with `fsGroup` will be used to read the token.
Otherwise the owner read permissions and user set with `RunAsUser` will be used.

Restart kubelet to trigger on-demand update of `/var/run/secrets/kubernetes.io/serviceaccount/token`:

```bash
docker exec permission-race-condition-control-plane systemctl restart kubelet
```

The side-effect will be that the connection to container will drop.
Re-execute to read the logs:

```bash
$ kubectl logs -f watcher
2022/12/23 20:49:40 Starting watcher as uid: 405 gid: 100
2022/12/23 20:49:40 Watching paths: /secret/, /var/run/secrets/kubernetes.io/serviceaccount/
2022/12/23 20:51:16 Error : open /var/run/secrets/kubernetes.io/serviceaccount/token: permission denied
2022/12/23 20:51:16 Open: succeeded: 0, failed: 1
```

The reason for the error is that the token will have mode `0600` and owner and group set to root for a short while after the update.
Later the mode will change to `0640` and group is set according to `fsGroup` allowing read to happen via group read permissions, as expected.

## Testing the proposed bug fix

At the time of writing there is proposed fix in PR [#114464](https://github.com/kubernetes/kubernetes/pull/114464).
To test the fix, fetch the code and recompile kubelet:

```bash
make WHAT=cmd/kubelet
```

Replace the kubelet binary in the cluster:

```bash
docker exec permission-race-condition-control-plane systemctl stop kubelet
docker cp ./_output/local/bin/linux/amd64/kubelet permission-race-condition-control-plane:/usr/bin/kubelet
docker exec permission-race-condition-control-plane systemctl start kubelet
```

Run above test again and observe that the error is no longer present.
Following output is from the secret volume read test:

```bash
2022/12/23 20:59:37 Starting watcher as uid: 405 gid: 100
2022/12/23 20:59:37 Watching paths: /secret/, /var/run/secrets/kubernetes.io/serviceaccount/
2022/12/23 21:01:04 Open: succeeded: 1, failed: 0
2022/12/23 21:02:29 Open: succeeded: 2, failed: 0
2022/12/23 21:03:33 Open: succeeded: 3, failed: 0
```

Following output is from the projected volume read test.
Restart kubelet several times to ensure that reading the token successfully was not just good luck.

```bash
2022/12/23 21:04:07 Starting watcher as uid: 405 gid: 100
2022/12/23 21:04:07 Watching paths: /secret/, /var/run/secrets/kubernetes.io/serviceaccount/
2022/12/23 21:04:17 Open: succeeded: 1, failed: 0
2022/12/23 21:04:26 Open: succeeded: 2, failed: 0
2022/12/23 21:05:15 Open: succeeded: 3, failed: 0
2022/12/23 21:05:30 Open: succeeded: 4, failed: 0
2022/12/23 21:05:40 Open: succeeded: 5, failed: 0
```
