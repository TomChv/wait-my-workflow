<p align="center">
  <a href="https://github.com/actions/typescript-action/actions"><img alt="typescript-action status" src="https://github.com/actions/typescript-action/workflows/build-test/badge.svg"></a>
</p>

# Github Action : Wait my workflow

This github action is based on  [wait-for-check](https://github.com/fountainhead/action-wait-for-check) action, please go put a star on his repo if you like my action. I simply altered the actions to match my needs.

It works thanks to the [check states](https://docs.github.com/en/rest/reference/checks) (github action jobs). The action will ping the github API and check if a job is running on the repo given as argument.

The objective of this action is to enable people to wait for jobs. It also handles the case where the job is not starting or does not exist.

The timeout starts when the job is starting and not when it is in a queue. This ensures that you do not have a timeout for nothing.


## Exemple

Let's imagine a scenario where you have to wait for a build to finish before launching your deployment workflow.
Your workflows files should look like this :


**Build workflow**
```yaml
name: Build my app

on:
  push:

jobs:
  build: # Here is your checkName var
    runs-on: self-hosted

    steps:
      ## Build you application.. Push it to docker for example
```

**Deployment workflow**
```yaml
    steps:
      - name: Wait a job
        uses: tomchv/waiter-test@v1.0.5
        id: wait-for-timeout
        with:
          token: ${{ secrets.GITHUB_TOKEN }}
          checkName: build
          ref: ${{ github.event.pull_request.head.sha || github.sha }}
          intervalSeconds: 10
          timeoutSeconds: 100

      - name: Do something if job success
        if: steps.wait-for-timeout.outputs.conclusion == 'success'
        run: echo success && true

      - name: Do something if job isn't launch
        if: steps.wait-for-timeout.outputs.conclusion == 'does not exist'
        run: echo job does not exist && true

      - name: Do something if job fail
        if: steps.wait-for-timeout.outputs.conclusion == 'failure'
        run: echo fail && false

      - name: Do something if job timeout
        if: steps.wait-for-timeout.outputs.conclusion == 'timed_out'
        run: echo Timeout && false

    #... Other steps to do after your build
```

## Inputs

This Action accepts the following configuration parameters via `with:`

- `token`

  **Required**
  
  The GitHub token to use for making API requests. Typically, this would be set to `${{ secrets.GITHUB_TOKEN }}`.
  
- `checkName`

  **Required**
  
  The name of the GitHub check to wait for. For example, `build` or `deploy`.

- `ref`

  **Default: `github.sha`**
  
  The Git ref of the commit you want to poll for a passing check.
  
  *PROTIP: You may want to use `github.pull_request.head.sha` when working with Pull Requests.*

  
- `repo`

  **Default: `github.repo.repo`**
  
  The name of the Repository you want to poll for a passing check.

- `owner`

  **Default: `github.repo.owner`**
  
  The name of the Repository's owner you want to poll for a passing check.

- `timeoutSeconds`

  **Default: `600`**

  The number of seconds to wait for the check to complete. If the check does not complete within this amount of time, this Action will emit a `conclusion` value of `timed_out`.
  
- `intervalSeconds`

  **Default: `10`**

  The number of seconds to wait before each poll of the GitHub API for checks on this commit.

## Outputs

This Action emits a single output named `conclusion`. Like the field of the same name in the [CheckRunEvent API Response](https://developer.github.com/v3/activity/events/types/#checkrunevent-api-payload), it may be one of the following values:

- `success`
- `failure`
- `neutral`
- `timed_out`
- `action_required`

These correspond to the `conclusion` state of the Check you're waiting on. In addition, this action will emit a conclusion of `timed_out` if the Check specified didn't complete within `timeoutSeconds`.

If the job does not exist, this Action will emit `not found` as conclusion.
