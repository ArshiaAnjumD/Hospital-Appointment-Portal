
pipeline {
    agent any

    parameters {
        string(name: 'VERSION', defaultValue: 'v1.0',
               description: 'Release version, e.g. v1.0 or v2.0')
        booleanParam(name: 'SIMULATE_FAILURE', defaultValue: false,
                     description: 'Demonstrate automatic rollback')
    }

    environment {
        IMAGE = 'arshiaanjumd/hospital-appointment-portal'
        DOCKER_EXE = 'C:\\Users\\arshi\\AppData\\Local\\Programs\\DockerDesktop\\resources\\bin\\docker.exe'
        KUBECTL_EXE = 'C:\\Users\\arshi\\AppData\\Local\\Programs\\DockerDesktop\\resources\\bin\\kubectl.exe'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Validate Version') {
            steps {
                script {
                    env.RELEASE_VERSION = params.VERSION?.trim() ?: 'v1.0'
                    if (!(env.RELEASE_VERSION ==~ /[A-Za-z0-9][A-Za-z0-9_.-]*/)) {
                        error('Use a valid version such as v1.0 or v2.0')
                    }
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                bat '''
                    "%DOCKER_EXE%" build -t %IMAGE%:%RELEASE_VERSION% .
                    if errorlevel 1 exit /b 1
                '''
            }
        }

        stage('Test Application') {
            steps {
                bat '''
                    "%DOCKER_EXE%" run --rm %IMAGE%:%RELEASE_VERSION% nginx -t
                    if errorlevel 1 exit /b 1
                '''
            }
        }

        stage('Push Image') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub-creds',
                    usernameVariable: 'DH_USER',
                    passwordVariable: 'DH_TOKEN'
                )]) {
                    bat '''
                        @echo off
                        echo %DH_TOKEN%| "%DOCKER_EXE%" login -u %DH_USER% --password-stdin
                        if errorlevel 1 exit /b 1
                        "%DOCKER_EXE%" push %IMAGE%:%RELEASE_VERSION%
                        set RESULT=%ERRORLEVEL%
                        "%DOCKER_EXE%" logout
                        exit /b %RESULT%
                    '''
                }
            }
        }

        stage('Deploy and Verify') {
            steps {
                withCredentials([file(
                    credentialsId: 'kubeconfig',
                    variable: 'KUBECONFIG'
                )]) {
                    script {
                        def previousImage = bat(
                            script: '@echo off & "%KUBECTL_EXE%" get deployment hospital-portal -o=jsonpath="{.spec.template.spec.containers[0].image}"',
                            returnStdout: true
                        ).trim()

                        if (!previousImage) {
                            error('Could not read the current stable image')
                        }

                        try {
                            if (params.SIMULATE_FAILURE) {
                                bat '''
                                    "%KUBECTL_EXE%" set image deployment/hospital-portal hospital-portal=%IMAGE%:%RELEASE_VERSION%-broken
                                    if errorlevel 1 exit /b 1
                                '''
                            } else {
                                bat '''
                                    "%KUBECTL_EXE%" set image deployment/hospital-portal hospital-portal=%IMAGE%:%RELEASE_VERSION%
                                    if errorlevel 1 exit /b 1
                                '''
                            }

                            bat '''
                                "%KUBECTL_EXE%" rollout status deployment/hospital-portal --timeout=120s
                                if errorlevel 1 exit /b 1
                                "%KUBECTL_EXE%" get deployment hospital-portal
                                "%KUBECTL_EXE%" get pods -l app=hospital-portal
                                "%KUBECTL_EXE%" get service hospital-portal-service
                            '''
                        } catch (err) {
                            echo "Deployment failed. Restoring ${previousImage}"
                            bat '''
                                "%KUBECTL_EXE%" rollout undo deployment/hospital-portal
                                if errorlevel 1 exit /b 1
                                "%KUBECTL_EXE%" rollout status deployment/hospital-portal --timeout=120s
                                if errorlevel 1 exit /b 1
                                "%KUBECTL_EXE%" get pods -l app=hospital-portal
                            '''
                            error('Release failed; rollback attempted.')
                        }
                    }
                }
            }
        }
    }

    post {
        always {
            echo 'Hospital Appointment Portal pipeline completed.'
        }
    }
}
