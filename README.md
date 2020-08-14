# concourse-cheatsheet

> Cheatsheet for working with Concourse CI

## Table of Contents

- [Fly CLI](#fly-cli)
- [Pipleline Configurations](#pipleline-configurations)
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

The same anchor syntax can be used to spread yaml object.
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

## Links

- [Concourse CI Pipeline Dashboard](https://ci.concourse-ci.org/) - Example dashboard/pipelines
- [Concourse Internals](https://concourse-ci.org/internals.html) - Deeper understanding of Concourse
