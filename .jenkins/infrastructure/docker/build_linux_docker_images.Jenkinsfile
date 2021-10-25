// Copyright (c) Open Enclave SDK contributors.
// Licensed under the MIT License.

OECI_LIB_VERSION = env.OECI_LIB_VERSION ?: "master"
oe = library("OpenEnclaveCommon@${OECI_LIB_VERSION}").jenkins.common.Openenclave.new()

pipeline {
    agent {
        label globalvars.AGENTS_LABELS["acc-ubuntu-20.04"]
    }
    options {
        timeout(time: 240, unit: 'MINUTES')
    }
    parameters {
        string(name: "SGX_VERSION", description: "SGX Version to install (Example: 2.15.100). For a list of versions see: https://download.01.org/intel-sgx/sgx_repo/ubuntu/apt_preference_files/")
        string(name: "REPOSITORY_NAME", defaultValue: "openenclave/openenclave", description: "GitHub repository to checkout")
        string(name: "BRANCH_NAME", defaultValue: "master", description: "The branch used to checkout the repository")
        string(name: "DOCKER_TAG", defaultValue: "standalone-linux-build", description: "The tag for the new Docker images")
        string(name: "INTERNAL_REPO", defaultValue: "https://oejenkinscidockerregistry.azurecr.io", description: "Url for internal Docker repository")
        string(name: "OECI_LIB_VERSION", defaultValue: 'master', description: 'Version of OE Libraries to use')
        booleanParam(name: "PUBLISH_DOCKER_HUB", defaultValue: false, description: "Publish container to OECITeam Docker Hub?")
        booleanParam(name: "TAG_LATEST", defaultValue: false, description: "Update the latest tag to the currently built DOCKER_TAG")
    }
    environment {
        INTERNAL_REPO_CREDS = 'oejenkinscidockerregistry'
        DOCKERHUB_REPO_CREDS = 'oeciteamdockerhub'
        BASE_DOCKERFILE_DIR = ".jenkins/infrastructure/docker/dockerfiles/linux/base/"
        LINUX_DOCKERFILE = ".jenkins/infrastructure/docker/dockerfiles/linux/Dockerfile"
    }
    stages {
        stage("Checkout") {
            steps {
                cleanWs()
                checkout scm
            }
        }
        stage("Base Image") {
            stages {
                stage('Build Base') {
                    steps {
                        dir(BASE_DOCKERFILE_DIR) {
                            sh """
                                chmod +x ./build.sh
                                mkdir build
                                cd build
                                ../build.sh -v "${params.SGX_VERSION}" -u "18.04" -t "${params.DOCKER_TAG}"
                                ../build.sh -v "${params.SGX_VERSION}" -u "20.04" -t "${params.DOCKER_TAG}"
                            """
                        }
                    }
                }
                stage('Test Base') {
                    parallel {
                        stage('Test Base - 18.04') {
                            steps {
                                script {
                                    def image = docker.image("openenclave-bionic:${params.DOCKER_TAG}"
                                    image.inside("--cap-add=SYS_PTRACE --device /dev/sgx:/dev/sgx --volume /var/run/aesmd/aesm.socket:/var/run/aesmd/aesm.socket") {
                                        sh """
                                            sudo apt update
                                            sudo apt install -y build-essentials open-enclave libssl-dev
                                        """
                                        helpers.TestSamplesCommand(false, "open-enclave")
                                    }
                                }
                            }
                        }
                        stage('Test Base - 20.04') {
                            steps {
                                script {
                                    def image = docker.image("openenclave-bionic:${params.DOCKER_TAG}"
                                    image.inside("--cap-add=SYS_PTRACE --device /dev/sgx:/dev/sgx --volume /var/run/aesmd/aesm.socket:/var/run/aesmd/aesm.socket") {
                                        sh """
                                            sudo apt update
                                            sudo apt install -y build-essentials open-enclave libssl-dev
                                        """
                                        helpers.TestSamplesCommand(false, "open-enclave")
                                    }
                                }
                            }
                        }
                    }
                }
            }
        }
        stage("Full CI/CD Image") {
            parallel {
                stage("Build Ubuntu 18.04 Docker Image") {
                    oe1804 = oe.dockerImage("oetools-18.04:${DOCKER_TAG}", LINUX_DOCKERFILE, "${buildArgs} --build-arg ubuntu_version=18.04 --build-arg devkits_uri=${DEVKITS_URI}")
                    puboe1804 = oe.dockerImage("oeciteam/oetools-18.04:${DOCKER_TAG}", LINUX_DOCKERFILE, "${buildArgs} --build-arg ubuntu_version=18.04 --build-arg devkits_uri=${DEVKITS_URI}")
                }
            }, "Build Ubuntu 20.04 Docker Image": {
                stage("Build Ubuntu 20.04 Docker Image") {
                    oe2004 = oe.dockerImage("oetools-20.04:${DOCKER_TAG}",LINUX_DOCKERFILE, "${buildArgs} --build-arg ubuntu_version=20.04 --build-arg devkits_uri=${DEVKITS_URI}")
                    puboe2004 = oe.dockerImage("oeciteam/oetools-20.04:${DOCKER_TAG}", LINUX_DOCKERFILE, "${buildArgs} --build-arg ubuntu_version=20.04 --build-arg devkits_uri=${DEVKITS_URI}")
                }
            }
            stage("Push to OE Docker Registry") {
            steps {
                script {
                    docker.withRegistry(params.INTERNAL_REPO, env.INTERNAL_REPO_CREDS) {
                        common.exec_with_retry { oe1804.push() }
                        common.exec_with_retry { oe2004.push() }
                        if(params.TAG_LATEST == "true") {
                            common.exec_with_retry { oe1804.push('latest') }
                            common.exec_with_retry { oe2004.push('latest') }
                        }
                    }
                }
            }
            stage("Push to OE Docker Hub Registry") {
                docker.withRegistry('', DOCKERHUB_REPO_CREDS) {
                    if(PUBLISH_DOCKER_HUB == "true") {
                        oe.exec_with_retry { puboe1804.push() }
                        oe.exec_with_retry { puboe2004.push() }
                        if(TAG_LATEST == "true") {
                            oe.exec_with_retry { puboe1804.push('latest') }
                            oe.exec_with_retry { puboe2004.push('latest') }
                        }
                    }
                }
            }
        }
        }
    }
}

buildLinuxDockerContainers()
