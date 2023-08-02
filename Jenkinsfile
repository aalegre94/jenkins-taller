pipeline {

    agent any

    tools {
        maven '3.9.3'
    }

    environment {
        SONAR_SCANNER = tool 'SonarScaner'
        SERVICE_ACCOUNT_KEY = credentials("service-account-jenkins")
    }

    stages {

        stage ('Compilar'){
            steps {
                echo 'Compilando...'
                sh 'mvn clean compile'
            }
        }

        stage ('Pruebas'){
            steps {
                echo 'Ejecutando pruebas...'
                sh 'mvn test'
            }
            post {
                success {
                    junit 'target/surefire-reports/*.xml'
                }
            }
        }

        stage ('Análisis de código estático') {
            steps {
                echo 'Analizando código con sonarqube...'

                withSonarQubeEnv('sonarqube-server') {
                    sh "${SONAR_SCANNER}/bin/sonar-scanner -Dproject.settings=devops/sonar.properties"
                }

                timeout(time: 2, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Build') {
            steps {
                echo 'Generando build...'
                sh "mvn clean package -DskipTests"
                sh "docker build -t us-central1-docker.pkg.dev/curso-devops-aalegre/develop/servicio-seguridad:$BUILD_NUMBER ."
            }
        }

        stage('Publicar imagen') {
            steps {
                echo 'Publicando imagen en Google Artifactory Registry'
                sh "docker login -u _json_key --password-stdin https://us-central1-docker.pkg.dev < $SERVICE_ACCOUNT_KEY"
                sh "docker push us-central1-docker.pkg.dev/curso-devops-aalegre/develop/servicio-seguridad:$BUILD_NUMBER"

            }
            post {
                always {
                    echo 'Eliminar la imagen generada localmente'
                    sh "docker rmi us-central1-docker.pkg.dev/curso-devops-aalegre/develop/servicio-seguridad:$BUILD_NUMBER"
                }
            }
        }

        stage('Desplegar') {
            steps {
                echo 'Desplegando en cloud run'
                sh "gcloud auth activate-service-account service-jenkins@curso-devops-aalegre.iam.gserviceaccount.com --key-file=$SERVICE_ACCOUNT_KEY"
                sh "gcloud run deploy servicio-seguridad --image=us-central1-docker.pkg.dev/curso-devops-aalegre/develop/servicio-seguridad:$BUILD_NUMBER --port=8080 --region=us-central1 --project=curso-devops-aalegre --allow-unauthenticated"
            }
        }
    }
}