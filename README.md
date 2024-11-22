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
  - run: ucacher <your_command>
```
- First Run: The command runs as usual, with a slight overhead to upload output files.
- Subsequent Runs: `ucacher` detects if the command’s output is unaffected by files changed. If so, it skips execution and restores the cached output files instantly.

## How `ucacher` works
`ucacher` utilizes `ptrace` to monitor files read (inputs) and written (outputs) by a command. When the command completes successfully, it uploads output files to persistent storage (e.g., GitHub Actions cache) along with some build metadata, in particular the hashes of all input files at the time they were read. 

On subsequent runs, `ucacher` checks for matching initial conditions (input files content, arguments, environment variables, system architecture, etc.). If they match, it skips execution and restores the cached output files instead.

## Example
Suppose the following steps on a `node-js` project, that run the unit tests of the client and server application components:
```yaml
steps:
  ...
- uses: earthly/setup-ucacher@main
  ...
  - run: ucacher yarn test --client
  - run: ucacher yarn test --server
```
and the workflow already run for that branch. Now, if a change in `server-impl.js` is pushed, then:
- `ucacher yarn test --client` would return the output from a compatible past execution right away, since that file is not used in the client tests. 
- `ucacher yarn test --server` would execute or return a cached result, depending on whether a compatible past execution is found (command already run for that file contents). 

The reason why `ucacher` determines that this file isn’t used is indirect, not by explicitly checking it, but by finding a previous execution whose input file contents that match the current filesystem state. This indicates that this file (or potentially other files) is irrelevant to this specific command instance.

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

## Inputs
### `github-token`
This token is used by `ucacher` to overwrite the Github Actions cache entry where the build metadata is stored. It requires **write** permissions for the `actions` scope in the GitHub REST API.

If omitted, the `GITHUB_TOKEN` of the job is used. This token should have enough permission by default. If that is not case, please refer to the [official GHA documentation](https://docs.github.com/en/actions/security-for-github-actions/security-guides/automatic-token-authentication#modifying-the-permissions-for-the-github_token) for guidance on modifying its permissions.  

## Best practices
1. **Split commands**: Divide large commands (e.g., `yarn test`) into smaller shards to maximize cache hit chances when only a subset of source files change.
2. **Disable daemon modes**: If tools support daemon modes, consider disabling them for `ucacher` to track inputs and outputs accurately.
3. **Use ignore files**: For maximum cache efficiency, use `.ucacherignore` and `.ucacherignore.env` to exclude irrelevant files and variables from tracking.

## Limitations
 - **OS support**: Only Linux runners supported.
 - **Architecture support**: Currently only supports `amd64`.
 - **Inter-Process communication**: External processes not under the tree spawned by the command (like daemon or client-server setups) won’t have their file interactions tracked.

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
