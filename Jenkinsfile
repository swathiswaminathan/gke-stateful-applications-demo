/*
Copyright 2018 Google LLC

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

    https://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.

*/

// The declarative agent is defined in yaml.  It was previously possible to
// define containerTemplate but that has been deprecated in favor of the yaml
// format
// Reference: https://github.com/jenkinsci/kubernetes-plugin
pipeline {
  agent {
    kubernetes {
      label 'k8s-infra'
      defaultContainer 'jnlp'
      yaml """
apiVersion: v1
kind: Pod
metadata:
  labels:
    jenkins: build-node
spec:
  containers:
  - name: k8s-node
    image: gcr.io/pso-helmsman-cicd/jenkins-k8s-node:1.1.0
    imagePullPolicy: Always
    command:
    - cat
    tty: true
    volumeMounts:
    # Mount the docker.sock file so we can communicate with the local docker
    # daemon
    - name: docker-sock-volume
      mountPath: /var/run/docker.sock
    # Mount the local docker binary
    - name: docker-bin-volume
      mountPath: /usr/bin/docker
    # Mount the dev service account key
    - name: dev-key
      mountPath: /home/jenkins/dev
  volumes:
  - name: docker-sock-volume
    hostPath:
      path: /var/run/docker.sock
  - name: docker-bin-volume
    hostPath:
      path: /usr/bin/docker
  # Create a volume that contains the dev json key that was saved as a secret
  - name: dev-key
    secret:
      secretName: jenkins-deploy-dev-infra
"""
    }
  }

  environment {
    CASSANDRA_VERSION = "3.11.3"
    REV = "v${env.BUILD_NUMBER}"
    IMAGE_TAG = "${CASSANDRA_VERSION}-${REV}"
    APP_NAME='cassandra'
    MANIFEST_FILE='manifests/cassandra-statefulset.yaml'
    GOOGLE_APPLICATION_CREDENTIALS    = '/home/jenkins/dev/jenkins-deploy-dev-infra.json'
  }

  stages {

    // Run our various linters against the project
    stage('Lint') {
      steps {
        container('k8s-node') {
           sh "make all"
        }
      }
    }

    // Setup the GCE access for Jenkins test run
    stage('Setup') {
      steps {
       container('k8s-node') {
          script {
                // env.CLUSTER_ZONE will need to be updated to match the
                // ZONE in the jenkins.propeties file
                env.CLUSTER_ZONE = "${CLUSTER_ZONE}"
                // env.PROJECT_ID will need to be updated to match your GCP
                // development project id
                env.PROJECT_ID = "${PROJECT_ID}"
                env.REGION = "${REGION}"
                def shortCommit = sh ( returnStdout: true, script: 'git rev-parse HEAD | cut -c 1-6').trim()
                env.CLUSTER_NAME = "mycsharp-${shortCommit}"
                env.KEYFILE = GOOGLE_APPLICATION_CREDENTIALS
            }
          // Setup gcloud service account access
          sh "gcloud auth activate-service-account --key-file=${env.KEYFILE}"
          sh "gcloud config set compute/zone ${env.CLUSTER_ZONE}"
          sh "gcloud config set core/project ${env.PROJECT_ID}"
          sh "gcloud config set container/cluster ${env.CLUSTER_NAME}"
          sh "gcloud config set compute/region ${env.REGION}"
         }
        }
    }

   // Use Cloud Build to build our two containers
   stage('Build Containers') {
      steps {
        container('k8s-node') {
           dir ('container') {
              sh 'gcloud builds submit --config=cloudbuild.yaml --substitutions=_CASSANDRA_VERSION=${CASSANDRA_VERSION},_REV_=${REV} .'
           }
        }
      }
    }

    // Update the manifest and build the Cassandra Cluster
    stage('Create') {
      steps {
        container('k8s-node') {
           timeout(time: 20, unit: 'MINUTES') {
             // update the cassandra image tag
             sh "./update_image_tag.sh ${PROJECT_ID} ${APP_NAME} ${IMAGE_TAG} ${MANIFEST_FILE}"
             sh "make create CLUSTER_NAME=${env.CLUSTER_NAME}"
             sh "sleep 360"
             // TODO add condition to test with `gcloud container clusters list --filter name=${env.CLUSTER_NAME} --format=value(status)` != "RUNNING" ]`
          }
        }
      }
    }

    // Validate the Cassandra Cluster
    stage('Validate') {
      steps {
        container('k8s-node') {
          script {
            for (int i = 0; i < 3; i++) {
               sh "make validate CLUSTER_NAME=${env.CLUSTER_NAME}"
            }
          }
        }
      }
    }
  }

  // Tear down everything
  post {
    always {
      container('k8s-node') {
        sh "make delete CLUSTER_NAME=${env.CLUSTER_NAME}"
        sh 'gcloud auth revoke'
      }
    }
  }
}
