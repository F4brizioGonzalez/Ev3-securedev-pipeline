// Jenkinsfile - Pipeline de "Sistema Alumnos" (SecureDev · Evaluación Parcial 3 · OCY1102)
pipeline {
    agent any

    options {
        timestamps()
        timeout(time: 40, unit: 'MINUTES')
        disableConcurrentBuilds()
    }

    // Revisa GitHub cada ~3 minutos: cada push (tuyo o un PR de Dependabot fusionado) dispara un build.
    triggers {
        pollSCM('H/3 * * * *')
    }

    environment {
        PIP_DISABLE_PIP_VERSION_CHECK = '1'
    }

    stages {
        stage('Build') {
            steps {
                script {
                    // Versión = 1.0.<nº de build>+<commit corto>: une cada despliegue con su commit (trazabilidad)
                    def sha = sh(returnStdout: true, script: 'git rev-parse --short HEAD').trim()
                    env.APP_VERSION = "1.0.${env.BUILD_NUMBER}+${sha}"
                }
                echo "Versión de este build: ${env.APP_VERSION}"
                sh 'git log -1 --format="Commit %h de %an: %s"'
                // Cada build parte limpio: sin reportes viejos y con un entorno virtual nuevo
                sh 'rm -rf .venv reports && mkdir -p reports'
                sh 'python3 -m venv .venv'
                sh '.venv/bin/pip install --quiet -r requirements.txt -r requirements-dev.txt'
                sh '.venv/bin/python -m compileall -q app.py'
            }
        }

        stage('Test') {
            steps {
                sh '.venv/bin/python -m pytest --junitxml=reports/junit.xml'
            }
            post {
                always {
                    junit allowEmptyResults: true, testResults: 'reports/junit.xml'
                }
            }
        }

        // >>> [SEGURIDAD]: etapas "Security Review (SAST)" y "Dependency Management"
        stage('Security Review (SAST)') {
            steps {
                catchError(buildResult: 'UNSTABLE', stageResult: 'UNSTABLE') {
                    sh '''#!/bin/bash
                    set -u
                    mkdir -p reports
                    .venv/bin/pip install --quiet bandit
                    echo "== Ejecutando Bandit (SAST) =="
                    .venv/bin/bandit -r app.py -f txt -o reports/bandit.txt
                    cat reports/bandit.txt
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
                    .venv/bin/pip install --quiet pip-audit
                    echo "== Ejecutando pip-audit (SCA) =="
                    .venv/bin/pip-audit -r requirements.txt -o reports/pip-audit.txt
                    cat reports/pip-audit.txt
                    '''
                }
            }
        }

        // >>> [ZAP]: etapas "Deploy Staging" y "DAST - OWASP ZAP"
        stage('Deploy Staging') {
            steps {
                sh 'bash scripts/deploy.sh staging 5001'
            }
        }

        stage('DAST - OWASP ZAP') {
            steps {
                catchError(buildResult: 'UNSTABLE', stageResult: 'UNSTABLE') {
                    sh '''#!/bin/bash
                    set -eu
                    mkdir -p reports
                    echo "== Ejecutando escaneo con OWASP ZAP =="
                    curl -fS "http://zap:8090/JSON/spider/action/scan/?url=http://jenkins:5001"
                    echo ""
                    sleep 15
                    curl -fS "http://zap:8090/OTHER/core/other/htmlreport/" -o reports/zap_report.html
                    curl -fS "http://zap:8090/JSON/core/view/alerts/" -o reports/zap_alerts.json
                    ls -lh reports/zap_report.html
                    if grep -q '"riskcode": "[23]"' reports/zap_alerts.json; then
                        echo "[ALERTA ZAP] Se encontraron vulnerabilidades de riesgo Medio o Alto."
                        exit 1
                    fi
                    '''
                }
            }
        }

        stage('Deploy Producción') {
            // >>> [GATE]: condición "when" que protege producción si hay hallazgos de seguridad
            when {
                expression { currentBuild.currentResult == 'SUCCESS' }
            }
            steps {
                sh 'bash scripts/deploy.sh produccion 5000'
                script { env.PRODUCCION = env.APP_VERSION }
            }
        }
    }

    post {
        always {
            // Registro de trazabilidad: build <-> versión <-> commit <-> pruebas <-> hallazgos <-> despliegue
            withEnv(["RESULTADO=${currentBuild.currentResult}"]) {
                sh 'python3 scripts/trazabilidad.py'
            }
            archiveArtifacts artifacts: 'reports/**', allowEmptyArchive: true
        }
    }
}