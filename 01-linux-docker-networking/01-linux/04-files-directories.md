# Files & directories

> Phase 1 — Linux, Docker & Networking › Linux

- [ ] Understood
- [ ] Practiced hands-on
- [ ] Can explain it without notes

## What it is

Linux represents information using files and organizes them into a hierarchical directory structure rooted at `/`. Files and directory operations are fundamental to administering Linux systems, managing application code, inspecting logs, handling configuration, and working with Docker images and Kubernetes workloads.

The most frequently used commands for file and directory management are:

- `ls`: List files and directories.
- `cp`: Copy files and directories.
- `mv`: Move or rename files and directories.
- `rm`: Remove files and directories.
- `find`: Locate files and directories by name, type, permissions, timestamps, and other attributes.
- `grep`: Search text for matching patterns.
- `pwd`: Display the current working directory.
- `cd`: Change the current working directory.
- `mkdir`: Create directories.
- `touch`: Create empty files or update timestamps.
- `cat`, `less`, `head`, `tail`: Read and inspect file contents.
- `file`: Identify a file's type.
- `stat`: Display file metadata.

These commands are particularly useful in data engineering. For example, you may use `find` to locate pipeline configuration files, `grep` to search application logs, `cp` to back up configuration, and `tail` to inspect a running application's log output.

## Key points

- **Linux filesystem hierarchy:** The root directory `/` is the starting point of the filesystem. Common directories include `/etc` for system configuration, `/var/log` for logs, `/home` for user directories, `/tmp` for temporary files, `/usr/bin` for many user commands, and `/proc` for process and kernel information.

- **Absolute versus relative paths:** An absolute path begins at `/`, such as `/home/app/config.yaml`. A relative path is interpreted from the current working directory, such as `config/config.yaml`.

- **Special path references:** `.` represents the current directory, `..` represents its parent, and `~` typically expands to the current user's home directory in a shell.

- **Listing files with `ls`:** `ls -l` displays detailed information, `ls -a` includes hidden entries, `ls -lh` presents file sizes in a human-readable format, and `ls -lt` sorts by modification time.

- **Hidden files:** Names beginning with `.` are hidden from ordinary `ls` output. Examples include `.bashrc`, `.gitignore`, and `.dockerignore`. Hidden does not mean secure or encrypted.

- **Copying with `cp`:** `cp source destination` copies a file. Use `cp -r` for directories and `cp -a` when you need to preserve attributes as far as possible. Be aware that copying can overwrite an existing destination file.

- **Moving and renaming with `mv`:** Moving a file changes its directory location; renaming changes its name. Both operations use `mv source destination`. When moving across filesystems, the operation may be implemented as a copy followed by deletion rather than a single atomic rename.

- **Deleting with `rm`:** `rm file.txt` deletes a file, while `rm -r directory` recursively removes a directory and its contents. Command-line deletion generally does not move items to a recycle bin. Check the target path carefully before using recursive deletion.

- **Creating directories:** `mkdir project` creates one directory. `mkdir -p project/data/raw` creates missing parent directories without failing when they already exist.

- **Searching with `find`:** `find /path -name '*.log'` searches for matching names. It can also filter by type, modification time, size, and permissions. Quote wildcard patterns so the shell does not expand them before `find` receives them.

- **Searching with `grep`:** `grep 'ERROR' app.log` prints lines containing a pattern. `grep -i` ignores case, `grep -n` displays line numbers, `grep -r` searches recursively, and `grep -E` supports extended regular expressions.

- **Combining commands:** Pipes (`|`) pass one command's output to another. For example, `find . -type f -name '*.log' | xargs grep -n 'ERROR'` searches matching files, although filenames containing unusual characters require more careful handling. Prefer `find ... -exec grep ... {} +` or null-delimited pipelines for robust scripts.

- **Wildcards and quoting:** `*` matches zero or more characters and `?` matches one character in shell pathname expansion. Quote patterns passed to tools when the tool, rather than the shell, should interpret them.

