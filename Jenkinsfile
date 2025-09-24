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
                    env.INSTANCE_NAME = vars.Instance_name
                    env.ZONE = vars.zone
                    env.NEW_MACHINE_TYPE = vars.new_machine_type
                }
            }
        }

        stage("Stop VM Instance") {
            steps {
                script {
                    sh """
                        echo "Stopping VM: ${env.INSTANCE_NAME}"
                        gcloud compute instances stop ${env.INSTANCE_NAME} \
                            --project ${env.PROJECT_ID} \
                            --zone ${env.ZONE} \
                            --quiet
                    """
                }
            }
        }

        stage("Change VM Instance Type") {
            steps {
                script {
                    sh """
                        echo "Changing machine type of ${env.INSTANCE_NAME} to ${env.NEW_MACHINE_TYPE}"
                        gcloud compute instances set-machine-type ${env.INSTANCE_NAME} \
                            --machine-type=${env.NEW_MACHINE_TYPE} \
                            --project ${env.PROJECT_ID} \
                            --zone ${env.ZONE} \
                            --quiet
                    """
                }
            }
        }

        stage("Start VM Instance & Validate") {
            steps {
                script {
                    sh """
                        echo "Starting VM: ${env.INSTANCE_NAME}"
                        gcloud compute instances start ${env.INSTANCE_NAME} \
                            --project ${env.PROJECT_ID} \
                            --zone ${env.ZONE} \
                            --quiet
                    """

                    def appliedType = sh(
                        script: "gcloud compute instances describe ${env.INSTANCE_NAME} --project ${env.PROJECT_ID} --zone ${env.ZONE} --format='get(machineType)'",
                        returnStdout: true
                    ).trim().split('/').last()

                    if (appliedType == env.NEW_MACHINE_TYPE) {
                        echo "✅ VM ${env.INSTANCE_NAME} successfully resized to ${appliedType}"
                    } else {
                        error("❌ Failed to resize ${env.INSTANCE_NAME}, got ${appliedType}")
                    }
                }
            }
        }
    }
}
