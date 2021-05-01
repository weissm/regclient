# regsync Documentation

- [Top Level Commands](#top-level-commands)
- [Configuration File](#configuration-file)

## Top Level Commands

```text
$ regsync --help
Utility for mirroring docker repositories
More details at https://github.com/regclient/regclient

Usage:
  regsync [command]

Available Commands:
  check       processes each sync command once but skip actual copy
  help        Help about any command
  once        processes each sync command once, ignoring cron schedule
  server      run the regsync server
  version     Show the version


Flags:
  -c, --config string        Config file
  -h, --help                 help for regsync
      --logopt stringArray   Log options
  -v, --verbosity string     Log level (debug, info, warn, error, fatal, panic) (default "info")
```

The `check` command is useful for reporting any stale images that need to be updated.

The `once` command can be placed in a cron or CI job to perform the synchronization immediately rather than following the schedule.

The `server` command is useful to run a background process that continuously updates the target repositories as the source changes.

`--logopt` currently accepts `json` to format all logs as json instead of text.
This is useful for parsing in external tools like Elastic/Splunk.

The `version` command will show details about the git commit and tag if available.

## Configuration File

The `regsync` configuration file is yaml formatted with the following layout:

```yaml
x-sched-a: &sched-a "15 01 * * *"
version: 1
creds:
  - registry: localhost:5000
    tls: disabled
  - registry: docker.io
    user: "{{env \"HUB_USER\"}}"
    pass: "{{file \"/var/run/secrets/hub_token\"}}"
defaults:
  ratelimit:
    min: 100
    retry: 15m
  parallel: 2
sync:
  - source: busybox:latest
    target: localhost:5000/library/busybox:latest
    type: image
    interval: 60m
    backup: "backup-{{.Ref.Tag}}"
  - source: alpine
    target: localhost:5000/library/alpine
    type: repository
    tags:
      allow:
      - "latest"
      - "edge"
      - "3"
      - "3.\\d+"
      deny:
      - "3.0"
    schedule: *sched-a
    backup: "{{$t := time.Now}}{{printf \"%s/backups/%s:%s-%d%d%d\" .Ref.Registry .Ref.Repository .Ref.Tag $t.Year $t.Month $t.Day}}"
```

- `version`:
  This should be left at version 1 or not included at all.
  This may be incremented if future `regsync` releases change the configuration file structure.

- `creds`:
  Array of registry credentials and settings for connecting.
  To avoid saving credentials in the same file with the other settings, consider using the `${HOME}/.docker/config.json` or a template in the `user` and `pass`
  fields to expand a variable or file contents.
  When using the `regclient/regsync` image, the docker config is read from `/home/appuser/.docker/config.json`.
  Each `creds` entry supports the following options:
  - `registry`:
    Hostname and port of the registry server used in image references.
    Use `docker.io` for Docker Hub.
    Note for parsing image names, a registry name must have a `.` or `:` to distinguish it from a path on Docker Hub.
  - `hostname`:
    Optional DNS name and port for the registry server, the default is the registry name.
    This allows multiple registry names to point to the same server with different configurations.
    This may be useful for different user logins, or different mirror configurations.
  - `user`:
    Username
  - `pass`:
    Password
  - `tls`:
    Whether TLS is enabled/verified.
    Values include "enabled" (default), "insecure", or "disabled".
  - `regcert`:
    Registry CA certificate for self signed certificates.
    This may be a string with `\n` for line breaks, or the yaml multi-line syntax may be used like:

    ```yaml
    regcert: |
      -----BEGIN CERTIFICATE-----
      MIIJDDCCBPSgAwIB....
      -----END CERTIFICATE-----
    ```

  - `pathPrefix`:
    Path added before all images pulled from this registry.
    This is useful for some mirror configurations that place images under a specific path.
  - `mirrors`:
    Array of registry names to use as a mirror for this registry.
    Mirrors are sorted by priority, highest first.
    This registry is sorted after any listed mirrors with the same priority.
    Mirrors are not used for commands that change the registry, only for read commands.
  - `priority`:
    Non-negative integer priority used for sorting mirrors.
    This defaults to 0.

- `defaults`:
  Global settings and default values applied to each sync entry:
  - `backup`:
    Tag or image reference for backing up target image before overwriting.
    This may include a Go template syntax.
    This backup is only run when the source changes and the target exists that is about to be overwritten.
    If the backup tag already exists, it will be overwritten.
  - `interval`:
    How often to run each sync step in `server` mode.
  - `schedule`:
    Cron like schedule to run each step, overrides `interval`.
  - `ratelimit`:
    Settings to throttle based on source rate limits.
    - `min`:
      Minimum number of pulls remaining to start the step.
      Actions while running the step can result in going below this limit.
      Note that parallel steps and multi-platform images may each result in more than one pull happening beyond this threshold.
    - `retry`:
      How long to wait before checking if the rate limit has increased.
  - `parallel`:
    Number of concurrent image copies to run.
    All sync steps may be started concurrently to check if a mirror is needed, but will wait on this limit when a copy is needed.
    Defaults to 1.
  - `mediaTypes`:
    Array of media types to include.
    These must also be supported by regclient.
    Defaults to: `["application/vnd.docker.distribution.manifest.v2+json", "application/vnd.docker.distribution.manifest.list.v2+json", "application/vnd.oci.image.manifest.v1+json", "application/vnd.oci.image.index.v1+json"]`
  - `skipDockerConfig`:
    Do not read the user credentials in `${HOME}/.docker/config.json`.

- `sync`:
  Array of steps to run for copying images from the source to target repository.
  - `source`:
    Source image or repository.
  - `target`:
    Target image or repository.
  - `type`:
    "repository" or "image".
    Repository will copy all tags from the source repository.
  - `tags`:
    Implements filters on tags for "repository" types, regex values are automatically bound to the beginning and ending of each string (`^` and `$`).
    - `allow`:
      Array of regex strings to allow specific tags.
    - `deny`:
      Array of regex strings to deny specific tags.
  - `platform`:
    Single platform to pull from a multi-platform image, e.g. `linux/amd64`.
    By default all platforms are copied along with the original upstream manifest list.
    Note that looking up the platform from a multi-platform image counts against the Docker Hub rate limit, and that rate limits are not checked prior to resolving the platform.
    When run with "server", the platform is only resolved once for each multi-platform digest seen.
  - `backup`, `interval`, `schedule`, `ratelimit`, and `mediaTypes`:
    See description under `defaults`.

- `x-*`:
  Any field beginning with `x-` is considered a user extension and will not be parsed in current or future versions of the project.
  These are useful for integrating your own tooling, or setting values for yaml anchors and aliases.

[Go templates](https://golang.org/pkg/text/template/) are used to expand values in `registry`, `user`, `pass`, `regcert`, `source`, `target`, and `backup`.
The `backup` template supports the following objects:

- `.Ref`: Reference object about to be overwritten
  - `.Ref.Reference`: Full reference
  - `.Ref.Registry`: Registry name
  - `.Ref.Repository`: Repository
  - `.Ref.Tag`: Tag
- `.Step`: Values from the current step
  - `.Step.Source`: Source
  - `.Step.Target`: Target
  - `.Step.Type`: Type
  - `.Step.Backup`: Backup
  - `.Step.Interval`: Interval
  - `.Step.Schedule`: Schedule

See [Template Functions](README.md#Template-Functions) for more details on the custom functions available in templates.