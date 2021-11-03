---
layout: post
title: "SSH deployments using GitHub Actions without dependencies"
date:   2021-11-03 16:45:44 +0200
categories: "github actions"
---

When it comes to deploying an application using GitHub Actions there is generally a standard way to do it when using a
PaaS, or you already use a deployment solution like [Capistrano](https://capistranorb.com/),
[Deployer](https://deployer.org/) or [shipit](https://github.com/shipitjs/shipit).

What about deploying using plain old SSH to get application code up to date though? This is by no means a proper
solution to deploying applications of course, but it is an always valid way to do it when deploying to nothing more than
a single directory on a single server.

There seems to be quite a few solutions that either solve this task or at least help on the
[actions marketplace](https://github.com/marketplace?category=deployment&type=actions&query=deploy+ssh), but do we
really need a ready-made Action for something this simple, especially since we are going to need to setup the
secrets anyway?

Is it not preferable to write the couple lines required to achieve this so that we do not depend on external factors and
get to name the secrets as we wish as a bonus in the process?

All in all, here is my take on how to achieve automated deployments using nothing but SSH.

## Deploying manually

First things first. We want to deploy to `example.com` which runs SSH on port `1025`, as the user `joe`, using the key
at `~/.ssh/id_rsa`. The application is in `/home/joe/app` and we will be using the remote `origin` which already exists,
because, well, we cloned the application in first place. Oh, and we want to run `touch tmp/restart.txt` after updating
the code.

To achieve this, without and Action, we would need to:

1. Setup key based authentication on the remote server by adding the contents of `~/.ssh/id_rsa.pub`
   to `/home/joe/.ssh/authorized_keys` (and chmod file to 600)
2. SSH into the server: `ssh ssh://joe@example.com:1025 -i ~/.ssh/id_rsa`
3. Go to the application's directory: `cd /home/joe/app`
4. Fetch updates: `git fetch origin deploy`
5. Update the code: `git reset --hard origin/deploy`
6. Restart the application: `touch tmp/restart.txt`

And if we were to use a one-liner:

```bash
ssh "ssh ssh://joe@example.com:1025 -i ~/.ssh/id_rsa " \
   cd /home/joe/app && \
   git fetch origin deploy && \
   git reset --hard origin/deploy && \
   touch tmp/restart.txt
```

Note that we prefer `git fetch origin deploy && git reset --hard origin/deploy` because this way we (almost) zero out
the chances to end up with conflicts.

## Deploying with an Action

The Action will be going with the one-liner version above of course, so let's set it up. Before anything, we should
generate a new private key for deployments using ```ssh-keygen -t rsa -b 4096 -q -N "" -f ~/.ssh/id_rsa_deploy``` and
add `~/.ssh/id_rsa_deploy.pub` contents to `/home/joe/.ssh/authorized_keys`.

> **Important!** For the action to work the server must have _direct_ pull access to the github repository. We may
> sometimes be able to pull changes over SSH because our keys are being forwarded without the server being able to pull.
> A deploy key must exist and the account pulling the changes should be configured accordingly. Refer to
> [managing deploy keys](https://docs.github.com/en/developers/overview/managing-deploy-keys#deploy-keys) on GitHub Docs
> for more information.

### Setting up the secrets

We will need all of the deployment information to be hidden into secrets for security of course:

| secret name | value |
| --- | --- |
| `DEP_HOST` | `example.com` |
| `DEP_PORT` | `1025` |
| `DEP_USER` | `joe` |
| `DEP_PRIVATE_KEY` | `~/.ssh/id_rsa_deploy` contents |
| `DEP_PATH` | `/home/joe/app` |
| `DEP_REMOTE` | `origin` |
| `DEP_RESTART_CMD` | `touch tmp/restart.txt` |

### Setting up the action

The action itself is quite straightforward.

#### Run the action when pushing to the `deploy` branch

We want the action to run only when pushing to the `deploy` branch, otherwise the application will be updated with
whatever work in progress code we happen to push every time:

```yml
{% raw %}name: Deploy
on: [ push ]

jobs:
   deploy:
      if: github.ref == 'refs/heads/deploy'
      runs-on: ubuntu-latest
...{% endraw %}
```

#### Resolve the base branch name

The only thing we need to take care of now, other than actually running the command, is to get the current branch name,
as the deployment needs to use it. After all, we are pushing to `deploy` because we want the application to use its
version of the code.

For this we will add a step to the action which does that and only that and extracts the branch name
from `github.ref` ([source](https://stackoverflow.com/questions/58033366/how-to-get-the-current-branch-within-github-actions)):

```yml
{% raw %}...
    steps:
      - uses: actions/checkout@v2
      - name: Extract branch name
        shell: bash
        run: echo "##[set-output name=branch;]$(echo ${GITHUB_REF#refs/heads/})"
        id: extract_branch
...{% endraw %}
```

#### Setting up environment variables

Then we pass the secrets we created earlier and run the deployment command. We use `steps.extract_branch.outputs.branch`
from the previous step to determine the branch that we will be using for deployment, and the final environment variables
are:

```yml
{% raw %}...
    steps:
...
      - name: "Deploy staging"
        env:
           DEP_HOST: "${{ secrets.DEPLOYMENT_SSH_HOST }}"
           DEP_PORT: "${{ secrets.DEPLOYMENT_SSH_PORT }}"
           DEP_USER: "${{ secrets.DEPLOYMENT_SSH_USER }}"
           DEP_PRIVATE_KEY: "${{ secrets.DEPLOYMENT_SSH_PRIVATE_KEY }}"
           DEP_PATH: "${{ secrets.DEPLOYMENT_PATH }}"
           DEP_REMOTE: "${{ secrets.DEPLOYMENT_GIT_REMOTE }}"
           DEP_BRANCH: "${{ steps.extract_branch.outputs.branch }}"
           DEP_RESTART_CMD: "${{ secrets.DEPLOYMENT_RESTART_CMD }}"
...{% endraw %}
```

#### Performing the deployment

At last, we can run the deployment command:

```yml
{% raw %}...
    steps:
...
      - name: "Deploy staging"
        env:
          ...
        run: |
           echo "$DEP_PRIVATE_KEY" > ./identity && \
           chmod 600 ./identity && \
           ssh "ssh://${DEP_USER}@${DEP_HOST}:${DEP_PORT}" -i ./identity -o "StrictHostKeyChecking no" " \
             cd \"$DEP_PATH\" && \
             git fetch \"$DEP_REMOTE\" \"$DEP_BRANCH\" && \
             git reset --hard \"$DEP_REMOTE/$DEP_BRANCH\" && \
             eval \"$DEP_RESTART_CMD\" \
             "
...{% endraw %}
```

Notice how we need to change the permissions on the identity file and add `-o "StrictHostKeyChecking no"`. Without these
the command will not run because SSH will reject our certificate for having improper permissions, or it does not have
our remote server in its `known_hosts`.

Disabling host key checking is not my favourite, but it seems quite safe here and I deemed running `ssh-keyscan` to add
the host key and fixing its permissions an overkill for the project I was working on when I wrote this.

## The complete Action

Do note that this will Action not switch the current branch on the remote server, rather it will have it point it to the
ref the branch we are deploying to points.

```yml
{% raw %}name: Deploy
on: [ push ]

jobs:
  deploy:
    # needs:
    #   - Test
    if: github.ref == 'refs/heads/deploy'
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v2
      - name: Extract branch name
        shell: bash
        run: echo "##[set-output name=branch;]$(echo ${GITHUB_REF#refs/heads/})"
        id: extract_branch
      - name: "Deploy staging"
        env:
          DEP_HOST: "${{ secrets.DEPLOYMENT_SSH_HOST }}"
          DEP_PORT: "${{ secrets.DEPLOYMENT_SSH_PORT }}"
          DEP_USER: "${{ secrets.DEPLOYMENT_SSH_USER }}"
          DEP_PRIVATE_KEY: "${{ secrets.DEPLOYMENT_SSH_PRIVATE_KEY }}"
          DEP_PATH: "${{ secrets.DEPLOYMENT_PATH }}"
          DEP_REMOTE: "${{ secrets.DEPLOYMENT_GIT_REMOTE }}"
          DEP_BRANCH: "${{ steps.extract_branch.outputs.branch }}"
          DEP_RESTART_CMD: "${{ secrets.DEPLOYMENT_RESTART_CMD }}"
        run: |
          echo "$DEP_PRIVATE_KEY" > ./identity && \
          chmod 700 ./identity && \
          ssh "ssh://${DEP_USER}@${DEP_HOST}:${DEP_PORT}" -i ./identity -o "StrictHostKeyChecking no" " \
            cd \"$DEP_PATH\" && \
            git fetch \"$DEP_REMOTE\" \"$DEP_BRANCH\" && \
            git reset --hard \"$DEP_REMOTE/$DEP_BRANCH\" && \
            eval \"$DEP_RESTART_CMD\" \
            "{% endraw %}
```

## Farewell

This is a simple Action for SSH deployments.

Of course we can easily have it require that our tests pass before deploying (see commented `needs:` above) or expand it
to support multiple environments by adding multiple jobs that run on different branches (ie deploy_staging,
deploy_production) using different keys.

It can also be improved by adding the server's SSH key to the known hosts and not disable `StrictHostKeyChecking` or
even by having it switch branches on the server upon deployment.

But enough is enough and this is just enough for the project it serviced (and maybe other similar use cases)!
