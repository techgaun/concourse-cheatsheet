[![Concourse CI](https://avatars1.githubusercontent.com/u/7809479?s=50&v=4)](https://concourse-ci.org/)
# concourse-cheatsheet

> Cheatsheet for working with Concourse CI

This cheatsheet lists various useful tips and tricks to use with [Concourse CI](https://concourse-ci.org/)
and thus omits the very basics such as logging in or setting pipeline from yaml file.

## Table of Contents

- [Fly CLI](#fly-cli)
- [Pipleline Configurations](#pipleline-configurations)
- [Miscellaneous](#miscellaneous)
- [Editor Support](#editor-support)
- [Links](#links)

## Fly CLI

### [Fly completion](https://concourse-ci.org/fly.html#fly-completion)

```shell
# for bash, put the following on your .bashrc
source <(fly completion --shell bash)

# for zsh, put the following on your .zshrc
source <(fly completion --shell zsh)
```

### List pipelines in all teams

```shell
# lists all the accessible pipelines regardless of what team you're on in the target
fly -t <your_target> ps -a
```

### List all workers with details

```shell
fly -t <your_target> workers --details
```

### List build containers

```shell
fly -t <your_target> containers

# above command also provides handle id which you can use to intercept
fly -t <your_target> intercept --handle <handle_id> # eg. handle_id - 5f588b86-116b-4e7c-5bef-811dad839539
```

### Intercept build aka grab an interactive shell inside build container for debugging

```shell
# simple example to connect to a particular pipeline's specific job
fly -t <your_target> intercept --job <PIPELINE>/<JOB_NAME>

# simple example to connect to a particular pipeline's specific job and run specific binary
# this is useful to specify custom shell such as sh in alpine or any arbitrary command during intercept
fly -t <your_target> intercept --job <PIPELINE>/<JOB_NAME> <your_custom_binary>
```

### Arbitrary API requests to your Concourse CI

```shell
fly -t <your_target> curl /api/v1/info
> {"version":"6.1.0","worker_version":"2.2","external_url":"https://concourse.example.com"}

fly -t <your_target> curl /api/v1/builds
> json_array_of build_lists
```

### Make pipeline visible to unauthenticated users

This is often useful for open-source projects so that the build pipeline
is visible for unauthenticated users.

```shell
fly -t <your_target> expose-pipeline --pipeline <YOUR_PIPELINE>
```

### Format your pipeline

Concourse has a handy fly command to format your pipeline in a "canonical" form.
Omitting `-w` / `--write` would print formatted pipeline on stdout instead.
This is useful when you've made a mess of your yaml configuration formatting.

```shell
fly format-pipeline -c <YOUR_PIPELINE.yml> -w
```

### Different team in your fly commands

By default, concourse sets team on a target when you login and its cumbersome
to manage multiple targets just to be working on a different team space. Fly
commands support `--team` so you can specify another team name in a single target.
However, this is a work in progress as tracked [HERE](https://github.com/concourse/concourse/issues/5215)
so make sure you check the progress in above issue or by checking the help information.

## Pipleline Configurations

### Root privileges in container for your jobs

- Use `privileged: true` in the task config.

```yaml
- task: some-task
  privileged: true
```

### Custom icons for your resources

- You can use icon name from [Material Design icon](https://materialdesignicons.com/) to get nice little icon by your resource.

```yaml
- resources:
  - name: my-image
    type: registry-image
    icon: docker
```

### YAML anchors to re-use configuration blocks

You can use YAML anchor to re-use configuration blocks and remove duplicates.
Often times, you can use `file` directive but still the anchor syntax can be useful.

```yaml
# this is an example from concourse docs
# the following repetitive blocks can be shortened using yaml anchor syntax
large_value:
  do_the_thing: true
  with_these_values: [1, 2, 3]

duplicate_value:
  do_the_thing: true
  with_these_values: [1, 2, 3]

# look how we anchor a block with &anchor_name syntax and reference it with *anchor_name
large_value: &my_anchor
  do_the_thing: true
  with_these_values: [1, 2, 3]

duplicate_value: *my_anchor
```

The same anchor syntax can be used to merge yaml objects.
On the example below, you can see how we can avoid duplicate
AWS ECR configuration by using anchor.

```yaml
aws-ecr-config: &aws-ecr-config
  aws_role_arn: ((ecr-role-arn))
  aws_region: ((ecr-region))
  aws_access_key_id: ((ecr-access-key-id))
  aws_secret_access_key: ((ecr-secret-access-key))

resources:
  - name: elixir-1.8.2
    type: registry-image
    source:
      repository: engineering/elixir
      tag: 1.8.2
      <<: *aws-ecr-config
      
  - name: elixir-1.10.1
    type: registry-image
    source:
      repository: engineering/elixir
      tag: 1.10.1
      <<: *aws-ecr-config
```

Here's another example:

```yaml
jobs:
- name: job-1
  plan:
    - &task-1
      task: task-1
      config:
        platform: linux

    - &task-2
      task: task-2
      config:
        platform: linux
- name: job-2
  - *task-1
  - *task-2
```

### Container CPU and Memory Limits for Task

```yaml
# you can configure and override default container limits
# cpu - max amount of CPU available to task container, measured in shares
# memory - max amount of memory available to task container
# 0 means unlimited
jobs:
  - name: container-limits-job
    plan:
      - task: task-with-container-limits
        config:
          platform: linux
          image_resource:
            type: mock
            source: {mirror_self: true}
          container_limits:
            cpu: 512
            memory: 1GB
          run:
            path: sh
            args: ["-c", "echo hello"]
```

### Pipeline Organization

While yaml tricks are nice for smaller pipelines, complex pipelines
benefit from a well-defined structure. Check out the [command schema](https://concourse-ci.org/tasks.html#schema.command)
and [task step config](https://concourse-ci.org/jobs.html#schema.step)
that shows an alternative `file` that allows you to point to a `.yml`
containing the task config.

A good starting example follows:

```shell
techgaun at techgaun in /home/techgaun/projects/rpi-ha
$ tre
      1 .
      2 └── .ci
      3     ├── dockerfiles
      4     │   ├── Dockerfile.alpine
      5     │   ├── Dockerfile.buster
      6     │   └── Dockerfile.distroless
      7     ├── pipelines
      8     │   ├── k8-build.yml
      9     │   └── swarm-build.yml
     10     ├── scripts
     11     │   ├── build
     12     │   ├── init
     13     │   └── test
     14     └── tasks
     15         ├── build.yml
     16         ├── e2e.yml
     17         └── test.yml
     18 
     19 5 directories, 11 files
```

## Miscellaneous

### Build Badges

- You can get a badge for your pipeline with the following URL:

```
/api/v1/teams/{team}/pipelines/{pipeline}/badge
```

Example from Concourse CI itself: [![Concourse CI Build](https://ci.concourse-ci.org/api/v1/teams/main/pipelines/concourse/badge)](https://ci.concourse-ci.org/teams/main/pipelines/concourse)
```
# snippet for above SVG
[![Concourse CI Build](https://ci.concourse-ci.org/api/v1/teams/main/pipelines/concourse/badge)](https://ci.concourse-ci.org/teams/main/pipelines/concourse)
```

- You can get a badge for your pipeline's specific jobs with the following URL:

```
/api/v1/teams/{team}/pipelines/{pipeline}/jobs/{job}/badge
```

Example from Concourse CI itself: [![Concourse CI Unit Tests](https://ci.concourse-ci.org/api/v1/teams/main/pipelines/concourse/jobs/unit/badge)](https://ci.concourse-ci.org/teams/main/pipelines/concourse/jobs/unit)

```
# snippet for above SVG
[![Concourse CI Unit Tests](https://ci.concourse-ci.org/api/v1/teams/main/pipelines/concourse/jobs/unit/badge)](https://ci.concourse-ci.org/teams/main/pipelines/concourse/jobs/unit)
```

- Additionally, Concourse API supports a team's pipeline in a [CCMenu](http://ccmenu.org/) compatible XML file.

```
/api/v1/teams/{team}/cc.xml
```

## Editor Support

- [Concourse CI Pipeline Editor](https://marketplace.visualstudio.com/items?itemName=Pivotal.vscode-concourse) - Provides validation and content assist for Concourse CI pipeline and task configuration yml files
- [Concourse-Vis](https://atom.io/packages/concourse-vis) - A plugin to preview Concourse pipelines in Atom.


## Links

- [Concourse CI Pipeline Dashboard](https://ci.concourse-ci.org/) - Example dashboard/pipelines
- [Concourse Internals](https://concourse-ci.org/internals.html) - Deeper understanding of Concourse
- [Concourse Tutorial by Stark & Wayne](https://concoursetutorial.com/) - A great introduction to Concourse
- [Pipelines Used by Concourse Team](https://github.com/concourse/pipelines) - A collection of pipelines used by the Concourse Team
- [Sample Concourse Pipeline Examples](https://github.com/concourse/concourse/tree/master/testflight/fixtures) - Fixtures from Concourse source code to understand pipelines better

## Authors

- [techgaun](https://github.com/techgaun)
- [bpote](https://github.com/bpote)
