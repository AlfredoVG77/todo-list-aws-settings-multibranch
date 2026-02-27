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
                    git branch: 'develop', url: 'https://github.com/AlfredoVG77/todo-list-aws-app-multibranch.git'
                }
                stash name: 'app-code', includes: 'app/**'
            }
        }

        stage('Checkout settings') {
            steps {
                dir('settings') {
                    git branch: 'develop', url: 'https://github.com/AlfredoVG77/todo-list-aws-settings-multibranch.git'
                }
                stash name: 'settings-code', includes: 'settings/**'
            }
        }

        stage('Static Test') {
            steps {
                catchError(buildResult: 'UNSTABLE', stageResult: 'FAILURE') {

                    unstash 'app-code'
                    unstash 'settings-code'

                    sh '''
                        . /var/lib/jenkins/venv/bin/activate
                        python -m flake8 --exit-zero --format=pylint app/src > flake8.out
                    '''

                    recordIssues(tools: [flake8(pattern: 'flake8.out')])

                    sh '''
                        . /var/lib/jenkins/venv/bin/activate
                        python -m bandit --exit-zero -r app/src -f custom -o bandit.out --msg-template "{abspath}:{line}: [{test_id}] {msg}"
                    '''

                    recordIssues(tools: [pyLint(pattern: 'bandit.out')])
                }
            }
        }

        stage('Deploy to Staging') {
            steps {
                unstash 'app-code'
                unstash 'settings-code'

                sh '''
                    . /var/lib/jenkins/venv/bin/activate

                    echo "=== SAM BUILD ==="
                    sam build --template-file app/template.yaml

                    echo "=== SAM VALIDATE ==="
                    sam validate --template-file app/template.yaml --region us-east-1

                    echo "=== SAM DEPLOY TO STAGING ==="
                    sam deploy \
                        --stack-name todo-list-aws-staging --region us-east-1 --capabilities CAPABILITY_IAM --resolve-s3 --s3-prefix todo-list-aws --parameter-overrides Stage=staging --no-fail-on-empty-changeset --no-confirm-changeset || true
                '''
            }
        }

        stage('API Tests') {
            steps {
                unstash 'app-code'
                unstash 'settings-code'

                sh '''
                    BASE_URL=$(aws cloudformation describe-stacks --stack-name todo-list-aws-staging --query "Stacks[0].Outputs[?OutputKey=='BaseUrlApi'].OutputValue" --region us-east-1 --output text)

                    . /var/lib/jenkins/venv/bin/activate
                    export BASE_URL="$BASE_URL"

                    pytest app/test/integration/todoApiTest.py --junitxml=api-tests.xml -q --disable-warnings --maxfail=1
                '''

                junit 'api-tests.xml'
            }
        }

stage('Promote app to master') {
    steps {
        unstash 'app-code'
        dir('app') {
            withCredentials([string(credentialsId: 'github-token', variable: 'GITHUB_TOKEN')]) {
                sh '''
                    git config user.email "jenkins@ci"
                    git config user.name "Jenkins CI"

                    git checkout develop
                    git pull origin develop

                    git checkout master
                    git pull origin master

                    # Evitar conflicto del Jenkinsfile
                    git checkout origin/master -- Jenkinsfile

                    git merge origin/develop --no-edit || true

                    git remote set-url origin https://AlfredoVG77:${GITHUB_TOKEN}@github.com/AlfredoVG77/todo-list-aws-app-multibranch.git
                    git push origin master
                '''
            }
        }
    }
}
stage('Promote settings to master') {
    steps {
        unstash 'settings-code'
        dir('settings') {
            withCredentials([string(credentialsId: 'github-token', variable: 'GITHUB_TOKEN')]) {
                sh '''
                    git config user.email "jenkins@ci"
                    git config user.name "Jenkins CI"

                    git checkout develop
                    git pull origin develop

                    git checkout master
                    git pull origin master

                    # Evitar conflicto del Jenkinsfile
                    git checkout origin/master -- Jenkinsfile

                    git merge origin/develop --no-edit || true

                    # Actualizar CHANGELOG SOLO en master
                    LAST_VERSION=$(grep -oP '^## \\[\\K[0-9]+\\.[0-9]+\\.[0-9]+' CHANGELOG.md | head -n 1)
                    NEW_VERSION=$(echo $LAST_VERSION | awk -F. '{$NF+=1; OFS="."; print}')
                    TODAY=$(date +%Y-%m-%d)

                    awk -v ver="$NEW_VERSION" -v date="$TODAY" '
                        NR==1 {print; print ""; print "## [" ver "] - " date; print "### Changed"; print "- Promoción automática a master."; next}
                        {print}
                    ' CHANGELOG.md > CHANGELOG.tmp && mv CHANGELOG.tmp CHANGELOG.md

                    git add CHANGELOG.md
                    git commit -m "Promoción automática settings - versión $NEW_VERSION" || true

                    git remote set-url origin https://AlfredoVG77:${GITHUB_TOKEN}@github.com/AlfredoVG77/todo-list-aws-settings-multibranch.git
                    git push origin master
                '''
            }
        }
    }
}

    }
}
