# buildkit-prestop-script

[![Software License](https://img.shields.io/badge/License-Apache--2.0-brightgreen.svg?style=flat-square)](LICENSE)
[![Build Status](https://img.shields.io/github/actions/workflow/status/seatgeek/buildkit-prestop-script/tests.yml?branch=main&style=flat-square)](https://github.com/seatgeek/buildkit-prestop-script/actions?query=workflow%3ATests+branch%3Amain)
[![Latest Release](https://img.shields.io/github/v/release/seatgeek/buildkit-prestop-script?style=flat-square)](https://github.com/seatgeek/buildkit-prestop-script/releases)
[![Script file size in bytes](https://img.shields.io/github/size/seatgeek/buildkit-prestop-script/buildkit-prestop.sh?style=flat-square)](./buildkit-prestop.sh)

A simple [PreStop] script for [BuildKit] that ensures all ongoing build complete before Kubernetes stops the pod.

## Why?

[BuildKit does not (currently) support graceful shutdowns when it receives a `SIGTERM` signal][buildkit-4090].
This means that if a BuildKit pod is stopped while any builds are in progress, those builds will be terminated immediately,
resulting in failed builds.

We can mitigate this by providing a script that will wait for all ongoing builds to finish before allowing the pod to stop.

## Usage

1. Add [the `buildkit-prestop.sh` script](./buildkit-prestop.sh) to your BuildKit pod's container image. For example:

    ```Dockerfile
    FROM moby/buildkit:v0.15.1-rootless

    ADD --chmod=755 https://raw.githubusercontent.com/seatgeek/buildkit-prestop-script/main/buildkit-prestop.sh /usr/local/bin/buildkit-prestop.sh
    ```

2. Add the following to your BuildKit pod's spec:

    ```yaml
    lifecycle:
      preStop:
        exec:
          command: ["/usr/local/bin/buildkit-prestop.sh"]
    ```

#### Alternative usage via configmap
Instead of extending the buildkit docker image, it's also possible to mount the preStop script inside the pod, via a configmap:
1. Add this to your BuildKit deployment manifest:
   ```
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      labels:
        app: buildkitd
      name: buildkitd
    spec:
      strategy:
        type: RollingUpdate
        rollingUpdate:
          maxUnavailable: 0
      template:
        spec:
          terminationGracePeriodSeconds: 1200
          containers:
            - name: buildkitd
              image: moby/buildkit:v0.16.0
              lifecycle:
                preStop:
                  exec:
                    command: ["/bin/sh", "/usr/local/bin/buildkit-prestop.sh"]
              volumeMounts:
                - name: prestop-script-volume
                  mountPath: /usr/local/bin/buildkit-prestop.sh
                  subPath: buildkit-prestop.sh
          volumes:
          - name: prestop-script-volume
            configMap:
              name: buildkit-prestop-script
              defaultMode: 0777  # Ensure the script is executable
   ```
2. Create the configmap with the script in this repo:
   ```
   apiVersion: v1
   kind: ConfigMap
   metadata:
     name: buildkit-prestop-script
   data:
     buildkit-prestop.sh: |
        #!/bin/bash
        ...
   ```

## How it works

BuildKit does not provide any built-in API for querying the count and status of ongoing builds, forcing us to rely on
external observations to determine when it is safe to stop the pod.

We found that the most reliable method is to check whether any clients are connected to the BuildKit daemon via `netstat`.
If we see any established connections then we assume that there is an ongoing build process.

## Configuration

The script contains a few configuration options that you can adjust to suit your needs. See the script for details.

## License

This project is licensed under the Apache License, Version 2.0. See [LICENSE](./LICENSE) for the full license text.

[PreStop]: https://kubernetes.io/docs/concepts/containers/container-lifecycle-hooks/#container-hooks
[BuildKit]: https://github.com/moby/buildkit
[buildkit-4090]: https://github.com/moby/buildkit/issues/4090
