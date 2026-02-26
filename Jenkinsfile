pipeline {
    agent any

    options {
        skipDefaultCheckout()
        skipStagesAfterUnstable()
    }

    stages {

        stage('Clean workspace') {
            steps {
                deleteDir()
            }
        }

        stage('Checkout app code') {
            steps {
                dir('app') {
                    git branch: 'master', url: 'https://github.com/AlfredoVG77/todo-list-aws-app-multibranch.git'
                }
                stash name: 'app-code', includes: 'app/**'
            }
        }

        stage('Checkout settings') {
            steps {
                dir('settings') {
                    git branch: 'master', url: 'https://github.com/AlfredoVG77/todo-list-aws-settings-multibranch.git'
                }
                stash name: 'settings-code', includes: 'settings/**'
            }
        }

        stage('Deploy to Production') {
            steps {
                catchError(buildResult: 'FAILURE', stageResult: 'FAILURE') {

                    unstash 'app-code'
                    unstash 'settings-code'

                    sh '''
                        . /var/lib/jenkins/venv/bin/activate

                        echo "=== SAM BUILD ==="
                        sam build --template-file app/template.yaml

                        echo "=== SAM VALIDATE ==="
                        sam validate --template-file app/template.yaml --region us-east-1

                        echo "=== SAM DEPLOY TO PRODUCTION ==="
                        sam deploy --stack-name todo-list-aws-production --region us-east-1 --capabilities CAPABILITY_IAM --resolve-s3 --s3-prefix todo-list-aws --parameter-overrides Stage=production --no-fail-on-empty-changeset --no-confirm-changeset || true
                    '''
                }
            }
        }

        stage('API Tests') {
            steps {
                unstash 'app-code'
                unstash 'settings-code'

                sh '''
                    BASE_URL=$(aws cloudformation describe-stacks --stack-name todo-list-aws-production --query "Stacks[0].Outputs[?OutputKey=='BaseUrlApi'].OutputValue" --region us-east-1 --output text)

                    . /var/lib/jenkins/venv/bin/activate
                    export BASE_URL="$BASE_URL"

                    pytest app/test/integration/todoApiTest.py -k 'test_api_listtodos or test_api_gettodo' --junitxml=api-tests.xml -q --disable-warnings --maxfail=1
                '''

                junit 'api-tests.xml'
            }
        }
    }
}
