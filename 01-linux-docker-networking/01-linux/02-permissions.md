# Permissions

> Phase 1 — Linux, Docker & Networking › Linux

- [ ] Understood
- [ ] Practiced hands-on
- [ ] Can explain it without notes

## What it is

Linux permissions control who can read, modify, execute, or access files and directories. Every file and directory has an owning user, an owning group, and a set of permission bits that determine access.

Linux uses three permission categories:

- **User (`u`)**: The owner of the file.
- **Group (`g`)**: Users belonging to the file's owning group.
- **Others (`o`)**: Everyone else.

Three basic permission bits define the access granted to each category:

- **Read (`r`)**: Read file contents; for directories, list their entries.
- **Write (`w`)**: Modify file contents; for directories, create, delete, or rename entries, subject to other restrictions.
- **Execute (`x`)**: Run an executable file; for directories, traverse or access entries within them.

The primary commands are:

- `chmod`: Change file or directory permissions.
- `chown`: Change file ownership.
- `chgrp`: Change the owning group.
- `useradd`, `usermod`, `groupadd`: Manage users and groups.
- `umask`: Control default permissions assigned to newly created files and directories.

Permissions are essential for Linux administration, Docker containers, and Kubernetes workloads because applications need appropriate access to configuration files, mounted volumes, scripts, logs, and other resources without granting unnecessary privileges.

## Key points

- **Reading permission notation:** Run `ls -l` to inspect ownership and permissions. For example, `-rwxr-xr--` means the owner can read, write, and execute; the group can read and execute; and others can only read. The first character identifies the file type, such as `-` for a regular file or `d` for a directory.

- **Understanding octal permissions:** Each permission has a numeric value: read = 4, write = 2, and execute = 1. Add these values for each category to obtain the octal representation.

  | Permission | Calculation | Octal |
  |---|---|---:|
  | `rwx` | 4 + 2 + 1 | 7 |
  | `rw-` | 4 + 2 | 6 |
  | `r-x` | 4 + 1 | 5 |
  | `r--` | 4 | 4 |
  | `---` | 0 | 0 |

- **Common permission combinations:**
  - `755`: Owner has full permissions; group and others can read and execute.
  - `644`: Owner can read and write; group and others can read.
  - `700`: Only the owner has access.
  - `600`: Only the owner can read and write.
  - `775`: Owner and group have full permissions; others can read and execute.
  - `777`: Everyone has full permissions. Avoid using it as a general-purpose fix for access problems.

- **Symbolic versus numeric permissions:** `chmod u+x script.sh` adds execute permission for the owner. `chmod 750 script.sh` replaces the permission bits with owner `rwx`, group `r-x`, and no permissions for others.

- **File ownership:** `chown user:group file.txt` changes both the owner and group. Changing ownership generally requires root privileges or the relevant capabilities. `chgrp group file.txt` changes only the group, subject to ownership and privilege rules.

- **Users and groups:** Groups let administrators grant access to multiple users without managing each user's permissions individually. Use `id`, `groups`, `getent passwd`, and `getent group` to inspect account and group information.

- **Directory permissions behave differently from file permissions:** Read permission lets a user list directory entries; execute permission allows traversal; write permission allows directory entries to be created, removed, or renamed when the other required permissions and restrictions permit it. Accessing a file also requires suitable permissions on its parent directories.

- **The principle of least privilege:** Give users and applications only the permissions they need. Avoid running applications as root or granting broad write access merely to resolve a permission error.

- **Understanding `umask`:** The umask removes permission bits from the defaults used when new files and directories are created. A common umask of `022` typically produces files with `644` and directories with `755`, assuming conventional creation modes and no overriding default ACL.

- **Special permission bits:** Setuid, setgid, and the sticky bit provide additional behavior. For example, the sticky bit on `/tmp` prevents ordinary users from deleting or renaming other users' files in that directory. Do not enable special bits without understanding their security implications.

