// Jenkinsfile - Pipeline de "Sistema Alumnos" (OCY1102) - Guías 3.3.2 y 3.3.3
pipeline {
    agent any

    options {
        timestamps()
        timeout(time: 30, unit: 'MINUTES')
        disableConcurrentBuilds()
    }

    environment {
        PIPENV_VENV_IN_PROJECT    = '1'
        PIPENV_IGNORE_VIRTUALENVS = '1'
        PIPENV_NOSPIN             = '1'
    }

    stages {
        stage('Build') {
            steps {
                script {
                    def sha = sh(returnStdout: true, script: 'git rev-parse --short HEAD').trim()
                    env.GIT_SHA     = sha
                    env.APP_VERSION = "1.0.${env.BUILD_NUMBER}+${sha}"
                }
                echo "Versión de este build: ${env.APP_VERSION}"
                sh 'rm -rf .venv reports && mkdir -p reports'
                sh 'pipenv install --dev --deploy'
                sh 'pipenv run python -m compileall -q app.py'
            }
        }

        stage('Test') {
            steps {
                sh 'pipenv run pytest --junitxml=reports/junit.xml'
            }
            post {
                always {
                    junit allowEmptyResults: true, testResults: 'reports/junit.xml'
                }
            }
        }

        stage('Analyze') {
            steps {
                catchError(buildResult: 'UNSTABLE', stageResult: 'UNSTABLE') {
                    sh '''#!/bin/bash
                        set -u
                        mkdir -p reports
                        rc=0
                        echo "== flake8 (estilo y errores de programación) =="
                        flake8 app.py tests > reports/flake8.txt || rc=1
                        cat reports/flake8.txt
                        echo "== bandit (seguridad del código) =="
                        bandit -q -r app.py -f txt -o reports/bandit.txt || rc=1
                        cat reports/bandit.txt
                        exit $rc
                    '''
                }
            }
        }

        stage('Dependency Management') {
            steps {
                catchError(buildResult: 'UNSTABLE', stageResult: 'UNSTABLE') {
                    sh '''#!/bin/bash
                        set -u
                        mkdir -p reports
                        pipenv requirements --hash > reports/requirements.lock.txt
                        pipenv run pip list --outdated > reports/dependencias-desactualizadas.txt || true
                        cat reports/dependencias-desactualizadas.txt
                        pip-audit -r reports/requirements.lock.txt --no-deps --disable-pip --aliases on -f json -o reports/pip-audit.json || true
                        rc=0
                        pip-audit -r reports/requirements.lock.txt --no-deps --disable-pip --aliases on -o reports/pip-audit.txt || rc=1
                        [ -f reports/pip-audit.txt ] || echo "Sin vulnerabilidades conocidas" > reports/pip-audit.txt
                        cat reports/pip-audit.txt
                        exit $rc
                    '''
                }
            }
        }

        // >>> [DEPLOY] aquí irá la etapa Deploy (paso 6)
    }

    post {
        always {
            archiveArtifacts artifacts: 'reports/**', allowEmptyArchive: true
        }
    }
}