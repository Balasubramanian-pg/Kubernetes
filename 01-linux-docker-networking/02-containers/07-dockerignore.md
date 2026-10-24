# .dockerignore

> Phase 1 — Linux, Docker & Networking › Containers

- [ ] Understood
- [ ] Practiced hands-on
- [ ] Can explain it without notes

## What it is

`.dockerignore` is a text file that tells Docker which files and directories to exclude from the **build context** before sending that context to the Docker builder.

The build context is the set of files available to Docker during an image build. When you run:

```bash
docker build -t my-app .
```

the final `.` identifies the current directory as the build context. Docker can access files within that context when processing instructions such as `COPY` and `ADD`, subject to the build environment and any additional build contexts.

Without a `.dockerignore` file, unnecessary files can be included in the build context, increasing the amount of data Docker must process or transfer. Sensitive files may also become accessible to build instructions, creating an avoidable security risk.

A `.dockerignore` file is normally placed at the root of the build context:

```text
my-app/
├── .dockerignore
├── Dockerfile
├── package.json
├── package-lock.json
├── src/
│   └── index.js
├── node_modules/
├── .git/
├── .env
└── README.md
```

Example `.dockerignore`:

```dockerignore
.git
node_modules
.env
.env.*
*.log
coverage/
dist/
```

When Docker builds this project, matching files are excluded from the build context. The application source and dependency manifests remain available to the Dockerfile.

**Important distinction:** `.dockerignore` does not delete files from your computer. It controls which files are included in the build context. It also does not automatically remove files that were already copied into an earlier image layer or prevent the application from reading files mounted into a container at runtime.

## Key points

- **Build context filtering**
  - Docker evaluates `.dockerignore` patterns against paths in the build context.
  - Ignored files are not available to ordinary `COPY` or `ADD` instructions that read from that context.
  - Excluding large directories such as `.git`, `node_modules`, test artifacts, and local caches can reduce context transfer and improve build efficiency.

- **File location**
  - The default location is the root of the build context, not necessarily the directory containing the command you ran.
  - If you build with `docker build -f docker/Dockerfile .`, the context is the current directory represented by `.`, so the root `.dockerignore` belongs in that directory.
  - Docker also supports Dockerfile-specific ignore files, such as `docker/build.Dockerfile.dockerignore`, which take precedence over the context-root `.dockerignore` for that Dockerfile.

- **Common ignore patterns**

  | Pattern | Purpose |
  |---|---|
  | `.git` | Excludes Git repository metadata |
  | `node_modules` | Excludes locally installed Node.js dependencies |
  | `.env` | Excludes a local environment file |
  | `.env.*` | Excludes files such as `.env.local` and `.env.production` |
  | `*.log` | Excludes matching log files |
  | `coverage/` | Excludes test coverage output |
  | `__pycache__/` | Excludes Python bytecode cache directories |
  | `*.py[cod]` | Excludes common Python compiled files |
  | `dist/` | Excludes a directory containing generated build output |

  Choose patterns based on the application. For example, ignoring `dist/` is appropriate when the image build generates its own production assets, but incorrect if the Dockerfile needs to copy a prebuilt `dist/` directory.

- **Pattern matching**
  - Patterns can match files and directories at different levels of the context.
  - `*` matches characters within a path component, while `**` can match across directory levels.
  - A leading `!` creates an exception that re-includes a path previously excluded by an earlier matching pattern.
  - The order of rules matters. The last matching rule determines whether a path is included or excluded.
  - A single `.` pattern is ignored for historical compatibility.

- **`.dockerignore` versus `.gitignore`**

  | `.dockerignore` | `.gitignore` |
  |---|---|
  | Controls files included in the Docker build context | Controls which untracked files Git normally ignores |
  | Affects Docker builds | Affects Git tracking behavior |
  | Does not stop Git from tracking a file | Does not automatically exclude a file from Docker builds |

  A file listed in `.gitignore` can still be included in a Docker build unless `.dockerignore` excludes it.