- **File metadata:** `stat file.txt` reports information such as file size, ownership, permissions, and timestamps. Modification time (`mtime`), status-change time (`ctime`), and access time (`atime`) describe different events.

- **Symbolic links:** A symbolic link points to another path. Create one using `ln -s target link_name`. If the target is moved or removed, the link may become broken.

- **Case sensitivity:** Linux filenames are generally case-sensitive. `Data.csv` and `data.csv` may be different files.

- **Permissions still apply:** File operations can fail because of ownership, permissions, parent directory access, ACLs, or read-only mounts. Use `ls -l`, `stat`, and `namei -l` to investigate access issues.

- **Disk space and inodes:** A filesystem may fail to create files even when it appears to have free space if it has exhausted its available inodes. Use `df -h` to inspect capacity and `df -i` to inspect inode usage.

- **File operations in containers:** Files created inside a container's writable layer may not persist after the container is removed. Use appropriate bind mounts or volumes when data must survive container replacement.

- **Logs and large files:** Avoid loading large log files entirely into memory or printing them all to the terminal. Use `less`, `head`, `tail`, and targeted `grep` searches to inspect only the relevant sections.

## Commands / manifests

```bash
# 1. Inspect the current location
pwd
ls
ls -lah
ls -lt

# 2. Navigate the filesystem
cd /tmp
cd ..
cd ~
cd -

# 3. Create files and directories
mkdir linux-lab
mkdir -p linux-lab/data/raw
touch linux-lab/notes.txt
printf 'INFO: Pipeline started\nERROR: Source unavailable\n' > linux-lab/app.log

# 4. Copy files
cp linux-lab/notes.txt linux-lab/notes-backup.txt
cp -r linux-lab linux-lab-copy

# Copy while preserving attributes where possible
cp -a linux-lab linux-lab-archive

# 5. Move and rename files
mv linux-lab/notes-backup.txt linux-lab/backup.txt
mv linux-lab/backup.txt linux-lab/data/backup.txt

# 6. Inspect file contents
cat linux-lab/app.log
less linux-lab/app.log
head -n 5 linux-lab/app.log
tail -n 5 linux-lab/app.log

# 7. Search filenames
find linux-lab -type f
find linux-lab -type f -name '*.log'
find linux-lab -type d
find linux-lab -type f -iname '*backup*'

# 8. Search file contents
grep 'ERROR' linux-lab/app.log
grep -in 'error' linux-lab/app.log
grep -nE 'ERROR|WARN' linux-lab/app.log

# 9. Search recursively
grep -rn 'Pipeline' linux-lab
grep -rl 'ERROR' linux-lab

# 10. Inspect metadata and file type
file linux-lab/app.log
stat linux-lab/app.log
du -sh linux-lab
df -h
df -i

# 11. Find recently modified files
find linux-lab -type f -mmin -60

# 12. Find large files above 100 MB
find /var/log -type f -size +100M 2>/dev/null

# 13. Create a symbolic link
ln -s app.log linux-lab/app-log-link
ls -l linux-lab/app-log-link
readlink linux-lab/app-log-link

# 14. Delete practice files only
rm linux-lab/data/backup.txt
rm -r linux-lab-copy
rm -r linux-lab-archive

# Verify remaining files
find linux-lab -maxdepth 3 -print
```

### Practical search patterns

```bash
# Find all Python files under the current directory
find . -type f -name '*.py'

# Find YAML manifests
find . -type f \( -name '*.yaml' -o -name '*.yml' \)

# Search for a configuration setting
grep -rn --include='*.yaml' 'replicas:' .

# Search logs while ignoring case
grep -in 'timeout' application.log

# Show the last 100 lines of a log
tail -n 100 application.log

# Follow new log entries as they are written
tail -f application.log

# Find files modified in the last 24 hours
find . -type f -mtime -1

# Find empty files
find . -type f -empty

# Count matching lines
grep -c 'ERROR' application.log
```

