
# setup-ucacher

This action installs Earthly `ucacher`.

> Read our new blog post about ucacher [here](https://earthly.dev/blog/ucacher/)

## Universal caching with `ucacher`
`ucacher` is a CLI tool that tracks which files are accessed by commands, skips unnecessary executions, and restores cached outputs when possible, resulting in significantly **reduced execution times**.

It brings: 
* **Faster feedback loops**
   * Skips redundant tasks by detecting if specific files or commands are unaffected by changes.
   * Reduces build times by caching outputs and avoiding unnecessary re-execution.
* **Simplified caching and skipping**
   * Automates caching and skipping with minimal setup.
   * Eliminates the need for developers to manage cache keys, paths, or complex conditions.
   * Removes the risk of human errors in configuration, reducing pipeline failures.
* **Enhanced precision** (traditional caching mechanisms often rely on broad keys, which can invalidate the entire cache for small, unrelated changes)
   * Tracks file dependencies at the command level using syscall instrumentation, enabling finer-grain caching.
   * Only re-executes commands or jobs affected by actual changes.
   * Prevents unnecessary invalidation and reruns, saving developer time and resources.
* **Resource efficiency**
   * Reduces resource consumption by skipping unaffected commands.
   * Minimizes cloud infrastructure costs, making CI/CD pipelines more sustainable.
   * Helps developers focus on coding rather than waiting for long-running jobs.
* **Better developer experience**
   * Offers out-of-the-box integration with tools like GitHub Actions, requiring minimal setup.
   * Automatically manages input/output tracking, making caching and skipping invisible to the developer.
   * Improves workflow reliability by ensuring that only the necessary jobs execute.
* **Scalability for complex workflows**
   * Handles complex scenarios like monorepos or task sharding with precision.
   * Automatically adjusts caching and skipping based on changes in specific files or commands.
   * Reduces unnecessary duplication in matrix jobs, saving time and resources across large teams.


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

| Feature        | `actions/cache`               | `ucacher`                          |
|----------------|-------------------------------|------------------------------------|
| Output restore | File-based                    | Command-based                      |
| Skipping       | Manual via `if` + `cache-hit` | Automatic                          |
| Keying         | Manual, broad scope           | Automatic, only relevant files     |
| Persistence    | GitHub cache                  | GitHub cache, S3, Minio (upcoming) |
| Scope          | Step                          | Command                            |

`ucacher` offers a more automated, precise, and efficient alternative to `actions/cache` in GitHub Actions by tracking file-level dependencies and automating caching and skipping at the command level. 

Unlike `actions/cache`, which requires manual setup with defined paths and keys, `ucacher` eliminates human errors and dynamically determines when commands should be skipped or re-executed based on actual file changes. 

This results in finer-grain caching, better resource optimization, and significant time savings, especially in complex workflows like monorepos or matrix builds, where broad cache keys in `actions/cache` often lead to inefficiencies like unnecessarily invalidating unrelated steps, redundant execution of unchanged tasks and manual configuration errors.

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