- **Security implications**
  - Exclude local credentials, private keys, `.env` files, cloud configuration, and other secrets that the build does not need.
  - Build context files can be exposed to Dockerfile instructions, including `COPY`, and may be processed by build tools.
  - Do not rely on `.dockerignore` as the only protection for secrets. Use BuildKit secret mounts for credentials required during a build.
  - Never copy a secret into an image and assume deleting it in a later Dockerfile instruction makes it safe. The secret may remain in an earlier image layer.

- **Effect on build caching**
  - Excluding irrelevant files can prevent unrelated changes from affecting build inputs and can reduce context processing.
  - Docker cache behavior depends on the particular instruction and its inputs. A change to a file copied by `COPY` can invalidate that instruction's cache, while a change to an excluded file does not affect that copy.
  - `.dockerignore` does not guarantee that every build becomes faster. The impact depends on context size, build steps, and the builder.

- **Build context versus runtime filesystem**
  - `.dockerignore` controls the build context, not files created inside a running container.
  - It does not prevent a container from accessing files explicitly mounted with `-v` or `--mount`.
  - It does not replace filesystem permissions, runtime security controls, or secret-management mechanisms.

## Commands / manifests

**1. Create a `.dockerignore` file**

```bash
cat > .dockerignore <<'EOF'
.git
.gitignore
node_modules/
.env
.env.*
*.log
coverage/
__pycache__/
*.py[cod]
EOF
```

This creates a starter file for a Node.js or Python project. Remove patterns that match files your Dockerfile actually needs.

**2. Build an image from the current directory**

```bash
docker build -t my-app:dev .
```

- `docker build` builds an image.
- `-t my-app:dev` assigns the image name and tag.
- `.` specifies the build context.
- Docker uses the applicable `.dockerignore` rules to filter the context.

**3. Use a minimal Dockerfile**

For a Node.js application:

```dockerfile
FROM node:22-alpine

WORKDIR /app

COPY package*.json ./
RUN npm ci --omit=dev

COPY src/ ./src/

CMD ["node", "src/index.js"]
```

Possible project structure:

```text
my-app/
├── .dockerignore
├── Dockerfile
├── package.json
├── package-lock.json
├── src/
│   └── index.js
├── node_modules/
├── .env
└── .git/
```

The Dockerfile copies the dependency manifests and source code, but does not need the local `node_modules`, `.git`, or `.env` files. The `.dockerignore` rules help keep those files out of the build context.

This example assumes `package-lock.json` exists and the application does not need other files at runtime. Adjust the Dockerfile and ignore patterns to match the application.

**4. Inspect the build output**

```bash
docker build --progress=plain -t my-app:dev .
```

Read the build output to understand which steps run and whether cached steps are reused. Depending on the builder, the output may report the build context size or transfer details.

To inspect the image size:

```bash
docker image ls my-app
```

A smaller image is useful, but image size alone does not prove that the build context is small or that sensitive files were excluded.

**5. Check which files Git tracks**

```bash
git status --short
git ls-files
```

These commands inspect Git's view of the repository. They do **not** show the effective Docker build context, and `.gitignore` rules are not a substitute for `.dockerignore`.

**6. Test whether a file is excluded**

Create a temporary file:

```bash
mkdir -p tmp-test
echo "test" > tmp-test/example.txt
```

Add this pattern to `.dockerignore`:

```dockerignore
tmp-test/
```

Then build an image with a Dockerfile that attempts to copy the directory:

```dockerfile
FROM alpine:3.21
COPY tmp-test/ /tmp-test/
```

Run:

```bash
docker build -t ignore-test .
```

The build should fail because the ignored directory is not available as a source for `COPY`. This is a useful way to verify that an ignore rule is taking effect. Remove the temporary test files when finished.

**7. Use exceptions carefully**

Example:

```dockerignore
*.log
!important.log
```

