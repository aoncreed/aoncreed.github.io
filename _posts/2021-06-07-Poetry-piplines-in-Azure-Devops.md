---
title: How to use Poetry for installing and publishing to and from an Artifact Feed 
tags: poetry python azure-devops pipelines devops artifact-feed
---

# How to use poetry for installing and publishing to and from an Artifact Feed 

Poetry is a great package manager for Python, however it is not always supported for the different devops tools out there. This little guide offers a simple solution for the cases when you want to both install a private python package from a Azure Devops Artifact Feed and also publish the result of your current build to an Arifact Feed by using the `poetry build`command.

Poetry is not directly suppored by Azure Devops, as it only has tasks for pip & twine. However we can use poetry in our pipelines to build our python package by using a few tricks. See the example below.

```yml
trigger:
  - main

stages:
  - stage: buildAndRelease
    jobs:
      - job: build
        pool:
          vmImage: ubuntu-20.04
        displayName: build and publish python package
        steps:
          - task: UsePythonVersion@0
            displayName: Use Python 3.8
            inputs:
              versionSpec: 3.8

          - script: |
              cd $PYTHON_PATH
              python -m pip install poetry
              poetry config repositories.<ARTIFACT_FEED_NAME> https://dev.azure.com/<ORG_NAME>/<PROJECT_NAME>/_packaging/<ARTIFACT_FEED_NAME>/pypi/
              poetry config http-basic.<ARTIFACT_FEED_NAME> whatever $(System.AccessToken)
            displayName: Install & configure up Poetry
            env:
              PYTHON_PATH: .

          - script: |
              poetry lock
              poetry install
            displayName: Install packages

          - script: |
              poetry build
            displayName: Build package

          - script: |
              poetry add twine
            displayName: Install twine

          - task: TwineAuthenticate@1
            inputs:
              artifactFeed: <PROJECT_NAME>/<ARTIFACT_FEED_NAME>
            displayName: Twine auth task

          - script: |
              poetry run python -m twine upload -r <ARTIFACT_FEED_NAME> --config-file $(PYPIRC_PATH) dist/*
            displayName: Upload package to Artifact Feed with twine
            env:
              PYTHON_PATH: .
```
We simply use poetrys config command to give it an external PyPi referance, which for Azure Devops will be your Artifact Feed. For the http-basic config key we can simply use whatever we want for a username, as Azure Devops only cares about the password field when doing a token based authentication. The `$(System.AccessToken)` is an Environment Variable giving us access to an access token to verify with the feed.
```yml
...
              poetry config repositories.<ARTIFACT_FEED_NAME> https://dev.azure.com/<ORG_NAME>/<PROJECT_NAME>/_packaging/<ARTIFACT_FEED_NAME>/pypi/
              poetry config http-basic.<ARTIFACT_FEED_NAME> whatever $(System.AccessToken)
...
```
For publishing the end result of `poetry build` twine is used. This is simply because of an 404 error caused when trying to use `poetry publish -r <ARTIFACT_FEED_NAME>`. The error encountered was 
```bash
  UploadError

  HTTP Error 404: Not Found

  at /opt/hostedtoolcache/Python/3.8.10/x64/lib/python3.8/site-packages/poetry/publishing/uploader.py:216 in _upload
      212│                     self._register(session, url)
      213│                 except HTTPError as e:
      214│                     raise UploadError(e)
      215│ 
    → 216│             raise UploadError(e)
      217│ 
      218│     def _do_upload(
      219│         self, session, url, dry_run=False
      220│     ):  # type: (requests.Session, str, Optional[bool]) -> None
##[error]Bash exited with code '1'.
```
Therefore simply installing twine with poetry and making use of the `TwineAuthenticate@1` devops task.
