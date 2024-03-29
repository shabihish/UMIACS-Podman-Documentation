# UMIACS Podman Documentation
## Overview

Podman is a tool for managing OCI containers and pods. While it is very similar to Docker in the sense that you can build images and run containers isolated containers, it is different from Docker in that it does not require a separate daemon to run containers, making it more lightweight and secure. Some key aspects of Podman include:

- It uses the same container and image formats as Docker (including the Docker image format), so images hosted on Docker Hub can be used. 

- Podman supports both unprivileged and privileged operations. Unprivileged mode allows users to run Podman without requiring root privileges. 

- Podman supports pods in addition to just individual containers. Pods allow grouping related containers that can share namespaces and networks.

- Storage for images, containers, and pods is local by default. There is no daemon and no dependency on a remote Docker host.




## Podman on the UMIACS Nexus Clusters

Podman is supported by UMIACS for Nexus Clusters by default. A documentation for this can be found here: 

https://wiki.umiacs.umd.edu/umiacs/index.php/Podman

Here are a few considerations to make regarding this:

- By default, Podman runs containers in a new user namespace to allow non-root users to manage containers. The rootless mode translates user IDs between the root and namespaced views of the system. For this to work, there has to be a specification of the `subuid` ranges for the requesting user in `/etc/subuid`.
- Because the containers are run in root-less mode and due to the use of `subuid` ranges for the translation of user IDs, when running a container, the container is given a false sense of root-level system access. This means that even though the container in question would not be granted system-level root access, it would have virtual root access over its isolated namespace.
- Container storage and image storage `graphroot` defaults to the local directory `/var/lib/containers`.


## Storage

As previously mentioned, the storage `graphroot` (where images are stored) in Podman defaults to the local directory `/var/lib/containers`. However, on Nexus clusters access to this directory is limited and attempts to pull images into or run containers from this directory will raise errors because of permission denials by default. The UMIACS wiki page mentioned above provides a section on how this issue can be solved, but their documentation also seems to lack a global view of how this parameter can or cannot be used for different purposes in different environments.

### Unsuccessful Attempts

First, they assign subdirectories of `/scratch0/` to the `graphroot` parameter in this documentation, which does not seem to be valid. Upon contacting the UMIACS staff members regarding an error I was getting when attempting to run containers from this directory on UMIACS SLURM nodes I was told that this `graphroot` directory is intended only for purposes of storing Podman images and/or containers and not running them:
```.
ERRO[0000] cannot find mappings for user username: No subuid ranges found for user "username" in /etc/subuid.
```
The error is obviously about no `subuid` ranges for my username being mentioned in the `/etc/subuid` file in the configuration of the SLURM node under consideration. I was notified of the following reason for this upon contacting the UMIACS helpdesk regarding the issue: "_The reason why we do not deploy subuids/gids on the compute nodes themselves is because the subuid/gid ranges that we deploy for each user account on submission nodes are not guaranteed to be unique to that user across all other nodes, resulting in potential collisions._"

Secondly, I tried using network storage partitions (`/fs/nexus-scratch/USERNAME/...`) as they provided much more storage space than local partitions, but these attempts were also all unsuccessful. Across all my attempts, I constantly got the following error, which was again seemingly because the network-based storage locations are configured not to allow namespace translations into local storage spaces due to security concerns:

```
ERRO[0001] While applying layer: ApplyLayer stdout:  stderr: setting up pivot dir: mkdir /fs/nexus-scratch/shabihi/containers/storage/vfs/dir/3a03f09d212915b240e9d216069aba5652ed4765c7e4b098c65e71860d47b8e1/.pivot_root3600685657: permission denied exit status 1
Error: creating build container: copying system image from manifest list: writing blob: adding layer with blob "sha256:527f5363b98e562da03d2e0bbf86fd7c34f487bffd9b27a3cf0a9afea2f0ee1f": ApplyLayer stdout:  stderr: setting up pivot dir: mkdir /fs/nexus-scratch/shabihi/containers/storage/vfs/dir/3a03f09d212915b240e9d216069aba5652ed4765c7e4b098c65e71860d47b8e1/.pivot_root3600685657: permission denied exit status 1
```

Upon contacting the UMIACS team members regarding this issue, I was informed that the wiki page for Podman was changed to underline the constraint that Podman containers can only be run on **local** storage partitions. 


### Final Approach

Finally, I was able to successfully pull, build, and commit to Podman images as well as run Podman containers with full "virtual" root access with the following two directories used as the `graphroot`, both of which reside on the local storage:
- `/var/tmp/USERNAME/containers/storage/`
- `/nfshomes/USERNAME/.local/share/containers/storage`

I was also able to build a full `Dockerfile` and run containers based on it in local mode.

While this approach has limitations due to the limited storage space and computational power provided while running containers in local mode, I was also informed that one can use Podman to build `Dockerfile`'s including specifications of the required environment and use **Apptainer** to run them with storage configurations on the network partitions **without run-time privilege elevations** according to this [documentation](https://wiki.umiacs.umd.edu/umiacs/index.php/Apptainer) and this [example](https://gitlab.umiacs.umd.edu/derek/gpudocker). However, I have yet had no experiments with this approach to verify whether it works as expected or not.