- **Permissions and access control lists (ACLs):** Traditional mode bits are not always the entire story. ACLs can grant or restrict access for specific users and groups. Inspect them with `getfacl` and modify them with `setfacl` where supported.

- **Permissions inside Docker:** A process inside a container has a user and group identity. Bind-mounted files retain relevant host filesystem ownership and permission behavior, so a container process may be unable to write to a mounted directory even when it can write elsewhere.

- **Permissions in Kubernetes:** `runAsUser`, `runAsGroup`, `runAsNonRoot`, and `fsGroup` can affect the identity and filesystem access of a container or Pod. Their exact effects depend on the volume type, storage driver, security context, and cluster configuration.

- **Ownership is not the same as permission:** `chown` changes who owns a file; `chmod` changes the permission bits. Use the appropriate command instead of changing both unnecessarily.

## Commands / manifests

```bash
# 1. Inspect the current user, groups, and permissions
whoami
id
groups
ls -l
ls -ld .
stat file.txt

# 2. Create practice files and directories
mkdir permissions-lab
cd permissions-lab
touch file.txt
mkdir data
printf '#!/bin/sh\necho "Hello from Linux"\n' > script.sh

# 3. Change permissions using numeric notation
chmod 644 file.txt
chmod 755 script.sh
chmod 700 data

# 4. Change permissions using symbolic notation
chmod u+x script.sh
chmod g-w file.txt
chmod o-r file.txt
chmod u=rw,g=r,o= file.txt

# 5. Inspect the results
ls -l
stat file.txt
./script.sh

# 6. Inspect ownership
id
ls -ln file.txt
getent passwd "$(whoami)"
getent group

# 7. Change group ownership
# Replace data-team with an existing group
sudo chgrp data-team file.txt

# 8. Change user and group ownership
# Replace appuser and data-team with existing accounts
sudo chown appuser:data-team file.txt

# 9. Inspect and change the umask
umask
umask 022
touch default-file.txt
mkdir default-directory
ls -ld default-file.txt default-directory

# 10. Inspect ACLs if the utilities and filesystem support them
getfacl file.txt
```

### User and group administration

```bash
# Create a group
sudo groupadd data-team

# Create a user with a home directory
sudo useradd -m appuser

# Add an existing user to a supplementary group
sudo usermod -aG data-team appuser

# Verify group membership
id appuser
getent group data-team

# Delete the practice user or group when no longer needed
sudo userdel -r appuser
sudo groupdel data-team
```

Use these commands only in a disposable Linux environment or where you have administrative authorization. Do not remove real accounts or groups as part of practice.

### Hands-on permission test

```bash
# Create a file that only its owner can read and write
touch secret.txt
chmod 600 secret.txt
ls -l secret.txt

# Remove the owner's read permission, then test access
chmod u-r secret.txt
cat secret.txt

# Restore the original permissions
chmod 600 secret.txt

# Create a directory that only the owner can access
mkdir private
chmod 700 private
ls -ld private

# Compare the results
stat secret.txt
stat private
```

Expected observations:

- `secret.txt` should initially display `-rw-------`.
- `private` should display `drwx------`.
- Reading `secret.txt` as another unprivileged user should fail unless another access mechanism grants permission.
- The commands that change permissions may succeed even if a later access test fails. Permission checks depend on the identity of the process performing the operation.

### Docker example

```dockerfile
FROM alpine:3.22

# Create a non-root application user
RUN addgroup -S app && adduser -S app -G app

WORKDIR /app
COPY --chown=app:app script.sh /app/script.sh

# Make the script executable
RUN chmod 755 /app/script.sh

USER app

CMD ["/app/script.sh"]
```

Useful Docker commands:

```bash
# Inspect the configured user and process inside an image
docker run --rm --entrypoint id IMAGE_NAME

# Inspect a container's filesystem ownership
docker exec CONTAINER_NAME ls -ln /app

# Inspect the user configured in image metadata
docker image inspect IMAGE_NAME --format '{{.Config.User}}'
```

