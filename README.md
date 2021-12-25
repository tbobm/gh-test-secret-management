# Github Actions """Environment""" Secrets

[![Dynamic Secret Names](https://github.com/tbobm/gh-test-secret-management/actions/workflows/main.yaml/badge.svg)](https://github.com/tbobm/gh-test-secret-management/actions/workflows/main.yaml) [![Env-based dynamic names](https://github.com/tbobm/gh-test-secret-management/actions/workflows/synthetic.yaml/badge.svg)](https://github.com/tbobm/gh-test-secret-management/actions/workflows/synthetic.yaml)

**Goal:** Allow to use a specific secret if it exists and fallback to a default one otherwise.

Implementation based on [this StackOverflow answer][so-answer].

[so-answer]: https://stackoverflow.com/questions/61255989/dynamically-retrieve-github-actions-secret/61272209#61272209

## Context

We have two secrets: one for the "production" environment and another one for the "staging" environment.

- `DEPLOY_TARGET`:`main-server`
- `staging_DEPLOY_TARGET`: `staging-server`

When interacting (`push`) with the `main` branch, we want to use the `DEPLOY_TARGET` Secret.
When interacting (`push`) with the `staging` branch, we want to use the `staging_DEPLOY_TARGET` Secret.

Note: in this context, we take in account that [Github Actions Environments][gh-envs] are not enabled
for the concerned repository. This is mostly a custom implementation to mimic some features
exposed using [Environment Secrets][gh-env-secrets].

[gh-env-secrets]: https://docs.github.com/en/actions/deployment/targeting-different-environments/using-environments-for-deployment#environment-secrets
[gh-envs]: https://docs.github.com/en/actions/deployment/targeting-different-environments/using-environments-for-deployment#about-environments

## Implementation A

Configuration: [`main.yaml`](./.github/workflows/main.yaml)

Based on [Meir Gabay's answer][so-mg], we set the name of the Secret to be used as a key in the `prepare` job.
We can then reference it by specifying the `prepare` job as a `.needs` entry, thus allowing to reference the name
as a key in the `secrets` Github Action variable.

[so-mg]: https://stackoverflow.com/users/5285732/meir-gabay

```yaml
    env:
      # Get secret names
      DEPLOY_TARGET_NAME: ${{ needs.prepare.outputs.deploy_target_name }}
    steps:
      - name: Test Application
        env:
          # Inject secret values to environment variables
          DEPLOY_TARGET: ${{ secrets[env.DEPLOY_TARGET_NAME] }}
        run: |
          printenv | grep DEPLOY_
```

This is quite useful in situations where the secrets will be referenced more
than once to avoid duplicating steps.

## Implementation B

Configuration: [`synthetic.yaml`](./.github/workflows/synthetic.yaml)

This one is a more concise usage, only relying on Github Actions environment.

```yaml
      - name: Prepare custom Outputs
        id: prepare-env
        run: |
          echo "DEPLOY_TARGET_NAME=$([ $GITHUB_REF_SLUG == main ] && echo DEPLOY_TARGET || echo ${GITHUB_REF_SLUG}_DEPLOY_TARGET)" >> $GITHUB_ENV
      - name: Test Application
        env:
          # Inject secret values to environment variables
          DEPLOY_TARGET: ${{ secrets[env.DEPLOY_TARGET_NAME] }}
        run: |
          printenv | grep DEPLOY_
```

Here, we write to the `$GITHUB_ENV` variable, which leads to `DEPLOY_TARGET_NAME=staging_DEPLOY_TARGET` in further steps.

Exposing the content of the `DEPLOY_TARGET_NAME` variable could also be achieved using the prepare-env's `outputs`.

This second implementation is more useful in "simple" Workflows where we only reference the variables in a single Job and avoids
the overhead caused by the startup of a new Job.
