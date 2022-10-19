import groovy.json.JsonSlurper

node {

    stage('Clone repository') {
        checkout scm
    }

    stage('Install dependencies') {
        dir("client") {
            sh 'npm install'
        }
    }

    stage('Build VueJS App') {
        dir("client") {
            sh 'npm run build'
        }
    }

    withEnv(["DAVRA_CRED=Davra"]) {
        stage('Build Docker image') {        
                script {
                    sh "docker build -t ${env.AWS_ACCOUNT_ID}.dkr.ecr.${env.REGION}.amazonaws.com/${env.REGISTRY_NAME}:${env.BUILD_NUMBER} ./"
                }
            }

        // Uploading Docker images into AWS ECR
        withCredentials([[$class: 'UsernamePasswordMultiBinding', credentialsId: 'sample-aws-key', usernameVariable: 'AWS_ACCESS_KEY_ID', passwordVariable: 'AWS_SECRET_ACCESS_KEY']]) {
            stage('Pushing to ECR') {
                script {
                    sh "aws ecr get-login-password --region ${env.REGION} | docker login --username AWS --password-stdin ${env.AWS_ACCOUNT_ID}.dkr.ecr.${env.REGION}.amazonaws.com"
                    sh "docker push ${env.AWS_ACCOUNT_ID}.dkr.ecr.${env.REGION}.amazonaws.com/${env.REGISTRY_NAME}:${env.BUILD_NUMBER}"
                    sh "docker rmi -f ${env.AWS_ACCOUNT_ID}.dkr.ecr.${env.REGION}.amazonaws.com/${env.REGISTRY_NAME}:${env.BUILD_NUMBER}"
                    sh "docker rmi -f ${env.AWS_ACCOUNT_ID}.dkr.ecr.${env.REGION}.amazonaws.com/${env.REGISTRY_NAME}:latest"
                }  
            }

            stage('ECR Container scan') {
                script {
                    sh "aws ecr start-image-scan --region ${env.REGION} --repository-name ${env.REGISTRY_NAME} \
                            --image-id imageTag=${env.BUILD_NUMBER} --output json | tee ecr_start_scan_${env.BUILD_NUMBER}.txt"                   
                    sh "aws ecr wait image-scan-complete --region ${env.REGION} --repository-name ${env.REGISTRY_NAME} \
                            --image-id imageTag=${env.BUILD_NUMBER}"
                    sh "aws ecr describe-image-scan-findings --region ${env.REGION} --repository-name ${env.REGISTRY_NAME} \
                            --image-id imageTag=${env.BUILD_NUMBER}  --output json | tee ecr_scanResult_${env.BUILD_NUMBER}.txt"
                    def data = readFile(file: "ecr_scanResult_${env.BUILD_NUMBER}.txt")
                    def jsonSlurper = new JsonSlurper()
                    def scanResults = jsonSlurper.parseText(data)
                    scanResults["imageScanFindings"]["findings"].each { finding ->
                            assert finding["severity"] != "CRITICAL"
                            assert finding["severity"] != "HIGH"
                        }
                }
            }
        }

        stage('Update Davra Service Config'){          
            script{
                def serviceConfig = """
                    {"config": {
                        "dockerImage": "${env.AWS_ACCOUNT_ID}.dkr.ecr.${env.REGION}.amazonaws.com/${env.REGISTRY_NAME}:${env.BUILD_NUMBER}","dockerRegistrySecret": "${env.SECRET}", "routes": [
                        {
                          "type": "DAVRA_NAMED_VIRTUAL_HOST",
                          "hostname": "${env.HOST}"
                        }
                      ]
                    }}
                """
                httpRequest (consoleLogResponseBody: true,
                    contentType: 'APPLICATION_JSON',
                    authentication: env.DAVRA_CRED,
                    httpMode: 'PATCH',
                    requestBody: serviceConfig,
                    url: "https://${env.TENANTID}.davra.com/api/v1/serviceadmin/${env.SERVICE}",
                    validResponseCodes: '200')
            }
        }

        stage('Wait for Davra Service to run new container'){          
            script{   
                for (int i = 0; i < 10; i++) {
                    sleep(120)
                    def serviceRunning = false
                    def response = httpRequest (consoleLogResponseBody: true,
                                        contentType: 'APPLICATION_JSON',
                                        authentication: env.DAVRA_CRED,
                                        httpMode: 'GET',
                                        url: "https://${env.TENANTID}.davra.com/api/v1/serviceadmin/${env.SERVICE}/status",
                                        validResponseCodes: '200')
                    def jsonSlurper = new JsonSlurper()
                    def serviceDetails = jsonSlurper.parseText(response.getContent())
                    serviceDetails["pods"].each { pod ->
                        pod["status"]["containerStatuses"].each { container -> 
                            echo container.toString()
                            if(container["image"] == "${env.AWS_ACCOUNT_ID}.dkr.ecr.${env.REGION}.amazonaws.com/${env.REGISTRY_NAME}:${env.BUILD_NUMBER}" && container["state"]["running"]){
                                serviceRunning = true                       
                            }
                        }
                    }
                    if(serviceRunning){
                        break
                    }
                    // Fail job on last iteration if service state is not running
                    if(i == 9){
                        error("Davra service failed to run new container alloted time")
                    }
                }
            }
        }
    }

    stage('Run Playwright and Tenable jobs') {
        parallel(
            "Playwright" : {
                stage('Playwright job') {
                    build job: "playwright-job", parameters:[
                        string(name: 'BRANCH',value: 'main'),
                        string(name: 'WORKERS',value: '1')
                    ], wait: true
                }
            }, 
            "Tenable" : {
                stage('Tenable job') {
                    build job: "tenable-job", wait: true
                }
            }
        )  
    }
} 