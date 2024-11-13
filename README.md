# setup-ucacher

This action installs Earthly `ucacher`.

## Universal caching with `ucacher`
`ucacher` is a CLI tool that tracks which files are accessed by commands, skips unnecessary executions, and restores cached outputs when possible, resulting in significantly **reduced execution times**.

## Quick start
### Install and configure `ucacher`
Add ucacher to your GitHub Actions workflow:

```yaml
steps:
- uses: earthly/setup-ucacher@main
  ...
```
### Cache your commands
To cache a command, simply wrap it with `ucacher`:
```yaml
steps:
  ...
  - run: ucacher yarn test
```
- First Run: The command runs as usual, with a slight overhead to upload output files.
- Subsequent Runs: `ucacher` detects if the command’s output would remain unchanged (e.g., only **README.md** was modified but isn’t accessed by `yarn test`). If so, it skips execution and restores the cached output files instantly.

## How `ucacher` works
`ucacher` utilizes `ptrace` to monitor files read (inputs) and written (outputs) by a command. When the command completes successfully, it uploads output files to persistent storage (e.g., GitHub Actions cache) along with metadata about the build.

On subsequent runs, `ucacher` checks for matching initial conditions (input files, arguments, environment variables, system architecture, etc.). If they match, it skips execution and restores the cached output files instead.

> Returning to the previous example where `README.md` was modified, `ucacher` recognizes that this file doesn't affect the command output because a previous successful execution with the same initial state (environment, input files, etc.) has already been cached. Since the command is deterministic, `ucacher` can confidently skip re-execution, since if `README.md` wasn't accessed during the previous run, it won’t be accessed this time either.

## Customizing caching with ignored files and environment variables
To improve caching performance, certain files and environment variables can be ignored so that changes to them won’t prevent cache hits.

### `.ucacherignore`
Place a `.ucacherignore` file in the root directory to define file patterns to ignore. Each line is a glob pattern that `ucacher` will use to filter out files.

```plaintext
.git/**/*
node_modules/**/*
/tmp/**/*
```

### `.ucacherignore.env`

Define environment variable patterns to ignore in `.ucacherignore.env`:

```plaintext
GITHUB_TOKEN
ACTIONS_RUNTIME_TOKEN
GITHUB_PATH
GITHUB_OUTPUT
GITHUB_ENV
```
> Complete example here: [.ucacherignore.env](https://github.com/earthly/react/blob/uc-demo/.ucacherignore.env)

This prevents environment changes that don’t impact command output from invalidating the cache.

### Comparison with actions/cache

[actions/cache](https://github.com/actions/cache) is the official GitHub Action designed to cache dependencies and build outputs in GitHub Actions workflows. Here it is how they compare to each other:

| Feature        | `actions/cache`        | `ucacher`                        |
|----------------|----------------------|------------------------------------|
| API	          | File-based           | Command-based                      |
| Skipping       | Manual via cache-hit | Automatic                          |
| Output restore | Not supported        | Automatic                          |
| Persistence    | GitHub cache         | GitHub cache, S3, Minio (upcoming) |

- `actions/cache`: Primarily for dependency caching, requiring explicit file lists. Often used to avoid downloading dependencies multiple times, saving network and compute resources.
- `ucacher`: Like `actions/cache`, it can cache dependencies (for example: `ucacher npm install`), but it can also be used to cache the execution of any other command and restore its outputs, so successive commands depending on these files are not affected.

## Best practices
1. **Split commands**: Divide large commands (e.g., `yarn test`) into smaller shards to maximize cache hit chances when only a subset of source files change.
2. **Disable daemon modes**: If tools support daemon modes, consider disabling them for `ucacher` to track inputs and outputs accurately.
3. **Use ignore files**: For maximum cache efficiency, use `.ucacherignore` and `.ucacherignore.env` to exclude irrelevant files and variables from tracking.

## Limitations
 - **OS support**: Only Linux runners supported.
 - **Architecture support**: Currently only supports `amd64`.
 - **Inter-Process communication**: External processes not part of the spawned command tree (like daemon or client-server setups) won’t have their file interactions tracked.

## Licensing and privacy
 - **License**: `ucacher` is free to use; it may become open-source in the future.
 - **Data collection**: `ucacher` collects limited runtime metrics for maintenance, such as repository and organization names, commit author, and `ucacher`-specific errors. Source code or environment data is never collected.

## Who is using `ucacher`?
(in chronological order)

 1. First spot available!
 2. ...

> Using `ucacher`? Send a PR adding your repo at the end of the previous list

## Feedback
Please give us feedback to keep improving it by opening an issue in this repo. In particular, we'd like to know:
- Is it useful for you? Share some metrics with us
- Is performance not as good as you’d like?
- Is `ucacher` missing some file accesses?
- Which feature would you like us to work next?
- ...

Thanks!
