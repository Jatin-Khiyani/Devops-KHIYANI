# GitHub Actions

In this part I created the CI/CD workflows for the project.

The workflows are in:

```
.github/workflows/
├── main.yml
├── build-and-push.yml
└── deploy.yml
```

## main.yml - CI and Sonar

`main.yml` runs on push to `main` and `develop`, and on pull requests.

The first job is `test-backend`.

It does:

1. Checkout the repository.
2. Setup Java 21 with Temurin.
3. Run Maven tests from `TP-1/Backend/Springboot`.

Command used by the workflow:

```
mvn clean verify
```

## 2-1 What are testcontainers?

Testcontainers are Java libraries that start Docker containers during tests.

In this project they are used for integration tests with PostgreSQL. The tests can use a real database container instead of a fake database.

This makes the test closer to the real application.

## SonarCloud

The second job is `Sonar`. It depends on `test-backend`.

It runs:

```
mvn -B verify sonar:sonar
```

The Sonar configuration uses GitHub Secrets:

```
PROJECT_KEY
ORGANIZATION_KEY
SONAR_TOKEN
```

The goal is to check code quality, maintainability and security problems.

## 2-2 For what purpose do we need to use secured variables?

Secured variables are needed because the workflow uses private values.

Examples:

```
DOCKERHUB_USERNAME
DOCKERHUB_TOKEN
SONAR_TOKEN
SSH_PRIVATE_KEY
POSTGRES_PASSWORD
```

These values should not be in the repository. GitHub Secrets allows the pipeline to use them without exposing them in the code.

## build-and-push.yml - Docker images

`build-and-push.yml` runs after `CI devops 2025` completes successfully on `main`.

It logs into DockerHub and builds these images:

```
TP-1/database
TP-1/Backend/Springboot
TP-1/httpd
TP-1/frontend
```

Images pushed:

```
${DOCKERHUB_USERNAME}/tp-devops-database:latest
${DOCKERHUB_USERNAME}/tp-devops-simple-api:latest
${DOCKERHUB_USERNAME}/tp-devops-httpd:latest
${DOCKERHUB_USERNAME}/frontend:latest
```

## 2-3 Why did we put needs: build-and-test-backend on this job?

The Docker image must be published only if the backend builds and tests pass.

If this dependency is not there, GitHub Actions can push an image that contains broken code.

In this repository I split the workflows, so `build-and-push.yml` uses `workflow_run` instead of `needs`. The idea stays the same: DockerHub is updated only after CI is green.

## 2-4 For what purpose do we need to push docker images?

We push Docker images so that the deployment server can pull them.

The image is built by GitHub Actions, pushed to DockerHub, then Ansible pulls the image on the server.

This avoids building on the production server and keeps deployment cleaner.

## deploy.yml - Ansible deployment

`deploy.yml` installs Ansible in the GitHub runner and runs the playbook:

```
my-project/ansible/playbook.yml
```

It writes the SSH private key from GitHub Secrets into a temporary file:

```
private_key.pem
```

Then it executes:

```
ansible-playbook -i my-project/ansible/inventories/setup.yml my-project/ansible/playbook.yml
```

## Split pipelines

I separated the pipeline into different workflows:

1. `main.yml` for test and Sonar.
2. `build-and-push.yml` for Docker images.
3. `deploy.yml` for Ansible deployment.

This makes the pipeline easier to read and avoids publishing images before the tests pass.

