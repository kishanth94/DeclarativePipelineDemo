pipeline{
    agent any
    environment{
	    ENVIRONMENT = 'TEST'
	    TIME_STAMP = new java.text.SimpleDateFormat('yyyy_MM_dd').format(new Date())
        DOCKER_TAG = "${ENVIRONMENT}_${TIME_STAMP}_${env.BUILD_NUMBER}"
    }
	
    stages{
        stage("Git Checkout"){
            steps{
                git credentialsId: 'github', url: 'https://github.com/kishanth94/DeclarativePipelineDemo'
            }
        }
	
        stage("Build docker image, Push it to DockerHub & Remove the images from Jenkins"){
            steps{
                script{
                         sh '''
                            docker build -t springapp:${DOCKER_TAG} .
							cat /opt/my_password.txt | docker login --username kishanth1994 --password-stdin
							docker tag springapp:${DOCKER_TAG} kishanth1994/springapp:${DOCKER_TAG}
							docker push kishanth1994/springapp:${DOCKER_TAG}
							docker rmi springapp:${DOCKER_TAG}
				            docker rmi kishanth1994/springapp:${DOCKER_TAG}
                            docker logout
			             '''
		        }
		    }
	    }
	
        stage("Copying the playbook and kubernetes manifests to Ansible server"){
		    steps{
			    script{
				    echo "${WORKSPACE}"
					echo "${DOCKER_TAG}"
					        sh '''
								 sudo grep -irl {DOCKER_TAG} ${WORKSPACE}/kubernetes/manifests-yamls/deployment.yaml | xargs sed -i "s/{DOCKER_TAG}/${DOCKER_TAG}/g"
								 sudo scp -o StrictHostKeyChecking=no ${WORKSPACE}/kubernetes/manifests-yamls/*.yaml ansadmin@172.31.21.190:/opt/k8s-ansible/kubernetes/
							'''     
				}
            }
        }

        stage('manual approval'){
            steps{
                script{
                    timeout(10) {
                        mail bcc: '', body: "<br>Project: ${env.JOB_NAME} <br>Build Number: ${env.BUILD_NUMBER} <br> Go to build url and approve the deployment request <br> URL of the build: ${env.BUILD_URL}", cc: '', charset: 'UTF-8', from: '', mimeType: 'text/html', replyTo: '', subject: "${currentBuild.result} CI: Project name -> ${env.JOB_NAME}", to: "devopsawsfreetier@gmail.com";  
                        input(id: "Deploy Gate", message: "Deploy ${params.project_name}?", ok: 'Deploy')
                    }
                }
            }
        }

        stage('Deploying application on k8s cluster from ansible-server using playbook') {
            steps {
               script{
			        sh 'ssh -o StrictHostKeyChecking=no ansadmin@172.31.21.190 "kubectl config get-contexts; kubectl config use-context kubernetes-admin@kubernetes; whoami && hostname; ansible-playbook -i /opt/k8s-ansible/hosts /opt/k8s-ansible/kubernetes/playbook-deployment-service.yaml; sudo rm -rf /opt/k8s-ansible/kubernetes/*.yaml;"'   
               }
            }
        }	

        stage('verifying app deployment'){
            steps{
                script{
                     withCredentials([kubeconfigFile(credentialsId: 'kubernetes-configs', variable: 'KUBECONFIG')]) {
                         sh 'kubectl run curl --image=curlimages/curl -i --rm --restart=Never -- curl sample-tomcat-service:8080'

                     }
                }
            }
        }		
    }
    
    post {
	    always {
		echo 'Deleting the Workspace'
		deleteDir() /* Clean Up our Workspace */
	    }
	    success {
		mail to: 'devopsawsfreetier@gmail.com',
		     subject: "Success Build Pipeline: ${currentBuild.fullDisplayName}",
		     body: "The pipeline ${env.BUILD_URL} completed successfully"
	    }
	    failure {
		mail to: 'devopsawsfreetier@gmail.com',
		     subject: "Failed Build Pipeline: ${currentBuild.fullDisplayName}",
		     body: "Something is wrong with ${env.BUILD_URL}"
	    }
    }	
}
