## A Simple Nodejs Web Application's Full CI/CD pipeline:
  * CI/CD tool: Jenkins
  * Infrastructure: Private k3s cluster
  * Testing Frameworks/Utilities: Mocha, SonarQube (coming soon)
  * Container Registry for application: Amazon ECR
  * Container Builder: Kaniko
  * Deployment Tool: Helm

### Application Description
  - A super simple webserver with exposed 3000 port that says Hello World. Web Application is convenient when it comes to networking/availability tests.
  - Made with `npm init`
  - Application code: `myapp/app.js`
  - Application tests: `myapp/test.js`
  - Contents/dependencies: `myapp/package.json`
  - Dockerfile: `myapp/Dockerfile`

### Usage
  **In order to create the full CI/CD pipeline for basically any simple application similar to `myapp`, one would need to have a fully setup Kubernetes cluster with installed Jenkins, and Helm.**

- More on infrastructure can be found in the corresponding repo: https://github.com/december-man/rsschool-devops-course-infrastrutture/blob/main/README.md
- More on Jenkins configuration can be found in the corresponding repo: https://github.com/december-man/rsschool-devops-course-software/blob/main/README.md

Once you've configured your k3s cluster and Jenkins to have an exposed port with Jenkins UI, you would first need to install a number of Jenkins Plugins. It is pretty convenient to do it from the menu, or add them in JCasC config in your Jenkins setup or jenkins-values.yaml during Jenkins deployment on k8s. The required plugins are:
  * Kubernetes (The most important plugin, that is usually specified in jenkins-values.yaml)
  * Pipeline (Easily create and deploy pipelines in Jenkins)
  * NodeJS
  * AWS ECR/AWS Steps/Amazon Web Services SDK (For AWS interactions in pipeline stages)
  * SonarQube Scanner (integrate SonarQube utility with Jenkins)
  * Git, obviously

After that you can start building your pipeline: 
  * Dashboard -> New Item -> Pipeline. Name it, then proceed.
  * Give it a proper description 
  * Add SCM Polling for changes, Example Schedule: H/5 * * * * (every 5 minutes)
  * Setup Pipeline Definition to be Pipeline Script From SCM (Git): Specify this repository. Branch - `master`, file: `Jenkinsfile`.
  * You may want to change something in the pipeline. In that case I'd suggest you to use `Pipeline Script` mode instead. Just paste the jenkinsfile there and modify it to comply with your needs.
  * Before triggering the pipeline, notice that you have to add necessary secrets to your k8s cluster:
    - secretName: **aws-secret (~/.aws/credentials file)**
    - secretName: **ecr-secret (docker secret with aws ecr get-login-password output)**
    - secretName: **k3s-kubeconfig (your k3s kubeconfig for Helm deployment stage)**
  * Finally, you can either just wait until polling will trigger the pipeline or just trigger it yourself.
  * Happy HELMing!

### Additional Notes
- This repository acts as a Helm Repository, hence an `index.yaml` file in the root.
- **Nodejs is a Helm Chart**. Lookup values.yaml for configuration.
- In order to add email notifications, remove the active `post {}` section and uncomment the lower one. But before you do that, make sure to set up an SMTP server in Jenkins (Manage Jenkins -> System -> E-mail Notification).
- Image tag (container build version) is hardcoded in Jenkins Pipeline. It can be easily added to environment to remove the need in constantly updating the image tag in kaniko/helm stages.
- There is a file called `successful_pipeline_log_example` where you can look at the extracted log file from Jenkins which demonstrates a successful pipeline run.