pipeline {
  agent {
    kubernetes {
      inheritFrom 'default'
      label 'hextris-agent'
      yaml """
apiVersion: v1
kind: Pod
spec:
  containers:
    - name: kaniko
      image: gcr.io/kaniko-project/executor:debug
      command: ['sleep']
      args: ['99d']
      tty: true
      volumeMounts:
        - name: docker-config
          mountPath: /kaniko/.docker
    - name: skopeo
      image: quay.io/skopeo/stable:latest
      command: ['sleep']
      args: ['99d']
      tty: true
    - name: helm
      image: alpine/helm:3.14.0
      command: ['sleep']
      args: ['99d']
      tty: true
    - name: kubectl
      image: bitnami/kubectl:latest
      command: ['sleep']
      args: ['99d']
      tty: true
  volumes:
    - name: docker-config
      emptyDir: {}
"""
    }
  }
  environment {
    REGISTRY = "registry.digitalocean.com/ruhi-creg"
    IMAGE_NAME = "hextris-app"
  }
  stages {
    stage('Checkout') {
      steps {
        checkout([
          $class: 'GitSCM',
          branches: [[name: '*/main']],
          userRemoteConfigs: [[
            url: 'https://github.com/AlRuh-ux/devops-assignment.git'
          ]]
        ])
        script {
          env.GIT_COMMIT_SHORT = sh(
            script: "git rev-parse --short=7 HEAD",
            returnStdout: true
          ).trim()
          echo "Repository checked out successfully"
          echo "Git commit: ${env.GIT_COMMIT_SHORT}"
        }
      }
    }

    stage('Check if Image Exists') {
      steps {
        container('skopeo') {
          script {
            withCredentials([string(credentialsId: 'do-api-token', variable: 'DO_API_TOKEN')]) {
              def imageExists = sh(
                script: '''
                  set +x

                  # Try to inspect the image with the specific tag
                  # If it exists, skopeo will succeed; if not, it will fail
                  if skopeo inspect --creds "doctl:${DO_API_TOKEN}" \
                     docker://${REGISTRY}/${IMAGE_NAME}:${GIT_COMMIT_SHORT} > /dev/null 2>&1; then
                    echo "EXISTS"
                  else
                    echo "NOT_FOUND"
                  fi
                ''',
                returnStdout: true
              ).trim()

              if (imageExists == "EXISTS") {
                env.SKIP_BUILD = "true"
                echo "Image already exists for commit ${env.GIT_COMMIT_SHORT} - skipping build"
              } else {
                env.SKIP_BUILD = "false"
                echo "Image not found for commit ${env.GIT_COMMIT_SHORT} - will build"
              }
            }
          }
        }
      }
    }

    stage('Build and Push Image') {
      when {
        environment name: 'SKIP_BUILD', value: 'false'
      }
      steps {
        container('kaniko') {
          dir('hextris-app') {
            withCredentials([string(credentialsId: 'do-api-token', variable: 'DO_API_TOKEN')]) {
              sh '''
                set +x

                # Create Docker config for DigitalOcean registry
                AUTH=$(printf "doctl:%s" "${DO_API_TOKEN}" | base64 -w 0)
                cat > /kaniko/.docker/config.json <<EOF
{
  "auths": {
    "registry.digitalocean.com": {
      "auth": "${AUTH}"
    }
  }
}
EOF

                echo "Docker config created successfully"

                # Build and push with Kaniko (both commit SHA and latest tags)
                set -x
                /kaniko/executor \
                  --context . \
                  --dockerfile Dockerfile \
                  --destination ${REGISTRY}/${IMAGE_NAME}:${GIT_COMMIT_SHORT} \
                  --destination ${REGISTRY}/${IMAGE_NAME}:latest \
                  --cache=true \
                  --cleanup
              '''
            }
          }
        }
      }
    }
  }
  post {
    success {
      script {
        if (env.SKIP_BUILD == "true") {
          echo "Build skipped - image already exists: $REGISTRY/$IMAGE_NAME:${env.GIT_COMMIT_SHORT}"
        } else {
          echo "Image built and pushed successfully: $REGISTRY/$IMAGE_NAME:${env.GIT_COMMIT_SHORT} (also tagged as latest)"
        }
      }
    }
    failure {
      echo "Build or push failed. Check the console logs for details."
    }
  }
}