Replace `IMAGE_NAME` and `CONTAINER_NAME` with actual image and container names. The Dockerfile example assumes `script.sh` exists and is compatible with Alpine Linux.

### Kubernetes example

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: permissions-demo
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 10001
    runAsGroup: 10001
    fsGroup: 10001
  containers:
    - name: app
      image: busybox:1.36
      command: ["sh", "-c", "id; ls -ld /tmp; sleep 3600"]
      securityContext:
        allowPrivilegeEscalation: false
```

Apply and inspect the Pod in a test cluster:

```bash
kubectl apply -f permissions-demo.yaml
kubectl get pod permissions-demo
kubectl exec permissions-demo -- id
kubectl exec permissions-demo -- ls -ld /tmp
kubectl delete pod permissions-demo
```

This example demonstrates process identity and a basic Pod security context. `fsGroup` is most relevant to supported mounted volumes; it does not automatically change the ownership of every directory in the container image. Image contents, volume types, and cluster policies can affect the final result.

## Common failures & fixes

- **`Permission denied` when running a script:** Check whether the file has execute permission and whether its parent directories can be traversed. Use `ls -l script.sh` and `chmod u+x script.sh` when appropriate. Also check the interpreter specified by the script's shebang.

- **`Permission denied` when reading a file:** Inspect the file's mode, owner, group, and parent directory permissions with `ls -l`, `namei -l /path/to/file`, and `id`. Grant the minimum required access rather than setting permissions to `777`.

- **A process cannot write to a directory:** Check directory write and execute permissions, ownership, ACLs, read-only mounts, and the identity of the process. In Docker or Kubernetes, compare the container's UID/GID with the mounted directory's ownership.

- **`Operation not permitted` while using `chown`:** The current user may lack the required privileges or capabilities. Verify the command's target and execution context. Containers can have restricted capabilities even when a process appears to run as root inside the container.

- **`chmod` succeeds but access is still denied:** Check parent directory traversal permissions, ACLs, read-only filesystems, mount options, SELinux or AppArmor policies, and whether the process is using a different UID/GID than expected.

- **A file is executable on the host but not inside a container:** Check the file mode, container user, mount configuration, interpreter availability, and whether the filesystem is mounted with `noexec`.

- **New files have unexpected permissions:** Inspect `umask` and any default ACLs. Remember that the umask removes permissions from the mode requested by the creating application; it does not simply assign one fixed mode to every file.

- **Group changes do not appear in an existing session:** A process may retain its previous group credentials. Start a new login session or use an appropriate group-switching command, then verify with `id`.

- **A Kubernetes container cannot write to a mounted volume:** Inspect the Pod's `securityContext`, volume type, storage driver behavior, and actual filesystem ownership. Use `runAsUser`, `runAsGroup`, or `fsGroup` only when appropriate for the workload and storage configuration.

- **Recursive permission changes cause unexpected effects:** Use `chmod -R` and `chown -R` carefully. Recursive operations can change executable files, sensitive files, symlinks' targets, or permissions that applications depend on. Inspect the target path before running them.

- **A container works as root but fails as a non-root user:** Identify the directories the application needs to write to, then assign suitable ownership and permissions during image construction or volume setup. Avoid switching back to root as the default workaround.

## References

- [Linux `chmod` manual](https://man7.org/linux/man-pages/man1/chmod.1.html)
- [Linux `chown` manual](https://www.man7.org/linux/man-pages/man1/chown.1.html)
- [Linux `umask` manual](https://man7.org/linux/man-pages/man2/umask.2.html)
- [Linux `getfacl` manual](https://man7.org/linux/man-pages/man1/getfacl.1.html)
- [Dockerfile reference](https://docs.docker.com/reference/dockerfile/)
- [Docker container run reference](https://docs.docker.com/reference/cli/docker/container/run/)
- [Kubernetes: Configure a Security Context for a Pod or Container](https://kubernetes.io/docs/tasks/configure-pod-container/security-context/)
