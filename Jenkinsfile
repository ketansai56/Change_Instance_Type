@Library('gcp-jenkins-library@1.97.0') _
def containerAgent = libraryResource 'agent.yaml'

pipeline {
    agent {
        kubernetes {
            label 'vm_type_change'
            defaultContainer 'jnlp'
            yaml "$containerAgent"
        }
    }

    stages {
        stage("Activate Service Account") {
            steps {
                script {
                    container('gcloud') {
                        withCredentials([file(credentialsId: "GCP_service_account", variable: 'SA')]) {
                            sh("gcloud auth activate-service-account --key-file=${SA}")
                        }
                    }
                }
            }
        }
        stage("Install Dependencies") {
            steps {
                container("gcloud") {
                    sh("apt-get update -qq")
                    sh("apt-get install -y git dnsutils")
                }
            }
        }
        stage("Read Variables") {
            steps {
                script {
                    def vars = readYaml(file: 'variables.yaml')
                    env.PROJECT_ID = vars.project_id
                    env.NEW_MACHINE_TYPE = vars.new_machine_type
                    env.HOSTNAMES = vars.hosts.join(" ")
                    env.ZONE = vars.zone
                }
            }
        }
        stage("Stop VM Instances") {
            steps {
                script {
                    container("gcloud") {
                        for (host in env.HOSTNAMES.split(" ")) {
                            echo "Stopping VM: ${host}"
                            sh("gcloud compute instances stop ${host} --project ${env.PROJECT_ID} --zone ${env.ZONE}")
                        }
                    }
                }
            }
        }
        stage("Change VM Instance Type") {
            steps {
                script {
                    container("gcloud") {
                        for (host in env.HOSTNAMES.split(" ")) {
                            echo "Changing machine type for VM: ${host}"
                            sh("gcloud compute instances set-machine-type ${host} --machine-type ${env.NEW_MACHINE_TYPE} --project ${env.PROJECT_ID} --zone ${env.ZONE}")
                        }
                    }
                }
            }
        }
        stage("Start VM Instances & Validate") {
            steps {
                script {
                    container("gcloud") {
                        for (host in env.HOSTNAMES.split(" ")) {
                            echo "Starting VM: ${host}"
                            sh("gcloud compute instances start ${host} --project ${env.PROJECT_ID} --zone ${env.ZONE}")
                        }
                    }
                }
            }
        }
        stage("Validate VM Instance Type") {
            steps {
                script {
                    container("gcloud") {
                        for (host in env.HOSTNAMES.split(" ")) {
                            echo "Validating VM: ${host}"
                            def appliedType = sh(
                                script: "gcloud compute instances describe ${host} --project ${env.PROJECT_ID} --zone ${env.ZONE} --format='value(machineType.scope(machineTypes))'",
                                returnStdout: true
                            ).trim()
                            if (appliedType == env.NEW_MACHINE_TYPE) {
                                echo "VM ${host} successfully resized to ${appliedType}"
                            } else {
                                error("Failed to resize ${host}, got ${appliedType}")
                            }
                        }
                    }
                }
            }
        }
    }
}
