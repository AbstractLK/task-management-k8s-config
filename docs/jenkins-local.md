# Local Jenkins in Docker

Run Jenkins locally with access to the host Docker engine:

```bash
docker network create jenkins
docker volume create jenkins_home
docker run --name jenkins -d --restart=unless-stopped --network jenkins -p 8080:8080 -p 50000:50000 -v jenkins_home:/var/jenkins_home -v /var/run/docker.sock:/var/run/docker.sock jenkins/jenkins:lts
```

Inside the Jenkins container, install:

- Docker CLI
- Git
- Node.js, or use Jenkins agents that include Node.js

Create credentials:

- `dockerhub-creds`: DockerHub username and access token
- `github-creds`: GitHub username and personal access token

The `task-management-app` repository uses one root `Jenkinsfile`. Jenkins builds the frontend, auth service, and task service images, pushes them to DockerHub, then updates the image tags in the `task-management-k8s-config` GitOps repository.
