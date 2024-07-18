pipeline {
    environment {
        DOCKER_ID = "poissonchat13"
        DOCKER_IMAGE = "spring-petclinic-vets-service"
        JMETER_TAG = "vets"
    }
    agent any

    stages {
        stage('Recuperation de la version Majeur') {
            steps {
                script {
                    VERSION_MAJEUR = sh(script: 'head -n 5 ./README.md | tail -n 1', returnStdout: true).trim()
                    env.DOCKER_TAG = "${VERSION_MAJEUR}.${BUILD_ID}"
                }
            }
        }
        stage('Docker Build Dev') {
            steps {
                sh '''
                docker build -t local-img/$DOCKER_IMAGE:latest .
                kind load docker-image local-img/$DOCKER_IMAGE:latest
                '''
            }
        }
        stage('Deploiement Developpement') {
            environment {
                KUBECONFIG = credentials("config")
            }
            steps {
                sh '''
                kubectl config use-context kind-kind
                cp -r /opt/helm/* ./
                rm -Rf .kube
                mkdir .kube
                ls
                cat $KUBECONFIG > .kube/config
                helm install petclinic-dev spring-pet-clinic-litecloud --values=./spring-pet-clinic-litecloud/value.yaml --set $JMETER_TAG.repo=local-img --set $JMETER_TAG.pull=Never 
                sleep 120 
                '''
            }
        }
        stage('Test Acceptance') {
            steps {
                sh 'curl localhost:30105'
            }
        }
        stage('Test Performance Jmeter') {
            steps {
                sh '''
                date >> /opt/custom/test/jmeter-commit-version.log
                echo "$DOCKER_IMAGE:$DOCKER_TAG" >> /opt/custom/test/jmeter-commit-version.log
                /opt/apache-jmeter/bin/jmeter -n -t /opt/custom/test/petclinic_test_plan.jmx -l /opt/custom/test/petclinic_result_test.jtl
                '''
            }
        }
        stage('Build Jmeter pour report') {
            environment {
                API_TOKEN = credentials("API_TOKEN")
            }
            steps {
                script {
                    def displayName = "${JMETER_TAG}-${DOCKER_TAG}"
                    def description = "Build trigger by the job ${JOB_NAME}  On the microservice ${DOCKER_IMAGE} "

                    def buildResult = build job: 'jmeter-perf-central', 
                                           wait: true, 
                                           propagate: false

                    def jobUrl = buildResult.getAbsoluteUrl()

                    echo "Job URL: ${jobUrl}"
                    
                    sh """
                    curl -X POST -u ${API_TOKEN} -F 'json={"displayName":"${displayName}","description":"${description}"}' \
                    '${jobUrl}configSubmit'
                    """
                }
            }
        }
        stage('Demontage Env Dev') {
            environment {
                KUBECONFIG = credentials("config")
            }
            steps {
                sh '''
                rm -Rf .kube
                mkdir .kube
                ls
                cat $KUBECONFIG > .kube/config
                helm uninstall petclinic-dev
                kubectl config use-context default
                '''
            }
        }
        stage('Push Image pour prod') {
            environment {
                DOCKER_PASS = credentials("DOCKER_HUB_PASS")
            }
            steps {
                sh '''
                docker login -u $DOCKER_ID -p $DOCKER_PASS
                docker tag local-img/$DOCKER_IMAGE:latest $DOCKER_ID/$DOCKER_IMAGE:$DOCKER_TAG
                docker tag $DOCKER_ID/$DOCKER_IMAGE:$DOCKER_TAG $DOCKER_ID/$DOCKER_IMAGE:latest
                docker push $DOCKER_ID/$DOCKER_IMAGE:$DOCKER_TAG
                docker push $DOCKER_ID/$DOCKER_IMAGE:latest
                '''
            }
        }
    }
}