### Safer file operations

```bash
# Preview files before deleting them
find ./temporary-data -type f -name '*.tmp' -print

# Delete matching files only after verifying the list
find ./temporary-data -type f -name '*.tmp' -delete

# Search filenames safely, including names with spaces or newlines
find . -type f -name '*.log' -print0 |
  xargs -0 grep -nH 'ERROR'

# Alternative: let find pass groups of files to grep
find . -type f -name '*.log' \
  -exec grep -nH 'ERROR' {} +
```

Run destructive commands only against paths you have verified. Avoid experimenting with recursive deletion against `/`, `/home`, production directories, or mounted data volumes.

## Common failures & fixes

- **`No such file or directory`:** The path may be misspelled, relative to a different working directory, or affected by case sensitivity. Check `pwd`, `ls`, and the exact path.

- **`Permission denied`:** Inspect file permissions, ownership, and parent directory traversal permissions using `ls -l`, `stat`, or `namei -l`. Request the necessary access instead of reflexively using `sudo`.

- **`cp` fails to copy a directory:** Use `cp -r source destination` or `cp -a source destination` when directory contents and attributes need to be preserved.

- **`mv` unexpectedly replaces a destination:** Check whether the destination already exists. Use `mv -i` to request confirmation before overwriting an existing destination in interactive use.

- **`rm` refuses to remove a directory:** Use `rm -r directory` only when you intend to remove the directory and its contents. Verify the path first; recursive deletion is difficult to reverse.

- **`find` returns too many permission errors:** Restrict the search to relevant directories. Redirecting standard error with `2>/dev/null` hides errors but can also hide important information about directories that were not searched.

- **`find -name '*.log'` returns no results:** Quote the wildcard so the shell passes it to `find`. Without quotes, the shell may expand the pattern against files in the current directory before `find` runs.

- **`grep` returns no output:** The pattern may not match, capitalization may differ, or the target file may be empty. Try `grep -i`, verify the file with `head` or `cat`, and inspect the command's exit status.

- **`grep` reports `Binary file matches`:** The file may contain binary data or null bytes. Use `grep -a` only when treating the file as text is appropriate, or inspect it with a tool suited to its format.

- **`grep` misinterprets a pattern:** Characters such as `.`, `*`, `[`, and `^` have special meanings in regular expressions. Use `grep -F` for a literal fixed-string search.

- **A symbolic link is broken:** Use `ls -l` and `readlink` to inspect the link. Check whether the target exists and whether a relative target is being resolved from the link's location.

- **Disk is full despite apparently available space:** Run `df -h` and `df -i`. Capacity and inode exhaustion are separate failure modes. Also check whether deleted files are still held open by running processes.

- **A file disappears after a container restarts:** Determine whether it was written to the container's writable layer instead of a persistent volume or bind mount. Configure persistent storage for data that must survive container replacement.

- **A recursive search is unexpectedly slow:** Narrow the search root, filter by file type or name, and exclude large dependency or generated directories. Searching the entire filesystem can be expensive and may traverse mounted filesystems.

- **A command behaves differently with filenames containing spaces:** Quote variable expansions and use null-delimited processing with `find -print0` and `xargs -0` for robust shell scripts.

## References

- [GNU Coreutils manual](https://www.gnu.org/software/coreutils/manual/coreutils.html)
- [GNU Findutils manual](https://www.gnu.org/software/findutils/manual/html_mono/find.html)
- [GNU Grep manual](https://www.gnu.org/software/grep/manual/grep.html)
- [Linux `ls` manual](https://man7.org/linux/man-pages/man1/ls.1.html)
- [Linux `find` manual](https://man7.org/linux/man-pages/man1/find.1.html)
- [Linux `grep` manual](https://man7.org/linux/man-pages/man1/grep.1.html)
- [Docker: Storage](https://docs.docker.com/engine/storage/)
- [Kubernetes: Volumes](https://kubernetes.io/docs/concepts/storage/volumes/)
