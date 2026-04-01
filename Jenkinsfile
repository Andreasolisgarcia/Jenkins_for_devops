pipeline {
    agent any
    
    environment {
        BUILD_MOVIE = 'false'
        BUILD_CAST = 'false'
        DEPLOY_MOVIE = 'false'
        DEPLOY_CAST  = 'false'
    }

    
    stages {
        stage('Checkout') { 
            steps {
                checkout scm
                script {
                    def changes = sh(
                        script: 'git diff --name-only HEAD~1 HEAD',
                        returnStdout: true
                    )
                    if (changes.contains('movie-service/')) {
                        env.BUILD_MOVIE = 'true'
                        env.DEPLOY_MOVIE = 'true'
                        env.IMAGE_MOVIE_NAME= 'reasg/jenkins_for_devops_movie_service'
                        env.IMAGE_MOVIE_TAG= "${env.IMAGE_MOVIE_NAME}:${env.GIT_COMMIT}"
                    }
                    if (changes.contains('cast-service/')) {
                        env.BUILD_CAST = 'true'
                        env.DEPLOY_CAST = 'true' 
                        env.IMAGE_CAST_NAME= 'reasg/jenkins_for_devops_cast_service'
                        env.IMAGE_CAST_TAG = "${env.IMAGE_CAST_NAME}:${env.GIT_COMMIT}"
                    }  
                    if (changes.contains('helm/')) {
                        env.DEPLOY_MOVIE = 'true'
                        env.DEPLOY_CAST = 'true'
                    }                           
                }
            }
        }
        stage('Build & push'){
            when {
                anyOf{
                    environment name: 'BUILD_MOVIE', value: 'true' 
                    environment name: 'BUILD_CAST', value: 'true' 
                }
            }
             steps {
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub',
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS'
                )]){
                    sh 'echo ${DOCKER_PASS} | docker login -u ${DOCKER_USER} --password-stdin'
                    echo 'Loging succesfully'
                    script{
                        if ( env.BUILD_MOVIE == 'true' ) {
                            sh "docker build -t ${env.IMAGE_MOVIE_TAG} ./movie-service"
                            sh "docker push ${env.IMAGE_MOVIE_TAG}"
                        }
                        if ( env.BUILD_CAST == 'true' ) {
                            sh "docker build -t ${env.IMAGE_CAST_TAG} ./cast-service"
                            sh "docker push ${env.IMAGE_CAST_TAG}"
                        }  
                    }
                  
                }         
            }
        }
        stage('Deploy dev'){
            when {
                allOf{
                    branch 'dev'
                    anyOf{
                        environment name: 'DEPLOY_MOVIE', value: 'true' 
                        environment name: 'DEPLOY_CAST', value: 'true'
                    }
                }
            }
            steps{
                withCredentials([
                    file(credentialsId: 'values-movie-secret', variable: 'MOVIE_SECRET'),
                    file(credentialsId: 'values-cast-secret', variable: 'CAST_SECRET')
                    ]){
                        script{
                            if (env.DEPLOY_MOVIE == 'true'){
                                sh "helm upgrade --install movie-service ./helm/charts \
                                    -f ./helm/values.yaml \
                                    -f ./helm/values-movie.yaml \
                                    -f ${MOVIE_SECRET} \
                                    -n dev"                        
                            }
                            if (env.DEPLOY_CAST == 'true'){
                                sh "helm upgrade --install cast-service ./helm/charts \
                                  -f ./helm/values.yaml \
                                  -f ./helm/values-cast.yaml \
                                  -f ${CAST_SECRET} \
                                  -n dev"                       
                            }                    
                        } 
                    }

            }
        }
        stage('Test dev') {
            steps {
                script {
                    if (env.DEPLOY_MOVIE == 'true') {
                        sh 'helm test movie-service -n dev'
                    }
                    if (env.DEPLOY_CAST == 'true') {
                       sh 'helm test cast-service -n dev'
                    }
                }
        }
        }
        stage('Deploy qa'){
            when {
                allOf{
                    branch 'dev'
                    anyOf{
                        environment name: 'DEPLOY_MOVIE', value: 'true' 
                        environment name: 'DEPLOY_CAST', value: 'true'
                    }
                }
            }
            steps{
                withCredentials([
                    file(credentialsId: 'values-movie-secret', variable: 'MOVIE_SECRET'),
                    file(credentialsId: 'values-cast-secret', variable: 'CAST_SECRET')
                    ]){
                        script{
                            if (env.DEPLOY_MOVIE == 'true'){
                                sh "helm upgrade --install movie-service ./helm/charts \
                                    -f ./helm/values.yaml \
                                    -f ./helm/values-movie.yaml \
                                    -f ${MOVIE_SECRET} \
                                    -n qa"                            
                            }
                            if (env.DEPLOY_CAST == 'true'){
                                sh "helm upgrade --install cast-service ./helm/charts \
                                  -f ./helm/values.yaml \
                                  -f ./helm/values-cast.yaml \
                                  -f ${CAST_SECRET} \
                                  -n qa"                          
                            }                    
                        } 
                    }

            }            
        }
    }
}
