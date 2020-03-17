@Library("OpenEnclaveCommon") _
oe = new jenkins.common.Openenclave()

GLOBAL_TIMEOUT_MINUTES = 240

OETOOLS_REPO_NAME = "oejenkinscidockerregistry.azurecr.io"
OETOOLS_REPO_CREDENTIAL_ID = "oejenkinscidockerregistry"
OETOOLS_DOCKERHUB_REPO_CREDENTIAL_ID = "oeciteamdockerhub"


// Build Docker images triggering 'OpenEnclave-Docker-Images' job with code from current branch and repository
stage("Build Docker Images") {
    build job: '/CI-CD_Infrastructure/OpenEnclave-Build-Docker-Images',
          parameters: [string(name: 'REPOSITORY_NAME', value: env.REPOSITORY),
                       string(name: 'BRANCH_NAME', value: env.BRANCH),
                       string(name: 'DOCKER_TAG', value: "e2e"),
                       string(name: 'AGENTS_LABEL', value: "images-build-e2e"),
                       booleanParam(name: 'TAG_LATEST',value: false)]
}

stage("Build Jenkins Agents images") {
    build job: '/CI-CD_Infrastructure/OpenEnclave-Build-Azure-Managed-Images',
          parameters: [string(name: 'REPOSITORY_NAME', value: env.REPOSITORY),
                       string(name: 'BRANCH_NAME', value: env.BRANCH),
                       string(name: 'OE_DEPLOY_IMAGE', value: "oetools-deploy:e2e"),
                       string(name: 'REGION', value: env.REGION),
                       string(name: 'RESOURCE_GROUP', value: env.RESOURCE_GROUP),
                       string(name: 'IMAGE_ID', value: "e2e"),
                       string(name: 'DOCKER_TAG', value: "e2e"),
                       string(name: 'AGENTS_LABEL', value: "images-build-e2e")]
}

stage("Run tests on new Agents") {
    build job: '/CI-CD_Infrastructure/OpenEnclave-Custom-Label-Testing',
          parameters: [string(name: 'REPOSITORY_NAME', value: env.REPOSITORY),
                       string(name: 'BRANCH_NAME', value: env.BRANCH),
                       string(name: 'DOCKER_TAG', value: "e2e"),
                       string(name: 'UBUNTU_1604_CUSTOM_LABEL', value: "xenial-e2e"),
                       string(name: 'UBUNTU_1804_CUSTOM_LABEL', value: "bionic-e2e"),
                       string(name: 'UBUNTU_NONSGX_CUSTOM_LABEL', value: "nonSGX-e2e"),
                       string(name: 'RHEL_8_CUSTOM_LABEL', value: "rhel-8-e2e"),
                       string(name: 'WINDOWS_2016_CUSTOM_LABEL', value: "windows-e2e"),
                       string(name: 'WINDOWS_2016_DCAP_CUSTOM_LABEL', value: "windows-dcap-e2e"),
                       string(name: 'WINDOWS_NONSGX_CUSTOM_LABEL', value: "nonSGX-Windows-e2e")]
}

if(params.UPDATE_PRODUCTION_INFRA) {
    def docker_images_names = ["oetools-full-16.04",
                               "oetools-full-18.04",
                               "oetools-minimal-18.04",
                               "oetools-deploy"]

    node("nonSGX") {
        timeout(GLOBAL_TIMEOUT_MINUTES) {
            stage("Backup current production Docker images") {
                docker.withRegistry("https://${OETOOLS_REPO_NAME}", OETOOLS_REPO_CREDENTIAL_ID) {
                    for (image_name in docker_images_names) {
                        def image = docker.image("${OETOOLS_REPO_NAME}/${image_name}:latest")
                        oe.exec_with_retry { image.pull() }
                        oe.exec_with_retry { image.push("latest-backup") }
                    }
                }
            }

            stage("Backup current production Azure managed images") {
                def az_backup_images_script = """
                    az login --service-principal -u \$SERVICE_PRINCIPAL_ID -p \$SERVICE_PRINCIPAL_PASSWORD --tenant \$TENANT_ID
                    az account set -s \$SUBSCRIPTION_ID

                    source ${WORKSPACE}/.jenkins/provision/utils.sh

                    for IMG in ubuntu-16.04-SGX ubuntu-18.04-SGX rhel-8-SGX ws2016-SGX ws2016-SGX-DCAP ws2016-nonSGX; do
                        retrycmd_if_failure 10 300 30m az image delete --resource-group ${BACKUP_RESOURCE_GROUP} --name stable-\$IMG

                        IMG_ID=`az image show --resource-group ${PRODUCTION_RESOURCE_GROUP} --name stable-\$IMG | jq -r '.id'`
                        retrycmd_if_failure 10 300 30m az resource move --ids \$IMG_ID --destination-group ${BACKUP_RESOURCE_GROUP}
                    done
                """
                oe.azureEnvironment(az_backup_images_script)
            }
        }
    }

    node("nonSGX-e2e") {
        timeout(GLOBAL_TIMEOUT_MINUTES) {
            stage("Update production Docker images") {
                docker.withRegistry("https://${OETOOLS_REPO_NAME}", OETOOLS_REPO_CREDENTIAL_ID) {
                    for (image_name in docker_images_names) {
                        def image = docker.image("${OETOOLS_REPO_NAME}/${image_name}:e2e")
                        oe.exec_with_retry { image.pull() }
                        oe.exec_with_retry { image.push("latest") }
                    }
                }
            }

            stage("Update production Azure managed images") {
                def az_update_images_script = """
                    az login --service-principal -u \$SERVICE_PRINCIPAL_ID -p \$SERVICE_PRINCIPAL_PASSWORD --tenant \$TENANT_ID
                    az account set -s \$SUBSCRIPTION_ID

                    source ${WORKSPACE}/.jenkins/provision/utils.sh

                    for IMG in ubuntu-16.04-SGX \
                               ubuntu-18.04-SGX \
                               rhel-8-SGX \
                               ws2016-SGX \
                               ws2016-SGX-DCAP \
                               ws2016-nonSGX; do
                        az image delete --resource-group ${RESOURCE_GROUP} --name stable-\$IMG

                        # Copy e2e image to stable image

                        az image delete --resource-group ${RESOURCE_GROUP} --name e2e-\$IMG
                    done
                """
                oe.azureEnvironment(az_update_images_script)
            }
        }
    }
}
