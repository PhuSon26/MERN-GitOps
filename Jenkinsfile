pipeline {
    agent {
        label 'Jenkins-Agent'
    }

    options {
        disableConcurrentBuilds()
    }

    parameters {
        string(
            name: 'IMAGE_TAG',
            defaultValue: '',
            description: 'Docker image tag built and pushed by mern-ci'
        )
    }

    environment {
        DOCKER_USER = 'phuson0206'
        GITOPS_BRANCH = 'main'
    }

    stages {
        stage('Cleanup Workspace') {
            steps {
                cleanWs()
            }
        }

        stage('Checkout GitOps Repository') {
            steps {
                checkout scm

                sh '''
                    echo "GitOps repository checked out"
                    git status --short
                    ls -la k8s/
                '''
            }
        }

        stage('Verify: Docker Image Tag') {
            steps {
                script {
                    if (!params.IMAGE_TAG?.trim()) {
                        error('IMAGE_TAG parameter is required')
                    }

                    echo "IMAGE_TAG: ${params.IMAGE_TAG}"
                }
            }
        }

        stage('Update Deployment Image Tags') {
            steps {
                sh '''
                    set -eu

                    BACKEND_FILE="k8s/backend-deployment.yaml"
                    FRONTEND_FILE="k8s/frontend-deployment.yaml"

                    test -f "$BACKEND_FILE"
                    test -f "$FRONTEND_FILE"

                    sed -i "s|image: ${DOCKER_USER}/mern-application-backend:.*|image: ${DOCKER_USER}/mern-application-backend:${IMAGE_TAG}|" "$BACKEND_FILE"

                    sed -i "s|image: ${DOCKER_USER}/mern-application-frontend:.*|image: ${DOCKER_USER}/mern-application-frontend:${IMAGE_TAG}|" "$FRONTEND_FILE"

                    echo "Backend image:"
                    grep 'image:' "$BACKEND_FILE"

                    echo "Frontend image:"
                    grep 'image:' "$FRONTEND_FILE"
                '''
            }
        }

        stage('Commit and Push GitOps') {
            steps {
                withCredentials([
                    gitUsernamePassword(
                        credentialsId: 'github-gitops',
                        gitToolName: 'Default'
                    )
                ]) {
                    sh '''
                        set -eu

                        git config user.name "PhuSon26"
                        git config user.email "leson2185@gmail.com"

                        git add \
                            k8s/backend-deployment.yaml \
                            k8s/frontend-deployment.yaml

                        if git diff --cached --quiet; then
                            echo "No manifest changes to commit"
                        else
                            git commit -m "deploy: update MERN images to ${IMAGE_TAG}"
                        fi

                        git push origin HEAD:${GITOPS_BRANCH}
                    '''
                }
            }
        }
    }

    post {
        success {
            script {
                def message = """
:rocket: *MERN CD SUCCESS*
*Job:* ${env.JOB_NAME}
*Build:* #${env.BUILD_NUMBER}
*Image Tag:* ${params.IMAGE_TAG}
*Status:* GitOps manifests updated successfully
*Build URL:* ${env.BUILD_URL}
""".stripIndent().trim()

                try {
                    withCredentials([
                        string(
                            credentialsId: 'slack-webhook',
                            variable: 'SLACK_WEBHOOK_URL'
                        )
                    ]) {
                        writeFile(
                            file: 'slack-payload.json',
                            text: groovy.json.JsonOutput.toJson([text: message])
                        )

                        sh '''
                            set +x
                            curl --fail --silent --show-error \
                              --connect-timeout 10 \
                              --max-time 20 \
                              -X POST \
                              -H 'Content-Type: application/json' \
                              --data-binary @slack-payload.json \
                              "$SLACK_WEBHOOK_URL"
                        '''
                    }
                } catch (err) {
                    echo "Slack notification failed: ${err.getMessage()}"
                }
            }
        }

        failure {
            script {
                def message = """
:rotating_light: *MERN CD FAILED*
*Job:* ${env.JOB_NAME}
*Build:* #${env.BUILD_NUMBER}
*Image Tag:* ${params.IMAGE_TAG}
*Build URL:* ${env.BUILD_URL}
""".stripIndent().trim()

                try {
                    withCredentials([
                        string(
                            credentialsId: 'slack-webhook',
                            variable: 'SLACK_WEBHOOK_URL'
                        )
                    ]) {
                        writeFile(
                            file: 'slack-payload.json',
                            text: groovy.json.JsonOutput.toJson([text: message])
                        )

                        sh '''
                            set +x
                            curl --fail --silent --show-error \
                              --connect-timeout 10 \
                              --max-time 20 \
                              -X POST \
                              -H 'Content-Type: application/json' \
                              --data-binary @slack-payload.json \
                              "$SLACK_WEBHOOK_URL"
                        '''
                    }
                } catch (err) {
                    echo "Slack notification failed: ${err.getMessage()}"
                }
            }
        }

        always {
            echo "MERN CD pipeline finished."
        }
    }
}
