pipeline {
    agent any
    environment {
        PATH = "/usr/local/bin:$PATH"
    }
    stages {
        stage('Verify gcloud') {
            steps {
                sh 'gcloud --version'
            }
        }

        stage("Activate Service Account") {
            steps {
                withCredentials([file(credentialsId: "GCP_service_account", variable: 'SA')]) {
                    sh "gcloud auth activate-service-account --key-file=${SA}"
                }
            }
        }

        stage("Install Dependencies") {
            steps {
                sh """
                    if ! command -v gcloud &> /dev/null; then
                        echo "❌ gcloud CLI not found. Please install gcloud CLI on Jenkins node"
                        exit 1
                    fi
                """
            }
        }

        stage("Read Variables") {
            steps {
                script {
                    def vars = readYaml(file: 'variables.yaml')
                    env.PROJECT_ID = vars.project_id
                    env.HOSTS = vars.hosts
                }
            }
        }

        stage("Stop VM Instances") {
            steps {
                script {
                    for (host in env.HOSTS) {
                        sh """
                            gcloud compute instances stop ${host.name} \
                                --project ${env.PROJECT_ID} \
                                --zone ${host.zone}
                        """
                    }
                }
            }
        }

        stage("Change VM Instance Type") {
            steps {
                script {
                    for (host in env.HOSTS) {
                        sh """
                            gcloud compute instances set-machine-type ${host.name} \
                                --machine-type=${host.new_machine_type} \
                                --project ${env.PROJECT_ID} \
                                --zone ${host.zone}
                        """
                    }
                }
            }
        }

        stage("Start VM Instances & Validate") {
            steps {
                script {
                    for (host in env.HOSTS) {
                        sh """
                            gcloud compute instances start ${host.name} \
                                --project ${env.PROJECT_ID} \
                                --zone ${host.zone}
                        """

                        def appliedType = sh(
                            script: "gcloud compute instances describe ${host.name} --project ${env.PROJECT_ID} --zone ${host.zone} --format='get(machineType)'",
                            returnStdout: true
                        ).trim().split('/').last() 

                        if (appliedType == host.new_machine_type) {
                            echo "✅ VM ${host.name} successfully resized to ${appliedType}"
                        } else {
                            error("❌ Failed to resize ${host.name}, got ${appliedType}")
                        }
                    }
                }
            }
        }
    }
}
