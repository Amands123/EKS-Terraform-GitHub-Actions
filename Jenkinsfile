pipeline {

    agent any

    options {
        timestamps()
        disableConcurrentBuilds()
        buildDiscarder(logRotator(
            numToKeepStr: '20',
            artifactNumToKeepStr: '10'
        ))
        timeout(time: 60, unit: 'MINUTES')
    }

    parameters {

        choice(
            name: 'Environment',
            choices: ['dev', 'qa', 'prod'],
            description: 'Environment'
        )

        choice(
            name: 'Terraform_Action',
            choices: ['plan', 'apply', 'destroy'],
            description: 'Terraform Action'
        )
    }

    environment {
        AWS_REGION = 'us-east-1'
        TF_DIR     = 'eks'
    }

    stages {

        stage('Clean Workspace') {
            steps {
                cleanWs()
            }
        }

        stage('Checkout Code') {
            steps {
                git branch: 'master',
                    url: 'https://github.com/Amands123/EKS-Terraform-GitHub-Actions.git'
            }
        }

        stage('Verify Tools') {
            steps {
                sh '''
                terraform version
                aws --version
                '''
            }
        }

        stage('Verify AWS Access') {
            steps {
                withAWS(credentials: 'aws-creds', region: "${AWS_REGION}") {
                    sh '''
                    aws sts get-caller-identity
                    '''
                }
            }
        }

        stage('Terraform Init') {
            steps {
                withAWS(credentials: 'aws-creds', region: "${AWS_REGION}") {

                    sh '''
                    rm -rf ${TF_DIR}/.terraform

                    terraform -chdir=${TF_DIR} init \
                      -reconfigure
                    '''
                }
            }
        }

        stage('Terraform Format Check') {
            steps {
                sh '''
                terraform -chdir=${TF_DIR} fmt -check -recursive
                '''
            }
        }

        stage('Terraform Validate') {
            steps {
                withAWS(credentials: 'aws-creds', region: "${AWS_REGION}") {

                    sh '''
                    terraform -chdir=${TF_DIR} validate
                    '''
                }
            }
        }

        stage('Terraform Plan') {

            when {
                anyOf {
                    expression { params.Terraform_Action == 'plan' }
                    expression { params.Terraform_Action == 'apply' }
                }
            }

            steps {

                withAWS(credentials: 'aws-creds', region: "${AWS_REGION}") {

                    sh """
                    terraform -chdir=${TF_DIR} plan \
                    -var-file=${params.Environment}.tfvars \
                    -out=tfplan
                    """

                }

                archiveArtifacts artifacts: "${TF_DIR}/tfplan",
                fingerprint: true
            }
        }

        stage('Approval For Apply') {

            when {
                expression {
                    params.Terraform_Action == 'apply'
                }
            }

            steps {

                input(
                    message: "Apply Terraform changes to ${params.Environment} ?",
                    ok: "Deploy"
                )

            }
        }

        stage('Terraform Apply') {

            when {
                expression {
                    params.Terraform_Action == 'apply'
                }
            }

            steps {

                withAWS(credentials: 'aws-creds', region: "${AWS_REGION}") {

                    sh '''
                    terraform -chdir=${TF_DIR} apply \
                    -auto-approve tfplan
                    '''

                }
            }
        }

        stage('Approval For Destroy') {

            when {
                expression {
                    params.Terraform_Action == 'destroy'
                }
            }

            steps {

                input(
                    message: "WARNING: Destroy Infrastructure?",
                    ok: "Destroy"
                )

            }
        }

        stage('Terraform Destroy') {

            when {
                expression {
                    params.Terraform_Action == 'destroy'
                }
            }

            steps {

                withAWS(credentials: 'aws-creds', region: "${AWS_REGION}") {

                    sh """
                    terraform -chdir=${TF_DIR} destroy \
                    -var-file=${params.Environment}.tfvars \
                    -auto-approve
                    """

                }
            }
        }
    }

    post {

        success {
            echo "Pipeline completed successfully."
        }

        failure {
            echo "Pipeline failed. Review logs."
        }

        always {
            archiveArtifacts artifacts: '**/*.log',
            allowEmptyArchive: true
        }
    }
}
