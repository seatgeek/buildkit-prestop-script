# buildkit-prestop-script

A simple [PreStop] script for [BuildKit] pods running in Kubernetes.

## Why?

[BuildKit does not (currently) support graceful shutdowns when it receives a `SIGTERM` signal][buildkit-4090].
This means that if a BuildKit pod is stopped while any builds are in progress, those builds will be terminated immediately,
resulting in failed builds.

We can mitigate this by providing a script that will wait for all ongoing builds to finish before allowing the pod to stop.

## Usage

1. Add [the `buildkit-prestop.sh` script](./buildkit-prestop.sh) to your BuildKit pod's container image. For example:

    ```Dockerfile
    FROM moby/buildkit:v0.15.1-rootless

    COPY buildkit-prestop.sh /usr/local/bin/buildkit-prestop.sh
    ```

2. Add the following to your BuildKit pod's spec:

    ```yaml
    lifecycle:
      preStop:
        exec:
          command: ["/usr/local/bin/buildkit-prestop.sh"]
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
