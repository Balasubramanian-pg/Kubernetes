# Shell commands & pipes

> Phase 1 — Linux, Docker & Networking › Linux

- [ ] Understood
- [ ] Practiced hands-on
- [ ] Can explain it without notes

## What it is

Shell command chaining and redirection allow you to combine Linux commands, control their execution order, and direct their input and output. Instead of running every command manually, you can build repeatable command sequences for file processing, log analysis, troubleshooting, and automation.

The shell provides operators that determine how commands interact:

- `|` (pipe): Sends the standard output of one command to the standard input of another.
- `>` (output redirection): Writes standard output to a file, creating it if necessary and truncating it if it already exists.
- `>>` (append redirection): Appends standard output to a file instead of replacing its contents.
- `&&` (AND operator): Executes the next command only if the previous command succeeds.
- `;` (command separator): Executes commands sequentially, regardless of whether the previous command succeeded.
- `||` (OR operator): Executes the next command only if the previous command fails.
- `<` (input redirection): Supplies a file as standard input to a command.
- `2>` (error redirection): Sends standard error to a file.
- `2>&1` (file descriptor duplication): Redirects standard error to the same destination currently used by standard output.
- `|&` (Bash pipe operator): Sends both standard output and standard error through a pipe.

These operators are essential for building shell scripts, processing large datasets through command-line utilities, chaining diagnostic commands, and automating Linux, Docker, and Kubernetes workflows.

## Key points

- **Standard input, output, and error:** Linux processes typically use three standard file descriptors: `stdin` (0), `stdout` (1), and `stderr` (2). Pipes and redirections connect these streams to files or other processes.

- **How pipes work:** In `cat app.log | grep ERROR`, `cat` sends its output through a pipe, and `grep` reads that output as input. The output is not automatically saved to a file unless you redirect it.

- **Pipes process streams:** A pipe connects commands while they run. It is useful for filtering, transforming, and aggregating data without first saving every intermediate result to disk.

- **Redirection order matters:** `command > output.txt 2>&1` sends both standard output and standard error to `output.txt`. By contrast, `command 2>&1 > output.txt` first duplicates the original standard output destination for standard error, then redirects standard output to the file. Standard error may therefore continue going to the terminal.

- **Overwrite versus append:** `>` replaces the contents of an existing file, while `>>` preserves the existing contents and adds new output at the end. Both create the destination file if it does not exist and permissions allow it.

- **`&&` versus `;`:** `mkdir output && cd output` changes directories only if directory creation succeeds. `mkdir output; cd output` attempts the second command regardless of the first command's result.

- **The `||` operator:** Use `command1 || command2` to execute a fallback command if the first command fails. For example, `test -f config.yaml || echo "Configuration missing"` prints a message if the file does not exist.

- **Exit status:** A command normally returns exit status `0` for success and a nonzero value for failure. `$?` contains the exit status of the most recently completed foreground command, so inspect it immediately after the command you want to check.

- **Pipeline exit status:** By default, Bash generally uses the exit status of the last command in a pipeline. With `set -o pipefail`, a pipeline returns a failure status if a command in the pipeline fails, helping scripts detect upstream failures.

- **Pipelines and parallel execution:** Commands in a pipeline can execute concurrently as data flows between them. This can reduce intermediate-file usage, but pipeline processes may still consume CPU, memory, and I/O resources.

- **Using `tee`:** `tee` writes standard input to both standard output and a file. It is useful when you want to inspect command output and save it simultaneously. Use `tee -a` to append.

- **Command substitution:** `$(command)` captures a command's standard output and inserts it into another command. For example, `echo "Files: $(find . -type f | wc -l)"` prints the number of files found.

- **Quoting and expansion:** Double quotes preserve spaces within variable expansions while allowing variable and command substitution. Single quotes preserve the literal characters inside them. Use quotes to avoid accidental word splitting and wildcard expansion.

- **Redirection applies to commands:** In `grep ERROR app.log > errors.txt`, the matching lines are written to `errors.txt`, while diagnostic messages may still appear on standard error.

- **Background execution:** `command &` starts a command asynchronously in the shell. It does not automatically provide reliable long-term process supervision. Use a service manager or another suitable supervisor for production workloads.

- **Here-documents:** `<<EOF` supplies multiple lines of text to a command's standard input. A quoted delimiter, such as `<<'EOF'`, prevents shell variable and command expansion inside the document.

- **Shell scripting reliability:** Use explicit error handling, quote variables, validate input paths, and understand how pipelines propagate exit statuses. `set -e` can help in some scripts but has exceptions and should not replace deliberate error handling.

- **Data engineering applications:** Pipes can filter logs, count records, extract error messages, and transform command output. Redirection can capture execution logs, while `&&` and `||` control whether subsequent workflow steps run.

## Commands / manifests