This excludes matching log files but allows `important.log` to be included when it matches the applicable path and pattern rules.

For nested directories, test the exact patterns you intend to use. Negation does not automatically make every parent directory available if the parent itself remains excluded.

**8. Keep secrets out of build instructions**

If a build needs a private package-registry token, avoid copying a `.env` file or token into the image. With BuildKit, use a secret mount instead.

Example Dockerfile instruction:

```dockerfile
# syntax=docker/dockerfile:1
FROM node:22-alpine

WORKDIR /app
COPY package*.json ./

RUN --mount=type=secret,id=npmrc,target=/root/.npmrc \
    npm ci --omit=dev

COPY src/ ./src/

CMD ["node", "src/index.js"]
```

Build command:

```bash
docker build \
  --secret id=npmrc,src="$HOME/.npmrc" \
  -t my-app:dev .
```

This example assumes the local `.npmrc` contains the required registry configuration and credentials. BuildKit makes the mounted secret available to that build instruction without adding the mounted file to the resulting image layer. Ensure the build process does not print or copy the secret elsewhere.

## Common failures & fixes

- **Problem: `COPY` fails with a message indicating that a source file does not exist or cannot be found.**
  - **Cause:** The file or directory may be excluded by `.dockerignore`.
  - **Fix:** Review the ignore patterns and remove or refine the matching rule if the file is required by the build. Keep unrelated files excluded.

- **Problem: The build context is still large.**
  - **Cause:** Large generated directories, dependency folders, datasets, or local artifacts may not be covered by the ignore rules.
  - **Fix:** Add targeted patterns such as `node_modules/`, `.git/`, `coverage/`, or a project-specific cache directory. Confirm that the Dockerfile does not need those files.

- **Problem: A file is still included even though it appears in `.gitignore`.**
  - **Cause:** Git ignore rules and Docker ignore rules are independent.
  - **Fix:** Add the relevant pattern to `.dockerignore` and rebuild the image.

- **Problem: A required directory is missing from the image.**
  - **Cause:** A broad ignore pattern may exclude application assets or build output required by `COPY`.
  - **Fix:** Narrow the pattern or use a carefully ordered negation rule. Verify the resulting image by running the application.

- **Problem: A secret appears in an image even though the file is now ignored.**
  - **Cause:** The secret may have been copied into an earlier image layer, included by another build path, or embedded in generated output.
  - **Fix:** Remove the secret from the Dockerfile and build inputs, use BuildKit secret mounts when needed, rebuild the image, and rotate any credential that may have been exposed. Deleting the file in a later layer is not sufficient.

- **Problem: A `.dockerignore` file seems to have no effect.**
  - **Cause:** It may be in the wrong directory, the build may use a different context, or a Dockerfile-specific ignore file may be taking precedence.
  - **Fix:** Check the final argument to `docker build`, locate the applicable ignore file, and verify the actual build output.

- **Problem: An exception rule does not restore a file.**
  - **Cause:** Pattern ordering or parent-directory exclusions may prevent the expected path from being included.
  - **Fix:** Review the patterns from top to bottom, place the exception after the rule it overrides, and test the exact nested path.

- **Problem: The image still contains unwanted files after updating `.dockerignore`.**
  - **Cause:** The Dockerfile may explicitly generate or copy those files through another source, or you may be inspecting an older image.
  - **Fix:** Inspect the Dockerfile and image contents, rebuild the image, and run the newly built tag. Remember that `.dockerignore` filters the build context, not files created during the build.

## References

- [Docker documentation: Build context](https://docs.docker.com/build/concepts/context/)
- [Docker documentation: `.dockerignore` files](https://docs.docker.com/build/concepts/context/#dockerignore-files)
- [Docker documentation: Build secrets](https://docs.docker.com/build/building/secrets/)
- [Docker documentation: Cache optimization](https://docs.docker.com/build/cache/optimize/)
- [Docker documentation: Dockerfile reference](https://docs.docker.com/reference/dockerfile/)
