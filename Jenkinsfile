currentBuild.rawBuild.project.description = 'Pipeline for building and publishing Taiko CI Docker images'

pipeline {
  agent { label 'docker-agent-general' }

  options {
    buildDiscarder logRotator(
      artifactDaysToKeepStr: '',
      artifactNumToKeepStr: '',
      daysToKeepStr: '',
      numToKeepStr: '5'
    )
    parallelsAlwaysFailFast()
    skipDefaultCheckout true
  }

  parameters {
    string(
      name: 'dockerCredentials',
      defaultValue: 'docker-hub-creds',
      description: 'Jenkins credentials ID for Docker Hub',
      trim: true
    )
    string(
      name: 'dockerTag',
      defaultValue: '',
      description: 'Docker image tag',
      trim: true
    )
  }

  environment {
    DOCKER_REPO = "t0mmili/taiko-ci"
  }

  stages {
    stage('Pre-check') {
      agent any
      when {
        anyOf {
          equals expected: '', actual: dockerCredentials
          equals expected: '', actual: dockerTag
        }
      }
      steps {
        error 'One or more required job parameters are empty.'
      }
      post {
        cleanup {
          cleanWs()
        }
      }
    }
    stage('Checkout') {
      steps {
        checkout scm
      }
    }
    stage('Docker login') {
      steps {
        withCredentials([usernamePassword(
          credentialsId: dockerCredentials,
          usernameVariable: 'DOCKER_USER',
          passwordVariable: 'DOCKER_PASS'
        )]) {
          sh '''
            echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
          '''
        }
      }
    }
    stage('Create buildx builder') {
      steps {
        sh '''
          docker buildx create \
            --name jenkins-builder \
            --driver docker-container \
            --driver-opt image=moby/buildkit:v0.15.1 \
            --use || docker buildx use jenkins-builder
          docker buildx inspect --bootstrap
        '''
      }
    }
    stage('Build image') {
      parallel {
        stage('bun') {
          steps {
            sh '''
              docker buildx build \
                --platform linux/amd64 \
                --progress plain \
                -f docker/Dockerfile.bun \
                -t ${DOCKER_REPO}:${dockerTag}-bun \
                --output type=docker,dest=image-bun.tar,compression=gzip \
                .
            '''
          }
          post {
            success {
              stash includes: 'image-bun.tar', name: 'docker-image-bun'
            }
          }
        }
        stage('npm') {
          steps {
            sh '''
              docker buildx build \
                --platform linux/amd64 \
                --progress plain \
                -f docker/Dockerfile.npm \
                -t ${DOCKER_REPO}:${dockerTag}-npm \
                --output type=docker,dest=image-npm.tar,compression=gzip \
                .
            '''
          }
          post {
            success {
              stash includes: 'image-npm.tar', name: 'docker-image-npm'
            }
          }
        }
      }
    }
    stage('Scan image') {
      parallel {
        stage('bun') {
          agent { label 'docker-agent-trivy' }
          steps {
            unstash 'docker-image-bun'

            sh '''
              trivy image \
                --input image-bun.tar \
                --format json \
                --output trivy-results-bun.json \
                --severity HIGH,CRITICAL \
                --exit-code 0 \
                --no-progress
            '''
          }
          post {
            always {
              archiveArtifacts artifacts: 'trivy-results-bun.json', fingerprint: true
              recordIssues(
                enabledForFailure: true,
                qualityGates: [
                  // Fail if even ONE NEW Critical (Error) is introduced
                  [criticality: 'FAILURE', integerThreshold: 1, type: 'NEW_ERROR'],
                  // Fail if > than 5 NEW Highs (High) are introduced
                  [criticality: 'FAILURE', integerThreshold: 5, type: 'NEW_HIGH'],
                  // Ustable if there are any TOTAL Criticals (Error)
                  [criticality: 'UNSTABLE', integerThreshold: 1, type: 'TOTAL_ERROR']
                ],
                tools: [trivy(id: 'trivy-bun', pattern: 'trivy-results-bun.json')]
              )
            }
          }
        }
        stage('npm') {
          agent { label 'docker-agent-trivy' }
          steps {
            unstash 'docker-image-npm'

            sh '''
              trivy image \
                --input image-npm.tar \
                --format json \
                --output trivy-results-npm.json \
                --severity HIGH,CRITICAL \
                --exit-code 0 \
                --no-progress
            '''
          }
          post {
            always {
              archiveArtifacts artifacts: 'trivy-results-npm.json', fingerprint: true
              recordIssues(
                enabledForFailure: true,
                qualityGates: [
                  // Fail if even ONE NEW Critical (Error) is introduced
                  [criticality: 'FAILURE', integerThreshold: 1, type: 'NEW_ERROR'],
                  // Fail if > than 5 NEW Highs (High) are introduced
                  [criticality: 'FAILURE', integerThreshold: 5, type: 'NEW_HIGH'],
                  // Ustable if there are any TOTAL Criticals (Error)
                  [criticality: 'UNSTABLE', integerThreshold: 1, type: 'TOTAL_ERROR']
                ],
                tools: [trivy(id: 'trivy-npm', pattern: 'trivy-results-npm.json')]
              )
            }
          }
        }
      }
    }
    stage('Push image') {
      parallel {
        stage('bun') {
          steps {
            sh '''
              docker buildx build \
                --platform linux/amd64,linux/arm64 \
                --sbom=true \
                -f docker/Dockerfile.bun \
                -t ${DOCKER_REPO}:${dockerTag}-bun \
                -t ${DOCKER_REPO}:latest-bun \
                --push \
                .
            '''
          }
        }
        stage('npm') {
          steps {
            sh '''
              docker buildx build \
                --platform linux/amd64,linux/arm64 \
                --sbom=true \
                -f docker/Dockerfile.npm \
                -t ${DOCKER_REPO}:${dockerTag}-npm \
                -t ${DOCKER_REPO}:latest-npm \
                --push \
                .
            '''
          }
        }
      }
    }
  }

  post {
    always {
      sh '''
        docker buildx rm jenkins-builder || true
        docker logout || true
      '''
      cleanWs()
    }
    success {
        echo """
Images pushed successfully:
 - ${DOCKER_REPO}:${dockerTag}-bun
 - ${DOCKER_REPO}:${dockerTag}-npm
"""
    }
  }
}
