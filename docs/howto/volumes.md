# Volume access

Volume support requires a small amount of work on your part.
The root directory where all the volumes can be found will be set to the `TELEPRESENCE_ROOT` environment variable in the shell run by `telepresence`.
You will then need to use that env variable as the root for volume paths you are opening.

Telepresence will attempt to gather the mount points that exist in the remote pod and list them in the `TELEPRESENCE_MOUNTS` environment variable, separated by `:` characters.
This allows automated discovery of remote volumes.

For example, all Kubernetes containers have a volume mounted at `/var/run/secrets` with the service account details.
Those files are accessible from Telepresence:

```console
$ telepresence
T: [...]
T: Setup complete. Launching your command.
@tel-testing|bash-3.2$ echo $TELEPRESENCE_ROOT
/tmp/tel-6cjjs3ba/fs
@tel-testing|bash-3.2$ echo $TELEPRESENCE_MOUNTS
/var/run/secrets/kubernetes.io/serviceaccount
@tel-testing|bash-3.2$ ls $TELEPRESENCE_ROOT/var/run/secrets/kubernetes.io/serviceaccount/
ca.crt          namespace       token
```

The files are available at a different path than they are on the actual Kubernetes environment.

One way to deal with that is to change your application's code slightly.
For example, let's say you have a volume that mounts a file called `/app/secrets`.
Normally you would just open it in your code like so:


```python
secret_file = open("/app/secrets")
```

To support volume proxying by Telepresence, you will need to change your code, for example:

```python
volume_root = "/"
if "TELEPRESENCE_ROOT" in os.environ:
    volume_root = os.environ["TELEPRESENCE_ROOT"]
secret_file = open(os.path.join(volume_root, "app/secrets"))
```

By falling back to `/` when the environment variable is not set your code will continue to work in its normal Kubernetes setting.

This approach is unavailable if you do not control the code that accesses the mounted filesystem, such as if you use a third-party library.
However, many such libraries offer configuration to work around this.
For example, the Java Kubernetes client library allows [configuration via environment variables](https://github.com/fabric8io/kubernetes-client#configuring-the-client).

To simplify this process, Telepresence optionally lets you set the value of `TELEPRESENCE_ROOT` to a known path using the `--mount` option.
Using a known value as the mount point (e.g., `--mount=/tmp/tel_root`) can you let you configure your tools and libraries once and rely on that configuration continuing to work across multiple Telepresence sessions.
When using the container method, the `--mount` option allows bind-mounting portions of the remote filesystem directly onto the usual paths.

For example, the `kubectl` command expects to find Kubernetes API service account credentials in `/var/run/secrets`.
This example shows `kubectl` successfully talking to the cluster while running in a local container:

```shell
$ telepresence --mount=/tmp/known --docker-run --rm -it -v=/tmp/known/var/run/secrets:/var/run/secrets lachlanevenson/k8s-kubectl version --short
Volumes are rooted at $TELEPRESENCE_ROOT. See https://telepresence.io/howto/volumes.html for details.

Client Version: v1.10.1
Server Version: v1.7.14-gke.1
```

Another way you can do this is by using the [proot](http://proot-me.github.io/) utility on Linux, which allows you to do fake bind mounts without being root.
For example, presuming you've installed `proot` (`apt install proot` on Ubuntu), in the following example we bind `$TELEPRESENCE_ROOT/var/run/secrets` to `/var/run/secrets`.
That means code doesn't need to be modified as the paths are in the expected location:

```console
@minikube|$ proot -b $TELEPRESENCE_ROOT/var/run/secrets/:/var/run/secrets bash
$ ls /var/run/secrets/kubernetes.io/serviceaccount/
ca.crt  namespace  token
```

Using the `TELEPRESENCE_MOUNTS` environment variable allows for automatic discovery and handling of mount points.
For example, the following Python code will create symlinks so that any mounted volumes appear in their normal locations:

```python
def telepresence_remote_mounts():
    mounts = os.environ.get('TELEPRESENCE_MOUNTS')
    if not mounts:
        return

    tel_root = os.environ.get('TELEPRESENCE_ROOT')

    for mount in mounts.split(':'):
        dir_name, link_name = os.path.split(mount)
        os.makedirs(dir_name, exist_ok=True)

        link_src = os.path.join(tel_root, mount[1:])
        os.symlink(link_src, mount)
```

### Docker volumes via [vieux/sshfs](https://github.com/vieux/docker-volume-sshfs)
It is possible to use [vieux/sshfs](https://github.com/vieux/docker-volume-sshfs) volume driver instead of using sshfs directly. This is needed to run telepresence cli inside a container (for instance to use it on windows). To utilize 'vieux/sshfs' use --docker-mount cli option. It is mutually exclusive with --mount. It needs to be set to absolute path and can be used only in conjunction with --method container.

When --docker-mount is specified instead of using sshfs a randomly named docker volume will be created using vieux/sshfs volume driver. The volume will be mounted in the user container at the mount point specified by --docker-mount. As a side benefit sudo is no longer required for container method.

One downside is that it is not possible to mount volume sub-directory to a container. This means that using TELEPRESENCE_ROOT is obligatory.