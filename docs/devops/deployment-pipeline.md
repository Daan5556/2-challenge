# Deployment pipeline

**Author:** Daan Eggen  
**Date:** 13/06/2026  
**Version:** 1.0

---

This document describes my implementation of the deployment workflow for the
Area42 application. The workflow is stored in
`area42/.github/workflows/deploy.yml` (included at the end of this document) and
is used to build a Docker image and publish it to Amazon Elastic Container
Registry (ECR).

The goal of this implementation is to make releasing the application more
controlled and repeatable. Instead of building and uploading an image manually,
Github Actions performs the same deployment steps every time.

## Context

Area42 is a web application for the challenge project. The application is
containerized with Docker, which means it can be built into a single deployable
image. This image contains the application code, PHP runtime, Composer
dependencies, Node dependencies and built frontend assets.

For this stage of the project, the deployment workflow focuses on publishing the
deployable container image to AWS ECR. ECR then becomes the controlled place
where release images are stored. A later hosting step, for example with ECS, EC2
or another container platform, can pull the image from ECR and run it.

## Workflow implementation

The deployment workflow is named `Deploy`:

```yml
name: Deploy

on:
  workflow_dispatch:
  push:
    tags:
      - "*.*.*"
```

The workflow can be started in two ways. The first way is manually through
`workflow_dispatch`. This is useful when I want to test or repeat a deployment
without creating a release tag. The second way is by pushing a version-style git
tag, for example `1.0.0`. This makes releases intentional, because a normal push
to a branch does not publish a deployment image.

The workflow contains one job:

```yml
jobs:
  deploy:
    runs-on: ubuntu-latest
```

The job runs on a Github-hosted Ubuntu runner. This gives the workflow a clean
environment for every deployment run.

## Deployment steps

The first step checks out the repository:

```yml
- name: Checkout repo
  uses: actions/checkout@v4
```

This gives the runner access to the source code that has to be built.

The second step configures AWS access:

```yml
- name: Configure AWS credentials
  uses: aws-actions/configure-aws-credentials@v4
  with:
    aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
    aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
    aws-region: eu-central-1
```

The AWS access key and secret key are stored as Github repository secrets. This
is important because credentials should not be committed to the repository. The
workflow only receives the credentials when it runs. I also selected the
`eu-central-1` region, so the deployment target is explicit and does not depend
on a default AWS setting.

The third step logs in to Amazon ECR:

```yml
- name: Login to ECR
  id: login-ecr
  uses: aws-actions/amazon-ecr-login@v2
```

This action authenticates Docker against ECR and exposes the registry URL
through `steps.login-ecr.outputs.registry`. The next step uses this value to
build the full Docker image name.

The final step builds, tags and pushes the Docker image:

```yml
- name: Build, tag, and push docker image to Amazon ECR
  working-directory: software
  env:
    REGISTRY: ${{ steps.login-ecr.outputs.registry }}
    REPOSITORY: area42-webapp
    IMAGE_TAG: ${{ github.ref_type == 'tag' && github.ref_name || github.sha }}
  run: |
    echo $REGISTRY/$REPOSITORY:$IMAGE_TAG
    docker build -t $REGISTRY/$REPOSITORY:$IMAGE_TAG .
    docker push $REGISTRY/$REPOSITORY:$IMAGE_TAG
```

The image is pushed to the `area42-webapp` ECR repository. The image tag is
chosen automatically. When the workflow runs because of a git tag, the Docker
image receives that same tag. When the workflow is started manually, the Docker
image receives the commit SHA. This keeps every image traceable to the source
code that produced it.

The workflow currently builds from `working-directory: software`. This means the
deployment setup assumes that the Dockerfile and application code are available
from that directory in the repository structure used by Github Actions. In this
local checkout, the Laravel application is in `area42/server`, so the working
directory is an important point to validate before depending on the workflow for
production releases.

## Release management

This implementation makes the development and release process more predictable
in several ways.

First, deployment is not triggered by every branch push. It only happens when I
manually start the workflow or when a version-style tag is pushed. This lowers
the risk of publishing unfinished work.

Second, the workflow uses repository secrets for AWS credentials. This keeps
access control inside Github and avoids storing credentials in the codebase.
Access to deployment can therefore be controlled through Github repository
permissions and AWS IAM permissions.

Third, the image name and repository are fixed in the workflow:

```text
area42-webapp
```

This prevents every developer from pushing images to different places or using a
different naming convention. The deployment output is predictable.

Fourth, the image tag is based on either the git tag or the commit SHA. This is
important for traceability. If a problem is found in a deployed image, I can
look at the image tag and find the exact source version that created it.

## Deployment feedback

Github Actions gives basic visibility for this workflow. Every deployment run
has a status, duration and log output per step. This makes it possible to see
where a deployment failed, for example during AWS authentication, ECR login,
Docker build or Docker push.

The workflow also prints the full image name before building:

```bash
echo $REGISTRY/$REPOSITORY:$IMAGE_TAG
```

This gives a clear reference in the deployment logs. After the workflow has
finished, the same image tag can be checked in ECR to confirm that the image was
published.

