pipeline {
    agent any

    tools {
        nodejs 'NodeJS-18'
    }

    environment {
        AWS_REGION = 'us-east-1'
        SNYK_TOKEN = credentials('SNYK_TOKEN')
        SNYK_ORG = '67615456-3e82-4935-9968-23e1de24cd66'
        SNYK_PROJECT = 'snyk-jenkins-test'
    }

    stages {
        stage('Set AWS Credentials') {
            steps {
                withCredentials([
                    $class: 'AmazonWebServicesCredentialsBinding',
                    credentialsId: 'snyk_cred',
                    accessKeyVariable: 'AWS_ACCESS_KEY_ID',
                    secretKeyVariable: 'AWS_SECRET_ACCESS_KEY'
                ]) {
                    sh '''
                    export AWS_ACCESS_KEY_ID=$AWS_ACCESS_KEY_ID
                    export AWS_SECRET_ACCESS_KEY=$AWS_SECRET_ACCESS_KEY
                    aws sts get-caller-identity
                    '''
                }
            }
        }

        stage('Checkout Code') {
            steps {
                script {
                    echo "Checking out source code from GitHub..."
                    checkout([$class: 'GitSCM',
                        branches: [[name: '*/main']],
                        userRemoteConfigs: [[url: 'https://github.com/tiqsclass6/synk-jenkins']]
                    ])
                    echo "Code checkout successful."
                    sh 'ls -la'
                }
            }
        }

        stage('Install Snyk') {
            steps {
                script {
                    echo "Installing Snyk..."
                    sh 'npm install -g snyk snyk-to-sarif'
                    sh 'snyk --version'
                }
            }
        }

        stage('Update Dependencies') {
            steps {
                script {
                    echo "Checking for outdated npm packages..."
                    sh '''
                    npm install -g npm-check-updates
                    ncu -u
                    npm install
                    '''
                }
            }
        }

        stage('Install Dependencies') {
            steps {
                script {
                    echo "Checking for dependencies..."
                    if (fileExists('package.json')) {
                        echo "Node.js project detected. Installing dependencies..."
                        sh 'npm install'
                    } else {
                        echo "No package.json found. Skipping dependency installation."
                    }
                }
            }
        }

        stage('Snyk Scan & Publish to Snyk.io') {
            steps {
                script {
                    echo "Authenticating with Snyk..."
                    sh 'snyk auth $SNYK_TOKEN'

                    echo "Running Snyk security scan on all project files..."
                    sh 'snyk test --all-projects --json > snyk.json || echo "Snyk scan encountered issues, but pipeline continues."'

                    echo "Converting Snyk JSON report to SARIF format..."
                    sh 'snyk-to-sarif < snyk.json > snyk.sarif'

                    echo "Publishing project to Snyk.io..."
                    sh '''
                    snyk monitor --org=$SNYK_ORG --project-name=$SNYK_PROJECT \
                        --project-tags="project-owner:Imported By,environment:Internal,business-criticality:Medium,lifecycle:Sandbox"
                    '''
                }
            }
        }

        stage('Upload SARIF to GitHub Code Scanning') {
            steps {
                withCredentials([string(credentialsId: 'GITHUB_TOKEN', variable: 'GITHUB_AUTH_TOKEN')]) {
                    script {
                        def COMMIT_SHA = sh(script: "git rev-parse HEAD", returnStdout: true).trim()
                        def REF = sh(script: "git symbolic-ref --short HEAD || git rev-parse HEAD", returnStdout: true).trim()

                        echo "Uploading SARIF report to GitHub for commit: ${COMMIT_SHA}, ref: ${REF}"

                        sh """
                        curl -H "Authorization: token $GITHUB_AUTH_TOKEN" \
                             -H "Accept: application/vnd.github.v3+json" \
                             -X POST \
                             --data-binary @- https://api.github.com/repos/tiqsclass6/synk-jenkins/code-scanning/sarifs <<EOF
                        {
                          "commit_sha": "${COMMIT_SHA}",
                          "ref": "refs/heads/${REF}",
                          "sarif": "$(cat snyk.sarif | base64 | tr -d '\\n')"
                        }
                        EOF
                        """
                    }
                }
            }
        }

        stage('Initialize Terraform') {
            steps {
                sh '''
                set -e
                terraform init
                '''
            }
        }

        stage('Validate Terraform') {
            steps {
                sh '''
                set -e
                terraform validate
                '''
            }
        }

        stage('Plan Terraform') {
            steps {
                withCredentials([
                    $class: 'AmazonWebServicesCredentialsBinding',
                    credentialsId: 'snyk_cred',
                    accessKeyVariable: 'AWS_ACCESS_KEY_ID',
                    secretKeyVariable: 'AWS_SECRET_ACCESS_KEY'
                ]) {
                    sh '''
                    set -e
                    export AWS_ACCESS_KEY_ID=$AWS_ACCESS_KEY_ID
                    export AWS_SECRET_ACCESS_KEY=$AWS_SECRET_ACCESS_KEY
                    terraform plan -out=tfplan
                    '''
                }
            }
        }

        stage('Apply Terraform') {
            steps {
                input message: "Approve Terraform Apply?", ok: "Deploy"
                withCredentials([
                    $class: 'AmazonWebServicesCredentialsBinding',
                    credentialsId: 'snyk_cred',
                    accessKeyVariable: 'AWS_ACCESS_KEY_ID',
                    secretKeyVariable: 'AWS_SECRET_ACCESS_KEY'
                ]) {
                    sh '''
                    set -e
                    export AWS_ACCESS_KEY_ID=$AWS_ACCESS_KEY_ID
                    export AWS_SECRET_ACCESS_KEY=$AWS_SECRET_ACCESS_KEY
                    terraform apply -auto-approve tfplan
                    '''
                }
            }
        }

        stage('Deploy') {
            steps {
                echo 'Deploying application...'
            }
        }

        stage('Destroy Terraform') {
            steps {
                input message: "Do you want to destroy the Terraform resources?", ok: "Destroy"
                withCredentials([
                    $class: 'AmazonWebServicesCredentialsBinding',
                    credentialsId: 'snyk_cred',
                    accessKeyVariable: 'AWS_ACCESS_KEY_ID',
                    secretKeyVariable: 'AWS_SECRET_ACCESS_KEY'
                ]) {
                    sh '''
                    set -e
                    export AWS_ACCESS_KEY_ID=$AWS_ACCESS_KEY_ID
                    export AWS_SECRET_ACCESS_KEY=$AWS_SECRET_ACCESS_KEY
                    terraform destroy -auto-approve
                    '''
                }
            }
        }
    }

    post {
        always {
            echo 'Pipeline execution completed.'
        }
    }
}