```bash
# 1. Pipe output from one command into another
ls -lah | less
cat app.log | grep 'ERROR'

# More efficient: let grep read the file directly
grep 'ERROR' app.log

# 2. Overwrite a file with command output
ls -lah > directory-list.txt

# 3. Append output to a file
date >> execution.log
echo "Pipeline completed" >> execution.log

# 4. Chain commands with &&
mkdir -p output && cd output && touch result.txt

# 5. Execute commands sequentially with ;
echo "First command"
echo "Second command"

# 6. Execute a fallback command with ||
test -f config.yaml || echo "Configuration missing"

# 7. Redirect standard error
find /root -type f 2> errors.log

# 8. Redirect standard output and standard error to one file
./pipeline.sh > execution.log 2>&1

# 9. Discard standard output and standard error
./pipeline.sh > /dev/null 2>&1

# 10. Capture output and display it simultaneously
./pipeline.sh 2>&1 | tee execution.log

# 11. Append output while displaying it
./pipeline.sh 2>&1 | tee -a execution.log

# 12. Count matching log entries
grep -c 'ERROR' app.log

# 13. Count matching lines through a pipe
grep 'ERROR' app.log | wc -l

# 14. Use command substitution
echo "Current directory: $(pwd)"
echo "File count: $(find . -type f | wc -l)"

# 15. Check an exit status
test -f config.yaml
echo "$?"

# 16. Demonstrate pipeline exit status behavior in Bash
false | true
echo "$?"

set -o pipefail
false | true
echo "$?"
```

### Practical log-processing example

```bash
# Create a sample log
cat > application.log <<'EOF'
INFO Pipeline started
INFO Reading source data
ERROR Connection timeout
WARN Retrying request
ERROR Source unavailable
INFO Pipeline stopped
EOF

# Filter errors and save them
grep 'ERROR' application.log > errors.txt

# Append a timestamp to the processing log
date >> processing.log

# Count errors
grep -c 'ERROR' application.log

# Filter errors, count them, and save the count
grep 'ERROR' application.log | wc -l > error-count.txt

# Display errors while saving them
grep 'ERROR' application.log | tee errors.txt

# Run the next step only if validation succeeds
test -s errors.txt && echo "Error report generated"

# Clean up practice files
rm -f application.log errors.txt error-count.txt processing.log
```

### Reliable pipeline pattern for Bash scripts

```bash
#!/usr/bin/env bash

set -o pipefail

input_file="application.log"
output_file="errors.txt"

if [[ ! -f "$input_file" ]]; then
    echo "Input file not found: $input_file" >&2
    exit 1
fi

if grep -n 'ERROR' "$input_file" > "$output_file"; then
    echo "Error report created: $output_file"
else
    status=$?

    if [[ "$status" -eq 1 ]]; then
        echo "No matching errors found."
        : > "$output_file"
    else
        echo "Failed to search the input file." >&2
        exit "$status"
    fi
fi
```

This example distinguishes between `grep` finding no matches, which returns status `1`, and a genuine search error, which typically returns a different nonzero status. `set -o pipefail` is useful when pipelines are added, but it does not by itself handle every failure or validate application logic.

## Common failures & fixes

- **A command after `&&` does not execute:** The preceding command returned a nonzero exit status. Check its output and exit code before continuing.

- **A command after `;` executes despite a previous failure:** This is expected behavior. Use `&&` when the next operation depends on the previous one succeeding.

- **An output file loses its previous contents:** The command used `>` instead of `>>`. Use append redirection when you need to preserve earlier output.

- **Error messages still appear in the terminal:** `>` redirects standard output, not standard error. Use `2> errors.log` or `> execution.log 2>&1` as appropriate.

- **Standard error is missing from a log file:** Redirection order may be incorrect. In Bash, `command > output.log 2>&1` redirects both streams to the file.

- **A pipeline hides an earlier command's failure:** By default, the pipeline status is normally the status of its last command. Enable `set -o pipefail` in Bash scripts and check the pipeline's result.

- **A pipeline produces no output:** An upstream command may return no data, a filter may exclude every record, or output may be redirected elsewhere. Run each stage separately to locate the problem.

- **A script treats no search matches as a fatal error:** Tools such as `grep` return `1` when no lines match. Handle this as a separate case if no matches are an acceptable result.

- **A filename containing spaces is split into multiple arguments:** Quote variable expansions, for example `"$filename"`. Use null-delimited pipelines for filenames that may contain unusual characters.

- **A pipeline is unexpectedly slow:** Avoid unnecessary commands that merely copy data, reduce the amount of input early, and inspect the most expensive stages. For example, use `grep 'ERROR' app.log` instead of `cat app.log | grep 'ERROR'`.

- **A command succeeds but the overall workflow is wrong:** Exit status only reports what the command considers success. Validate expected outputs, row counts, files, and business rules separately.

- **A background command stops when a session closes:** Running a command with `&` alone does not guarantee that it will survive session termination. For long-running production jobs, use an appropriate process supervisor or workload orchestrator.

- **Sensitive data appears in logs:** Avoid redirecting unrestricted environment dumps, credentials, or personally identifiable information into shared log files. Limit what is captured and restrict file access.

- **A shell script stops unexpectedly after enabling `set -e`:** Its behavior depends on context, including conditionals and pipelines. Use explicit status checks for critical operations and test failure paths instead of relying on `set -e` alone.

## References

- [GNU Bash manual: Pipelines](https://www.gnu.org/software/bash/manual/html_node/Pipelines.html)
- [GNU Bash manual: Lists of Commands](https://www.gnu.org/software/bash/manual/html_node/Lists.html)
- [GNU Bash manual: Redirections](https://www.gnu.org/software/bash/manual/html_node/Redirections.html)
- [GNU Bash manual: Command Substitution](https://www.gnu.org/software/bash/manual/html_node/Command-Substitution.html)
- [GNU Coreutils manual](https://www.gnu.org/software/coreutils/manual/coreutils.html)
- [GNU Grep manual](https://www.gnu.org/software/grep/manual/grep.html)
- [ShellCheck: Static analysis tool for shell scripts](https://www.shellcheck.net/)
