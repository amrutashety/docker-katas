# CI/CD integration

## Learning Goals

- Understand why Docker is used as the build/test/ship unit in a CI/CD pipeline
- Lint a Dockerfile automatically as part of a pipeline
- Scan an image for vulnerabilities automatically as part of a pipeline
- Build, tag and push an image the way a pipeline would

## Introduction

A CI/CD pipeline that "builds" your application should, for a containerized app, really just be a sequence of `docker` commands run by a robot instead of a human:

1. `docker build` the image
2. Check it (lint the Dockerfile, scan the image, run tests inside a container built from it)
3. `docker tag` it with something traceable (a commit SHA, a version)
4. `docker push` it to a registry
5. Somewhere else, `docker pull` and run that exact same image

Because the image is immutable and contains everything the app needs to run, "it works on my machine" mostly stops being a problem: the image that passed CI is the exact same image that runs in production.

## Exercise

### Overview

- Lint a Dockerfile automatically with `hadolint`
- Build and tag an image the way a pipeline would, using the git commit as the tag
- Scan the built image for known vulnerabilities with `trivy`
- Look at a minimal GitHub Actions workflow that does all of this

### Step by step instructions

<details>
<summary>Lint the Dockerfile</summary>

A pipeline should catch bad practices in your Dockerfile before it even builds, using [hadolint](https://github.com/hadolint/hadolint) - the same linter mentioned in [image-best-practices](image-best-practices.md).

> :bulb: This exercise uses the complete Dockerfile in [cicd-pipeline](cicd-pipeline/), rather than the fill-in-the-blank one from [07-building-an-image](07-building-an-image.md) - a pipeline needs a Dockerfile that actually builds, not a template with blanks left in it.

- From the [cicd-pipeline](cicd-pipeline/) folder, run hadolint against the Dockerfile via its Docker image (no local install needed, which is exactly why this is convenient in CI):

  ```bash
  cd cicd-pipeline

  docker run --rm -i hadolint/hadolint < Dockerfile
  ```

- Introduce an obvious bad practice, e.g. add `RUN apt-get update` on its own line without `apt-get install` in the same layer, and re-run hadolint - you should see it flag `DL3009` / `DL3015`-style warnings.

> :bulb: In a real pipeline, this step should **fail the build** (non-zero exit code) if issues are found, so bad Dockerfiles never even reach the build step.

</details>

<details>
<summary>Build and tag like a pipeline would</summary>

Pipelines never tag images `:latest` - they use something traceable back to the exact source code that produced the image, most commonly the git commit SHA.

- If this repository is a git checkout, simulate what a pipeline does:

  ```bash
  cd cicd-pipeline

  export IMAGE_TAG=$(git rev-parse --short HEAD)

  docker build -t myfirstapp:$IMAGE_TAG .

  docker image ls myfirstapp
  ```

- Also tag it `:latest` in addition to the commit SHA (common practice, so there is always one "current" tag alongside the immutable ones):

  ```bash
  docker tag myfirstapp:$IMAGE_TAG myfirstapp:latest
  ```

</details>

<details>
<summary>Scan the image for vulnerabilities</summary>

Even a correctly-built image can ship a vulnerable OS package or library. [Trivy](https://github.com/aquasecurity/trivy) scans an image's OS packages and language dependencies against known CVE databases.

```bash
docker run --rm -v /var/run/docker.sock:/var/run/docker.sock -v trivy-cache:/root/.cache aquasec/trivy image myfirstapp:$IMAGE_TAG
```

- Look at the severities reported (`LOW`, `MEDIUM`, `HIGH`, `CRITICAL`).
- Try scanning a much larger base image for comparison, e.g. `docker run --rm -v trivy-cache:/root/.cache aquasec/trivy image ubuntu:22.04` vs `alpine:latest` - this is the security payoff of the "minimize the attack surface" work from [11-security-networking](11-security-networking.md).

> :bulb: In a real pipeline you would pass `--exit-code 1 --severity HIGH,CRITICAL` (or similar) so the pipeline fails the build when serious vulnerabilities are found, instead of only reporting them.

</details>

<details>
<summary>Put it together in a pipeline definition</summary>

Here is a minimal GitHub Actions workflow that runs the same steps you just did by hand, on every push. It's checked into this repo as [.github/workflows/docker-ci.yml](../.github/workflows/docker-ci.yml) - the lint, build and scan steps run for real whenever `labs/cicd-pipeline/` changes on the `advanced-docker` branch; the login/push steps are switched off (`if: false`) since this training repo has no registry credentials configured.

```yaml
name: docker-ci

on:
  push:
    branches:
      - advanced-docker
    paths:
      - "labs/cicd-pipeline/**"
      - ".github/workflows/docker-ci.yml"
  workflow_dispatch:

jobs:
  build-scan-push:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Lint Dockerfile
        run: docker run --rm -i hadolint/hadolint < labs/cicd-pipeline/Dockerfile

      - name: Build image
        run: docker build -t myfirstapp:${{ github.sha }} labs/cicd-pipeline

      - name: Scan image
        run: docker run --rm -v /var/run/docker.sock:/var/run/docker.sock aquasec/trivy image --exit-code 1 --severity HIGH,CRITICAL myfirstapp:${{ github.sha }}

      - name: Log in to registry
        if: false && github.ref == 'refs/heads/advanced-docker'
        run: echo "${{ secrets.REGISTRY_PASSWORD }}" | docker login -u "${{ secrets.REGISTRY_USER }}" --password-stdin

      - name: Push image
        if: false && github.ref == 'refs/heads/advanced-docker'
        run: docker push myfirstapp:${{ github.sha }}
```

- Notice that credentials never appear in plain text - they come from CI **secrets**, the same principle from [09-multi-container](09-multi-container.md), just provided by the CI platform instead of a local file.
- Notice the pipeline fails fast: lint, then build, then scan-and-fail-on-critical, all before anything is ever pushed anywhere.
- Notice the workflow only triggers on the `advanced-docker` branch - that's deliberate, so it doesn't fire on every branch of every fork.

</details>

<details>
<summary>Run the pipeline yourself, on your own fork</summary>

The workflow above is a real, working GitHub Actions workflow, not just an illustration - you can run it in your own copy of this repository.

- Fork this repository on GitHub (top-right **Fork** button).
- Your fork needs the `advanced-docker` branch, since that's the only branch the workflow triggers on:

  ```bash
  git clone <your-fork-url>
  cd docker-katas
  git checkout advanced-docker
  git push origin advanced-docker
  ```

  > :bulb: If your fork was created *from* `advanced-docker` (i.e. you forked while that branch was checked out, or GitHub copied all branches), you may already have it - `git branch -a` will tell you.

- On GitHub, go to your fork's **Actions** tab. If this is the first workflow run on your fork, GitHub may show a button asking you to confirm/enable Actions - click it.
- Make a small change under `labs/cicd-pipeline/` (e.g. add a comment to the `Dockerfile`), commit, and push it to `advanced-docker`:

  ```bash
  git add labs/cicd-pipeline/Dockerfile
  git commit -m "trigger the pipeline"
  git push origin advanced-docker
  ```

- Back on the **Actions** tab, you should see a `docker-ci` run start automatically. Open it and watch the `Lint Dockerfile`, `Build image` and `Scan image` steps run in order.
- Prefer not to change any files? Use the **Run workflow** button on the `docker-ci` workflow page instead (enabled by the `workflow_dispatch` trigger) - this runs the pipeline on demand, without needing a new commit.

> :bulb: The `Log in to registry` and `Push image` steps are intentionally disabled (`if: false`). If you want to see a real push succeed, add a `REGISTRY_USER` and `REGISTRY_PASSWORD` secret to your fork (**Settings → Secrets and variables → Actions**) and remove `false &&` from both `if:` conditions.

</details>

### Clean up

```bash
docker image rm myfirstapp:latest myfirstapp:$IMAGE_TAG 2>/dev/null

docker volume rm trivy-cache 2>/dev/null
```
