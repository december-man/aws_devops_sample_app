pipeline {
  agent {
    kubernetes {
      yaml '''
        apiVersion: v1
        kind: Pod
        spec:
          containers:
          - name: nodejs
            image: node:alpine
            command:
            - cat
            tty: true
          - name: kaniko
            image: gcr.io/kaniko-project/executor:debug
            imagePullPolicy: IfNotPresent
            command:
            - /busybox/cat
            tty: true
            volumeMounts:
            - name: aws-secret-volume
              mountPath: root/.aws/
          - name: helm
            image: alpine/helm:latest
            command:
            - /bin/sh
            tty: true
            volumeMounts:
            - name: kubeconfig-volume
              mountPath: /etc/kubeconfig
            - name: ecr-secret-volume
              mountPath: /root/.aws/
            env:
            - name: KUBECONFIG
              value: /etc/kubeconfig/kubeconfig
          restartPolicy: Never
          volumes:
          - name: aws-secret-volume
            secret:
              secretName: aws-secret
          - name: ecr-secret-volume
            secret:
              secretName: ecr-secret
          - name: kubeconfig-volume
            secret:
              secretName: k3s-kubeconfig
        '''
    }
  }
  stages {
    stage('Git Clone') { // Clone github repo with a simple NodeJS web application
      steps {
        container('nodejs') {
          git branch: 'master', url: 'https://github.com/december-man/aws_devops_sample_app.git'
        }
      }
    }
    stage('Install Dependencies') { // npm install
      steps {
        container('nodejs') {
          sh 'npm install ./myapp'
        }
      }
    }
    stage('Run Tests') { // Run some trivial tests
      steps {
        container('nodejs') {
          script {
            sh 'node ./myapp/app.js &'
            sleep 5
            try {
              sh 'cd myapp && npm test'
            } finally {
              sh 'pkill -f "node app.js" || true'
            }
          }
        }
      }
    }
    stage('SonarCloud Analysis') {
      steps {
        script {
          withSonarQubeEnv('SonarCloud') { // Install SonarQube plugin for Jenkins
            env.SONAR_SCANNER = tool 'sonarqube-scanner' // Setup sonar-scanner in Jenkins Tools
            sh '''
            export APP="$(pwd)/myapp"
            echo "SONARSCANNER: $SONAR_SCANNER, APP: $APP"
            ${SONAR_SCANNER}/bin/sonar-scanner \
              -Dsonar.projectKey=december-man_aws_devops_sample_app \
              -Dsonar.organization=december-man \
              -Dsonar.sources=${APP} \
              -Dsonar.host.url=https://sonarcloud.io
            '''
          }
        }
      }
    }
    stage('Build Docker Image Using Kaniko and Push It to AWS ECR') { // Build and push Docker image to Amazon ECR using Kaniko
      steps {
        container('kaniko') {
          sh """ 
              #!/bin/sh
              cd myapp
              /kaniko/executor --context `pwd` --dockerfile `pwd`/Dockerfile --destination 302263083629.dkr.ecr.eu-central-1.amazonaws.com/aws_devops:volha
          """
        }
      }
    }

    stage('Upgrade Application using Helm') { // Run Helm with KUBECONFIG environment to upgrade kubernetes deployment with the new image from ECR
      steps {
        container('helm') {
          sh '''
          helm upgrade myapp ./nodejs -n jenkins --install \
            --set image.repository=302263083629.dkr.ecr.eu-central-1.amazonaws.com/aws_devops \
            --set image.tag=volha
          '''
        }
      }
    }
  }

  post { // NOTE: CONFIGURE SMTP SERVER TO HAVE EMAIL NOTIFICATIONS!
    success {
      mail to: 'aspasjutin@outlook.com',
        subject: "SUCCESS: ${currentBuild.fullDisplayName}",
        body: "The pipeline has succeeded."
    }
    failure {
      mail to: 'aspasjutin@outlook.com',
        subject: "FAILURE: ${currentBuild.fullDisplayName}",
        body: "The pipeline has failed."
    }
  }
}