This feedback is focused on the deployment process itself. Runtime visibility
for the application, such as container health, request errors, logs, CPU usage,
memory usage and database availability, is not implemented in this workflow yet.
A logical next step would be to connect the runtime platform to CloudWatch and
define health checks for the running container.

## Automation and improvements

The main improvement is that the build and upload process is automated. Without
this workflow, deployment would require manual Docker commands and manual AWS
login steps. Manual deployment is slower and easier to do inconsistently.

The workflow also improves communication within the team. Because the release
artifact is created by Github Actions, developers do not have to ask which local
machine produced an image or whether the correct version was uploaded. The
workflow log becomes the shared source of truth.

Using Docker also reduces environment differences. The same Dockerfile is used
to create the application image each time. This makes the rollout more
predictable because the application is delivered as a container image instead of
as loose files that have to be configured manually on a server.

There are still improvements that can be added later:

- add Docker layer caching to reduce build time
- run tests before publishing the image
- fail the workflow when the release tag does not follow the expected format
- use separate AWS IAM permissions that only allow pushing to the required ECR
  repository
- add a deployment step that updates the runtime environment after the image is
  pushed
- add a rollback strategy using previous image tags

## Challenges

One challenge was deciding when the deployment workflow should run. If the
workflow ran on every push to `main`, the ECR repository could quickly fill with
images that are not real releases. I chose tag-based deployment because a tag is
a deliberate release action. Manual dispatch is still available for controlled
testing.

A second challenge was handling credentials. Docker needs access to ECR, but the
AWS keys should not be visible in the repository. Github secrets solve this
problem because the values are injected into the workflow at runtime and are not
printed in the workflow file.

A third challenge was choosing the image tag. A fixed tag like `latest` would be
easy, but it would make rollback and debugging harder because `latest` changes
over time. I chose a tag-based or SHA-based image tag so every image points back
to a specific source version.

A fourth challenge is that the workflow only publishes the image. It does not
yet update a running production environment. This is acceptable for the current
stage because publishing a reliable image is the first step toward automated
commissioning. The next step is connecting the published image to the runtime
platform.

## Decisions

| Decision                                | Reason                                                                  |
| --------------------------------------- | ----------------------------------------------------------------------- |
| Use Github Actions                      | It is integrated with the repository and gives visible deployment logs  |
| Use AWS ECR                             | It is a managed container registry and fits with AWS deployment options |
| Use `workflow_dispatch`                 | It allows controlled manual deployment runs                             |
| Deploy on version-style tags            | It makes releases intentional and avoids deployment on every push       |
| Store AWS credentials in Github secrets | It keeps credentials out of the repository                              |
| Use `eu-central-1`                      | It makes the AWS target region explicit                                 |
| Tag images with git tag or commit SHA   | It keeps the deployment artifact traceable                              |
| Push to `area42-webapp`                 | It gives the application image one clear repository                     |

## Operational impact

The workflow defines how the Area42 web application is packaged and made
available for deployment. The release process is handled as a repeatable project
activity instead of a set of manual commands that have to be remembered by each
developer.

The deployment trigger keeps releases controlled. A normal branch push does not
publish a deployment image. Publishing only happens when the workflow is started
manually or when a version tag is pushed.

The Github Actions run gives direct feedback about the deployment process. Each
step has its own status and logs, so it is clear whether a failure happened
during checkout, AWS authentication, ECR login, Docker build or Docker push.

The Docker image tag keeps the deployment artifact traceable. Release builds use
the git tag, and manual builds use the commit SHA. This makes it possible to
connect a pushed image back to the exact source version.

The workflow also makes future rollout easier. Once a runtime environment is
added, it can pull a known image from ECR instead of depending on a manual copy
of the application code.

## Conclusion

With this deployment workflow, the Area42 project has a repeatable way to
publish Docker images to AWS ECR. Releases are limited to manual runs or version
tags, credentials are handled through Github secrets, and every pushed image can
be connected back to a source version.

The implementation is not a complete production deployment yet, because it does
not update a running container environment or check runtime health. However, it
is an important release step: the application can now be packaged and stored as
a traceable deployable artifact. This creates a solid base for later runtime
deployment, health checks and rollback improvements.

```yaml
# .github/workflows/deploy.yml
name: Deploy

on:
  workflow_dispatch:
  push:
    tags:
      - "*.*.*"

jobs:
  deploy:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout repo
        uses: actions/checkout@v4

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: eu-central-1

      - name: Login to ECR
        id: login-ecr
        uses: aws-actions/amazon-ecr-login@v2

      - name: Build, tag, and push docker image to Amazon ECR
        working-directory: software
        env:
          REGISTRY: ${{ steps.login-ecr.outputs.registry }}
          REPOSITORY: area42-webapp
          IMAGE_TAG:
            ${{ github.ref_type == 'tag' && github.ref_name || github.sha }}
        run: |
          echo $REGISTRY/$REPOSITORY:$IMAGE_TAG 
          docker build -t $REGISTRY/$REPOSITORY:$IMAGE_TAG .
          docker push $REGISTRY/$REPOSITORY:$IMAGE_TAG
```
