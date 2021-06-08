---
title: Poetry Pipelines in Azure Devops
tags: poetry python azure-devops pipelines devops
---

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
            displayName: "Install & configure up Poetry"
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
