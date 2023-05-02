## Introduction
This is a walkthrough to construct a CI/CD pipeline using Jenkins Multi-Branch Pipeline.
This functionality enables self-service for devs to provision their own environments, simplify their CI/CD management and operations,
and facilitate automation.

### High-Level Flow
The `main` branch has 1 static pipeline, and dynamic pipeline to feature branches.

When a new feature branch is created:
* a new pipeline will be created
* during push or PR events, pipeline will execute logic
* success/failure result is returned and used for PR approval
* pipeline can destroy resources or leave running for analysis
* when branch is deleted:
  * pipeline will be deleted after set time-period
  * cleanup activities can be performed

When a feature branch is merged to main branch:
* `main` pipeline begins execution
* pipeline logic can either:
  * reuse artifacts from feature pipeline
  * re-execute pipeline again and produce new set of artifacts
* pipeline will deploy to pd
* on failure pipeline can revert the feature branch PR that led to disruption
* an email is generated notifying owners of execution result

## Pipeline Creation
Use a multi Branch Pipeline
* Configure source control credentials
* Specify repo path to `Jenkinsfile`
* Configure the jenkins to filter `main` branch from `feature` branch when running jobs
* define the build triggers

## Pipeline Branch Management
In our `Jenkinsfile` we must make sure `main` branch only deploy to `prod` environment,
other branches can execute to `dev` and `pre-prod`
* Use `when` step in stage
* Use `if`/`else` logic within scripts block

## Pre-Prod/Prod Environment Isolation
Isolate the `pre-prod` and `prod` environments by assigning a different agent to each of them.
Within the `Jenkinsfile`:
* Set the global agent for the pipeline to the `pre-prod` agent
* Explicitly set a `prod` agent on the stage that should deploy to `prod` environment

## Shared Libraries
Use shared libraries, typically saved in `vars/` folder to import groovy scripts that perform your specified functions.

## Post Stage
Especially when using terraform, if you wish to destroy an infrastructure after the pipeline has run,
* include terraform destroy step within the `post.always` block or `post.success` block

