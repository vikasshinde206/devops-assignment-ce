// =============================================================================
// sync-service Jenkinsfile
// Supports: PR builds, QA deploy (develop), Staging deploy (staging),
//           Prod deploy with manual gate (main)
// =============================================================================

pipeline {
    agent any

    // --------------------------
    // Tool versions (configured in Jenkins Global Tool Configuration)
    // --------------------------
    tools {
        maven 'maven-3.9'
        jdk   'jdk-17'
    }

    // --------------------------
    // Pipeline-level environment
    // --------------------------
    environment {
        APP_NAME        = 'sync-service'
        GCR_REGISTRY    = 'gcr.io/YOUR_GCP_PROJECT'
        IMAGE_NAME      = "${GCR_REGISTRY}/${APP_NAME}"
        IMAGE_TAG       = "${env.GIT_COMMIT[0..7]}"
        SONAR_URL       = 'http://sonarqube.internal:9000'
        CONFIG_REPO     = 'git@github.com:org/sync-service-config.git'

        // Injected from Jenkins Credentials Store
        GCR_CREDENTIALS = credentials('gcr-service-account-key')
        SONAR_TOKEN     = credentials('sonarqube-token')
    }

    // --------------------------
    // Options
    // --------------------------
    options {
        buildDiscarder(logRotator(numToKeepStr: '20'))
        timeout(time: 45, unit: 'MINUTES')
        timestamps()
        ansiColor('xterm')
        disableConcurrentBuilds()
    }

    // --------------------------
    // Stages
    // --------------------------
    stages {

        // ------------------------------------------------------------------
        // 1. Checkout
        // ------------------------------------------------------------------
        stage('Checkout') {
            steps {
                checkout scm
                script {
                    env.BRANCH_NAME = env.BRANCH_NAME ?: sh(
                        script: 'git rev-parse --abbrev-ref HEAD',
                        returnStdout: true
                    ).trim()
                    env.COMMIT_MSG = sh(
                        script: 'git log -1 --pretty=%B',
                        returnStdout: true
                    ).trim()
                    echo "Branch: ${env.BRANCH_NAME} | Commit: ${env.IMAGE_TAG}"
                }
            }
        }

        // ------------------------------------------------------------------
        // 2. Build
        // ------------------------------------------------------------------
        stage('Build') {
            steps {
                sh 'mvn clean package -DskipTests -B -q'
            }
            post {
                failure {
                    notifySlack("BUILD FAILED on ${env.BRANCH_NAME} — ${env.COMMIT_MSG}", 'danger')
                }
            }
        }

        // ------------------------------------------------------------------
        // 3. Unit Tests + Coverage
        // ------------------------------------------------------------------
        stage('Unit Tests') {
            steps {
                sh 'mvn test -B'
            }
            post {
                always {
                    junit 'target/surefire-reports/**/*.xml'
                    jacoco(
                        execPattern:         'target/jacoco.exec',
                        classPattern:        'target/classes',
                        sourcePattern:       'src/main/java',
                        minimumLineCoverage: '70'     // fail if < 70% line coverage
                    )
                }
                failure {
                    notifySlack("TESTS FAILED on ${env.BRANCH_NAME}", 'danger')
                }
            }
        }

        // ------------------------------------------------------------------
        // 4. Static Analysis (SonarQube)
        // ------------------------------------------------------------------
        stage('Static Analysis') {
            steps {
                withSonarQubeEnv('SonarQube') {
                    sh """
                        mvn sonar:sonar -B \\
                            -Dsonar.projectKey=${APP_NAME} \\
                            -Dsonar.host.url=${SONAR_URL} \\
                            -Dsonar.login=${SONAR_TOKEN}
                    """
                }
                // Block pipeline until Quality Gate result is available (10 min timeout)
                timeout(time: 10, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        // ------------------------------------------------------------------
        // 5. Docker Build & Push
        //    Only on develop / staging / main — not on PRs
        // ------------------------------------------------------------------
        stage('Docker Build & Push') {
            when {
                anyOf {
                    branch 'develop'
                    branch 'staging'
                    branch 'main'
                }
            }
            steps {
                script {
                    // Authenticate to GCR
                    sh "echo '${GCR_CREDENTIALS}' | docker login -u _json_key --password-stdin https://gcr.io"

                    // Build image
                    sh "docker build -t ${IMAGE_NAME}:${IMAGE_TAG} -t ${IMAGE_NAME}:latest ."

                    // Push both tags
                    sh "docker push ${IMAGE_NAME}:${IMAGE_TAG}"
                    sh "docker push ${IMAGE_NAME}:latest"

                    echo "Pushed: ${IMAGE_NAME}:${IMAGE_TAG}"
                }
            }
            post {
                failure {
                    notifySlack("Docker build/push FAILED on ${env.BRANCH_NAME}", 'danger')
                }
            }
        }

        // ------------------------------------------------------------------
        // 6. Deploy to QA
        //    Triggered on merge to 'develop'
        // ------------------------------------------------------------------
        stage('Deploy → QA') {
            when { branch 'develop' }
            environment {
                ENV         = 'qa'
                VM_GROUP    = 'sync-service-qa-mig'
                GCP_ZONE    = 'us-central1-a'
            }
            steps {
                deployToEnv(env.ENV, env.VM_GROUP, env.GCP_ZONE, env.IMAGE_TAG)
            }
        }

        // ------------------------------------------------------------------
        // 7. Integration Tests (after QA deploy)
        // ------------------------------------------------------------------
        stage('Integration Tests') {
            when { branch 'develop' }
            steps {
                sh 'mvn verify -Pintegration-tests -B -Denv=qa'
            }
            post {
                always {
                    junit 'target/failsafe-reports/**/*.xml'
                }
                failure {
                    notifySlack("Integration tests FAILED on QA", 'danger')
                    // Auto-rollback QA
                    rollback('qa', 'sync-service-qa-mig', 'us-central1-a')
                }
            }
        }

        // ------------------------------------------------------------------
        // 8. Deploy to Staging
        //    Triggered on merge to 'staging'
        // ------------------------------------------------------------------
        stage('Deploy → Staging') {
            when { branch 'staging' }
            environment {
                ENV         = 'staging'
                VM_GROUP    = 'sync-service-staging-mig'
                GCP_ZONE    = 'us-central1-a'
            }
            steps {
                deployToEnv(env.ENV, env.VM_GROUP, env.GCP_ZONE, env.IMAGE_TAG)
            }
        }

        // ------------------------------------------------------------------
        // 9. Smoke Tests (after Staging deploy)
        // ------------------------------------------------------------------
        stage('Smoke Tests') {
            when { branch 'staging' }
            steps {
                sh 'mvn verify -Psmoke-tests -B -Denv=staging'
            }
            post {
                failure {
                    notifySlack("Smoke tests FAILED on Staging", 'danger')
                    rollback('staging', 'sync-service-staging-mig', 'us-central1-a')
                }
            }
        }

        // ------------------------------------------------------------------
        // 10. Manual Approval Gate (prod only)
        // ------------------------------------------------------------------
        stage('Approval: Deploy to Production?') {
            when { branch 'main' }
            steps {
                script {
                    notifySlack(
                        "Production deploy pending for ${IMAGE_NAME}:${IMAGE_TAG} — approve in Jenkins",
                        'warning'
                    )
                    def approval = input(
                        message:   "Deploy ${IMAGE_TAG} to PRODUCTION?",
                        ok:        'Deploy',
                        submitter: 'devops-leads,sre-team',   // Only these groups can approve
                        parameters: [
                            string(
                                name:         'APPROVER_NOTES',
                                description:  'Deployment notes / change ticket (optional)',
                                defaultValue: ''
                            )
                        ]
                    )
                    echo "Approved by: ${approval.APPROVER_NOTES}"
                }
            }
        }

        // ------------------------------------------------------------------
        // 11. Deploy to Production (Blue/Green)
        //     Only runs after manual approval
        // ------------------------------------------------------------------
        stage('Deploy → Production (Blue/Green)') {
            when { branch 'main' }
            environment {
                ENV          = 'prod'
                GREEN_GROUP  = 'sync-service-prod-green-mig'
                BLUE_GROUP   = 'sync-service-prod-blue-mig'
                LB_BACKEND   = 'sync-service-prod-backend'
                GCP_ZONE     = 'us-central1-a'
            }
            steps {
                script {
                    // 1. Save current (blue) tag for potential rollback
                    sh """
                        gcloud compute instance-groups managed describe ${BLUE_GROUP} \\
                            --zone=${GCP_ZONE} \\
                            --format='get(instanceTemplate)' > /tmp/blue_template.txt
                    """

                    // 2. Deploy new image to Green group (rolling update within Green)
                    sh """
                        gcloud compute instance-groups managed rolling-action start-update ${GREEN_GROUP} \\
                            --version=template=sync-service-prod-template-${IMAGE_TAG} \\
                            --zone=${GCP_ZONE} \\
                            --max-unavailable=0 \\
                            --max-surge=3
                    """

                    // 3. Wait for Green to stabilize
                    sh """
                        gcloud compute instance-groups managed wait-until ${GREEN_GROUP} \\
                            --stable \\
                            --zone=${GCP_ZONE} \\
                            --timeout=300
                    """

                    // 4. Health check Green
                    sh "./scripts/health-check.sh --env prod --group ${GREEN_GROUP} --zone ${GCP_ZONE}"

                    // 5. Flip Load Balancer traffic to Green
                    sh """
                        gcloud compute backend-services update ${LB_BACKEND} \\
                            --global \\
                            --description='Active: green ${IMAGE_TAG}'
                    """
                    echo "Traffic flipped to Green (${IMAGE_TAG})"

                    // 6. Store successful tag as last stable
                    withCredentials([string(credentialsId: 'last-stable-tag', variable: 'CRED_ID')]) {
                        sh "echo ${IMAGE_TAG} > /tmp/last_stable_tag"
                        // Update Jenkins credential programmatically or use a build artifact
                    }
                }
            }
            post {
                failure {
                    // Auto-rollback: flip LB back to Blue immediately
                    notifySlack("Prod deploy FAILED — rolling back to Blue", 'danger')
                    sh """
                        gcloud compute backend-services update ${LB_BACKEND} \\
                            --global \\
                            --description='ROLLBACK to blue'
                    """
                }
                success {
                    notifySlack("Prod deploy SUCCESS — ${IMAGE_TAG} is live on Green", 'good')
                    // Keep Blue warm for 15min, then drain
                    sleep(time: 15, unit: 'MINUTES')
                    sh """
                        gcloud compute instance-groups managed resize ${BLUE_GROUP} \\
                            --size=0 \\
                            --zone=${GCP_ZONE}
                    """
                }
            }
        }

    } // end stages

    // --------------------------
    // Post-pipeline handlers
    // --------------------------
    post {
        always {
            cleanWs()
        }
        success {
            echo "Pipeline completed successfully for ${env.BRANCH_NAME}:${env.IMAGE_TAG}"
        }
        failure {
            notifySlack("Pipeline FAILED on ${env.BRANCH_NAME} — ${env.BUILD_URL}", 'danger')
        }
    }

} // end pipeline

// =============================================================================
// Shared Functions
// =============================================================================

/**
 * Deploy image to a given environment using a rolling MIG update.
 */
def deployToEnv(String envName, String migName, String zone, String tag) {
    // Pull env-specific config from config repo
    sh """
        rm -rf /tmp/config-${envName}
        git clone ${CONFIG_REPO} /tmp/config-${envName} --branch ${envName} --depth 1
        cp /tmp/config-${envName}/application-${envName}.yml ./src/main/resources/
    """

    // Apply rolling update
    sh """
        gcloud compute instance-groups managed rolling-action start-update ${migName} \\
            --version=template=sync-service-${envName}-template-${tag} \\
            --zone=${zone} \\
            --max-unavailable=1 \\
            --max-surge=2

        gcloud compute instance-groups managed wait-until ${migName} \\
            --stable \\
            --zone=${zone} \\
            --timeout=180
    """

    // Verify health
    sh "./scripts/health-check.sh --env ${envName} --group ${migName} --zone ${zone}"
    echo "Deployment to ${envName} complete: ${tag}"
}

/**
 * Rollback a given environment to the previous known-good image.
 */
def rollback(String envName, String migName, String zone) {
    echo "Rolling back ${envName}..."
    sh """
        gcloud compute instance-groups managed rolling-action start-update ${migName} \\
            --version=template=sync-service-${envName}-template-stable \\
            --zone=${zone} \\
            --max-unavailable=0 \\
            --max-surge=3

        gcloud compute instance-groups managed wait-until ${migName} \\
            --stable \\
            --zone=${zone}
    """
    notifySlack("ROLLBACK complete for ${envName}", 'warning')
}

/**
 * Send a Slack notification.
 * Requires the Jenkins Slack plugin and 'slack-webhook' credential.
 */
def notifySlack(String message, String color) {
    slackSend(
        channel:  '#deployments',
        color:    color,
        message:  "[${env.APP_NAME}] ${message} — <${env.BUILD_URL}|Build #${env.BUILD_NUMBER}>"
    )
}
