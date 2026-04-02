pipeline {
    agent any
    
    environment {
        BUILD_MOVIE = 'false'
        BUILD_CAST = 'false'
        DEPLOY_MOVIE = 'false'
        DEPLOY_CAST  = 'false'
        MOVIE_RELEASE = 'movie-service'
        CAST_RELEASE = 'cast-service'
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
                    echo "=== CHANGED FILES ==="
                    echo changes
                    echo "=== GIT_BRANCH = ${env.GIT_BRANCH} ==="

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
                    
                    echo "=== DEPLOY_MOVIE = ${env.DEPLOY_MOVIE} ==="
                    echo "=== DEPLOY_CAST = ${env.DEPLOY_CAST} ==="                     
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
            environment {
                NAMESPACE = 'dev'
            }
            when {
                allOf{
                    expression { env.GIT_BRANCH == 'origin/dev' }
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
                                    -f ./helm/values-movie.yaml \
                                    -f ${MOVIE_SECRET} \
                                    -n ${NAMESPACE}"                        
                            }
                            if (env.DEPLOY_CAST == 'true'){
                                sh "helm upgrade --install cast-service ./helm/charts \
                                  -f ./helm/values-cast.yaml \
                                  -f ${CAST_SECRET} \
                                  -n ${NAMESPACE}"                       
                            }                    
                        } 
                    }

            }
        }
        stage('Test dev') {
            environment {
                NAMESPACE = 'dev'
            }
            when {
                allOf{
                    expression { env.GIT_BRANCH == 'origin/dev' }
                    anyOf{
                        environment name: 'DEPLOY_MOVIE', value: 'true'
                        environment name: 'DEPLOY_CAST', value: 'true'
                    }
                }
            }
            steps {
                script {
                    if (env.DEPLOY_MOVIE == 'true') {
                        sh "kubectl rollout status deployment/${MOVIE_RELEASE}-fastapiapp -n ${NAMESPACE}"
                        sh "kubectl delete pod ${MOVIE_RELEASE}-fastapiapp-test-connection -n ${NAMESPACE} --ignore-not-found"
                        sh "helm test ${MOVIE_RELEASE} --logs -n ${NAMESPACE}"
                        sh "kubectl delete pod ${MOVIE_RELEASE}-fastapiapp-test-connection -n ${NAMESPACE} --ignore-not-found"
                    }
                    if (env.DEPLOY_CAST == 'true') {
                        sh "kubectl rollout status deployment/${CAST_RELEASE}-fastapiapp -n ${NAMESPACE}"
                        sh "kubectl delete pod ${CAST_RELEASE}-fastapiapp-test-connection -n ${NAMESPACE} --ignore-not-found"
                        sh "helm test ${CAST_RELEASE} --logs -n ${NAMESPACE}"
                        sh "kubectl delete pod ${CAST_RELEASE}-fastapiapp-test-connection -n ${NAMESPACE} --ignore-not-found"
                    }
                }
            }
        }
        stage('Deploy qa'){
            environment {
                NAMESPACE = 'qa'
            }
            when {
                allOf{
                    expression { env.GIT_BRANCH == 'origin/dev' }
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
                                    -f ./helm/values-movie.yaml \
                                    -f ${MOVIE_SECRET} \
                                    -n ${NAMESPACE}"                            
                            }
                            if (env.DEPLOY_CAST == 'true'){
                                sh "helm upgrade --install cast-service ./helm/charts \
                                    -f ./helm/values-cast.yaml \
                                    -f ${CAST_SECRET} \
                                    -n ${NAMESPACE}"                          
                            }                    
                        } 
                    }
            }            
        }
        stage('Test qa') {
            environment {
                NAMESPACE = 'qa'
            }
            when {
                allOf{
                    expression { env.GIT_BRANCH == 'origin/dev' }
                    anyOf{
                        environment name: 'DEPLOY_MOVIE', value: 'true'
                        environment name: 'DEPLOY_CAST', value: 'true'
                    }
                }
            }
            steps {
                script {
                    if (env.DEPLOY_MOVIE == 'true') {
                        sh "kubectl rollout status deployment/${MOVIE_RELEASE}-fastapiapp -n ${NAMESPACE}"
                        sh "kubectl delete pod ${MOVIE_RELEASE}-fastapiapp-test-connection -n ${NAMESPACE} --ignore-not-found"
                        sh "helm test ${MOVIE_RELEASE} --logs -n ${NAMESPACE}"
                        sh "kubectl delete pod ${MOVIE_RELEASE}-fastapiapp-test-connection -n ${NAMESPACE} --ignore-not-found"
                    }
                    if (env.DEPLOY_CAST == 'true') {
                        sh "kubectl rollout status deployment/${CAST_RELEASE}-fastapiapp -n ${NAMESPACE}"
                        sh "kubectl delete pod ${CAST_RELEASE}-fastapiapp-test-connection -n ${NAMESPACE} --ignore-not-found"
                        sh "helm test ${CAST_RELEASE} --logs -n ${NAMESPACE}"
                        sh "kubectl delete pod ${CAST_RELEASE}-fastapiapp-test-connection -n ${NAMESPACE} --ignore-not-found"
                    }
                }
            }
        }
        stage('Deploy staging'){
            environment {
                NAMESPACE = 'staging'
            }
            when {
                allOf {
                    expression { env.GIT_BRANCH == 'origin/dev' }
                    anyOf {
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
                                    -f ./helm/values-movie.yaml \
                                    -f ./helm/values-staging-prod.yaml \
                                    -f ${MOVIE_SECRET} \
                                    -n ${NAMESPACE}"                            
                            }
                            if (env.DEPLOY_CAST == 'true'){
                                sh "helm upgrade --install cast-service ./helm/charts \
                                -f ./helm/values-cast.yaml \
                                    -f ./helm/values-staging-prod.yaml \
                                    -f ${CAST_SECRET} \
                                    -n ${NAMESPACE}"                          
                            } 
                            sh "kubectl apply -f ./kubernetes/ingress.yaml -n ${NAMESPACE}"                   
                        } 
                    }
            }
        }
        stage('Test staging (helm test - intern )') {
            environment {
                NAMESPACE = 'staging'
            }
            when {
                allOf{
                    expression { env.GIT_BRANCH == 'origin/dev' }
                    anyOf{
                        environment name: 'DEPLOY_MOVIE', value: 'true'
                        environment name: 'DEPLOY_CAST', value: 'true'
                    }
                }
            }
            steps {
                script {
                    if (env.DEPLOY_MOVIE == 'true') {
                        sh "kubectl rollout status deployment/${MOVIE_RELEASE}-fastapiapp -n ${NAMESPACE}"
                        sh "kubectl delete pod ${MOVIE_RELEASE}-fastapiapp-test-connection -n ${NAMESPACE} --ignore-not-found"
                        sh "helm test ${MOVIE_RELEASE} --logs -n ${NAMESPACE}"
                        sh "kubectl delete pod ${MOVIE_RELEASE}-fastapiapp-test-connection -n ${NAMESPACE} --ignore-not-found"
                    }
                    if (env.DEPLOY_CAST == 'true') {
                        sh "kubectl rollout status deployment/${CAST_RELEASE}-fastapiapp -n ${NAMESPACE}"
                        sh "kubectl delete pod ${CAST_RELEASE}-fastapiapp-test-connection -n ${NAMESPACE} --ignore-not-found"
                        sh "helm test ${CAST_RELEASE} --logs -n ${NAMESPACE}"
                        sh "kubectl delete pod ${CAST_RELEASE}-fastapiapp-test-connection -n ${NAMESPACE} --ignore-not-found"
                    }

                }
            }
        }
        stage('Test staging Ingress (curl extern)'){
            when {
                allOf{
                    expression { env.GIT_BRANCH == 'origin/dev' }
                    anyOf{
                        environment name: 'DEPLOY_MOVIE', value: 'true'
                        environment name: 'DEPLOY_CAST', value: 'true'
                    }
                }
            }
            steps {
                script {
                    def publicIP = sh(script: 'curl -s http://checkip.amazonaws.com', returnStdout: true).trim()
                    sh "curl -f http://${publicIP}/api/v1/movies"
                    sh "curl -f http://${publicIP}/api/v1/casts"
                }
            }
        }
        stage('Sanity Check'){
            when {
                expression { env.GIT_BRANCH == 'origin/master' }
            }
            steps {
                input "Does the staging environment look ok? Do you want to continue DEPLOYING to PROD?"
            }
        }
        stage('Deploy prod'){
            environment {
                NAMESPACE = 'prod'
            }
            when {
                allOf {
                    expression { env.GIT_BRANCH == 'origin/master' }
                    anyOf {
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
                                    -f ./helm/values-movie.yaml \
                                    -f ./helm/values-staging-prod.yaml \
                                    -f ${MOVIE_SECRET} \
                                    -n ${NAMESPACE}"                            
                            }
                            if (env.DEPLOY_CAST == 'true'){
                                sh "helm upgrade --install cast-service ./helm/charts \
                                -f ./helm/values-cast.yaml \
                                    -f ./helm/values-staging-prod.yaml \
                                    -f ${CAST_SECRET} \
                                    -n ${NAMESPACE}"                          
                            } 
                            sh "kubectl apply -f ./kubernetes/ingress.yaml -n ${NAMESPACE}"                   
                        } 
                    }
            }
        }
        stage('Test prod (helm test - intern )') {
            environment {
                NAMESPACE = 'prod'
            }
            when {
                allOf{
                    expression { env.GIT_BRANCH == 'origin/master' }
                    anyOf{
                        environment name: 'DEPLOY_MOVIE', value: 'true'
                        environment name: 'DEPLOY_CAST', value: 'true'
                    }
                }
            }
            steps {
                script {
                    if (env.DEPLOY_MOVIE == 'true') {
                        sh "kubectl rollout status deployment/${MOVIE_RELEASE}-fastapiapp -n ${NAMESPACE}"
                        sh "kubectl delete pod ${MOVIE_RELEASE}-fastapiapp-test-connection -n ${NAMESPACE} --ignore-not-found"
                        sh "helm test ${MOVIE_RELEASE} --logs -n ${NAMESPACE}"
                        sh "kubectl delete pod ${MOVIE_RELEASE}-fastapiapp-test-connection -n ${NAMESPACE} --ignore-not-found"
                    }
                    if (env.DEPLOY_CAST == 'true') {
                        sh "kubectl rollout status deployment/${CAST_RELEASE}-fastapiapp -n ${NAMESPACE}"
                        sh "kubectl delete pod ${CAST_RELEASE}-fastapiapp-test-connection -n ${NAMESPACE} --ignore-not-found"
                        sh "helm test ${CAST_RELEASE} --logs -n ${NAMESPACE}"
                        sh "kubectl delete pod ${CAST_RELEASE}-fastapiapp-test-connection -n ${NAMESPACE} --ignore-not-found"
                    }

                }
            }
        }
        stage('Test prod Ingress (curl extern)'){
            when {
                allOf{
                    expression { env.GIT_BRANCH == 'origin/master' }
                    anyOf{
                        environment name: 'DEPLOY_MOVIE', value: 'true'
                        environment name: 'DEPLOY_CAST', value: 'true'
                    }
                }
            }
            steps {
                script {
                    def publicIP = sh(script: 'curl -s http://checkip.amazonaws.com', returnStdout: true).trim()
                    sh "curl -f http://${publicIP}/api/v1/movies"
                    sh "curl -f http://${publicIP}/api/v1/casts"
                }
            }
        }        
    }      
}
