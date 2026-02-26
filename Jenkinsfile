pipeline {
    agent none

    options {
        skipDefaultCheckout()
        skipStagesAfterUnstable()
    }

    stages {

        /* ============================================================
         * CLEAN WORKSPACE
         * ============================================================ */
        stage('Clean workspace') {
            agent { label 'master' }
            steps {
                deleteDir()
            }
        }

        /* ============================================================
         * CHECKOUT APP CODE (master)
         * ============================================================ */
        stage('Checkout app code') {
            agent { label 'master' }
            steps {
                dir('app') {
                    git branch: 'master', url: 'https://github.com/AlfredoVG77/todo-list-aws-app.git'
                }
                stash name: 'app-code', includes: 'app/**'
            }
        }

        /* ============================================================
         * CHECKOUT SETTINGS (master)
         * ============================================================ */
        stage('Checkout settings') {
            agent { label 'master' }
            steps {
                dir('settings') {
                    git branch: 'master', url: 'https://github.com/AlfredoVG77/todo-list-aws-settings.git'
                }
                stash name: 'settings-code', includes: 'settings/**'
            }
        }

        /* ============================================================
         * DEPLOY TO PRODUCTION
         * ============================================================ */
        stage('Deploy to Production') {
            agent { label 'master' }
            steps {
                catchError(buildResult: 'FAILURE', stageResult: 'FAILURE') {

                    unstash 'app-code'
                    unstash 'settings-code'

                    sh '''
                        
                        echo "=== SAM BUILD ==="
                        sam build --template-file app/template.yaml

                        echo "=== SAM VALIDATE ==="
                        sam validate --template-file app/template.yaml --region us-east-1

                        echo "=== SAM DEPLOY TO PRODUCTION ==="
                        sam deploy \
                            --stack-name todo-list-aws-production --region us-east-1 --capabilities CAPABILITY_IAM --resolve-s3 --s3-prefix todo-list-aws --parameter-overrides Stage=production --no-fail-on-empty-changeset --no-confirm-changeset || true
                    '''
                }
            }
        }

        /* ============================================================
         * API TESTS (agente test)
         * ============================================================ */
        stage('API Tests') {
            agent { label 'test' }
            steps {

                unstash 'app-code'
                unstash 'settings-code'

                sh '''
                    echo "=== Obteniendo URL de la API desde CloudFormation ==="
                    BASE_URL=$(aws cloudformation describe-stacks --stack-name todo-list-aws-production --query "Stacks[0].Outputs[?OutputKey=='BaseUrlApi'].OutputValue" \
                        --region us-east-1 --output text)

                    echo "=== Ejecutando tests de integración ==="

                    . /var/lib/jenkins/venv/bin/activate
                    export BASE_URL="$BASE_URL"

                    pytest app/test/integration/todoApiTest.py -k 'test_api_listtodos or test_api_gettodo' --junitxml=api-tests.xml -q --disable-warnings --maxfail=1
                '''

                junit 'api-tests.xml'
            }
        }
    }
}
