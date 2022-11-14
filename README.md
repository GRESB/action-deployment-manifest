# action-deployment-manifest

A GitHub Action to generate and read deployment manifests.
This is a composite action that combines other actions, like:

- peter-evans/find-comment
- peter-evans/create-or-update-comment

This action is used to automate deployments from GitHub issues and pull requests.
Deployments from GitHub issues offer a way to interactively deploy a release to multiple environments while preserving
deployment order. The deployment issue title names the release to be deployed using a simple format `Deploy v1.2.3`.
This action will parse the release to be deployed from the issue title and produce a deployment manifest comment.
The comment will contain human-readable deployment information as well as a JSON structure for use in automation.

## Inputs

| Input           | Description                                                                                                                                                                                                      | Required | Default                  |
|-----------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------|--------------------------|
| create          | Whether to create a deployment manifest comment.                                                                                                                                                                 | true     | ''                       |
| read            | Whether to read a deployment manifest comment.                                                                                                                                                                   | true     | ''                       |
| issue-number    | The number of the deployment issue or pull request.                                                                                                                                                              | true     |                          |
| issue-title     | The title of the deployment issue, used when creating the manifest.                                                                                                                                              | false    |                          |
| comment-header  | The header of the manifest comment, used to create new comments and find existing comments.                                                                                                                      | true     | '## Deployment manifest' |
| target-runtimes | The runtimes where to deploy the release.                                                                                                                                                                        | true     | 'tst acc prd'            |
| deploy-label    | The label that is used to trigger the manifest generation.                                                                                                                                                       | true     | 'deploy'                 |
| release         | When reading the manifest, if this input is set then the action will skip reading the comment and will take this value as the release to deploy. Setting this value is only meaningful when `read = true`.       | false    |                          |
| runtime         | The runtime targeted by a deployment, which will be used to validate if the runtime is part of the target runtimes defined in the deployment manifest. Setting this value is only meaningful when `read = true`. | false    |                          |

## Outputs

| Output   | Description                         |
|----------|-------------------------------------|
| release  | The release to deploy.              |
| runtimes | The target runtimes for deployment. |

## Usage

The two main use-cases for this action are the creation of a deployment manifest for a deployment issue, and reading
those manifests right before triggering an actual deployment.

### Create a and consume deployment manifests

The following workflow configuration creates a deployment manifest when an issue is labeled with `deploy`, and it then
deploys the release on the manifest to a series of environments each time a user comments a deployment confirmation
message on the issue.

```yaml
name: Deploy Release


on:
  issues:
    types: [labeled]
  issue_comment:
    types: [created]


jobs:
  prepare-deployment:
    name: Create deployment manifest
    runs-on: ubuntu-latest
    # Only run on issues tagged with "deploy"
    if: github.event.action == 'labeled' && github.event.label.name == 'deploy'
    steps:
      - name: Checkout sources
        uses: actions/checkout@v3
      - name: Create deployment manifest
        uses: ./
        with:
          create: true
          issue-number: ${{ github.event.issue.number }}
          issue-title: ${{ github.event.issue.title }}


  deploy-tst:
    name: Deploy manifest to TST
    # only run if:
    #  - is an issue comment
    #  - issue is labeled with "deploy"
    #  - comment starts with "deploy tst"
    if: !github.event.issue.pull_request && contains(github.event.issue.labels.*.name, 'deploy') && startsWith(github.event.comment.body, 'deploy tst')
    runs-on: ubuntu-latest
    steps:
      - name: Checkout sources
        uses: actions/checkout@v3
      - name: Read release
        id: read-release
        uses: ./
        with:
          read: true
          issue-number: ${{ inputs.issue-number }}
      - run: |
          echo "Deploying to TST"
          echo "Release ${release}"
        env:
          release: ${{ steps.read-release.outputs.release }}


  deploy-acc:
    name: Deploy manifest to ACC
    # only run if:
    #  - is an issue comment
    #  - issue is labeled with "deploy"
    #  - comment starts with "deploy acc"
    if: !github.event.issue.pull_request && contains(github.event.issue.labels.*.name, 'deploy') && startsWith(github.event.comment.body, 'deploy acc')
    runs-on: ubuntu-latest
    steps:
      - name: Checkout sources
        uses: actions/checkout@v3
      - name: Read release
        id: read-release
        uses: ./
        with:
          read: true
          issue-number: ${{ inputs.issue-number }}
      - run: |
          echo "Deploying to ACC"
          echo "Release ${release}"
        env:
          release: ${{ steps.read-release.outputs.release }}


  deploy-prd:
    name: Deploy manifest to PRD
    # only run if:
    #  - is an issue comment
    #  - issue is labeled with "deploy"
    #  - comment starts with "deploy prd"
    if: !github.event.issue.pull_request && contains(github.event.issue.labels.*.name, 'deploy') && startsWith(github.event.comment.body, 'deploy prd')
    runs-on: ubuntu-latest
    steps:
      - name: Checkout sources
        uses: actions/checkout@v3
      - name: Read release
        id: read-release
        uses: ./
        with:
          read: true
          issue-number: ${{ inputs.issue-number }}
      - run: |
          echo "Deploying to PRD"
          echo "Release ${release}"
        env:
          release: ${{ steps.read-release.outputs.release }}


  close-issue:
    name: Close issue
    needs:
      - deploy-prd
    env:
      runtime: PRD
    runs-on: ubuntu-latest
    steps:
      - name: Close Issue
        uses: peter-evans/close-issue@v2
        if: ${{ needs.deploy-prd.outputs.result == 'success' }}
        with:
          issue-number: ${{ github.event.issue.number }}
          comment: Auto-closing issue after successful deployment
```
