def currentDeploymentId = null
def previousDeploymentId = null
def deploymentId = null
def internalDockerRegistry = null

node {
   timeout(30) {

        stage("Checkout SCM") {
            checkout scm
        }

        stage("Record Deployment") {
           openshift.withCluster() {
               openshift.withProject() {
                   def configMap = openshift.selector('configmap/drupal-deployments').object()
                   currentDeploymentId = configMap.data['CURRENT_DEPLOYMENT_ID'].toInteger()
                   previousDeploymentId = configMap.data['PREVIOUS_DEPLOYMENT_ID'].toInteger()
                   deploymentId = configMap.data['NEXT_DEPLOYMENT_ID'].toInteger() + 1
                   echo "The id of this deployment will be '${deploymentId}'."

                   configMap.data['NEXT_DEPLOYMENT_ID'] = "${deploymentId}"
                   openshift.apply(configMap)
               }
           }
        }

        stage('Build Docker Image') {
            echo "Building the Docker image for this deployment. Image will be tagged as 'redhatdeveloper/rhdp-drupal:${deploymentId}'..."
                openshift.withCluster() {
                    openshift.withProject() {

                        def buildConfig = openshift.create(openshift.process(readFile(file:'openshift/docker-image-build.yml'),'-p', "DEPLOYMENT_ID=${deploymentId}"))
                        def build = buildConfig.startBuild()
                        build.untilEach(1) {
                            echo "Waiting for build of Docker Image 'redhatdeveloper/rhdp-drupal:${deploymentId} to complete..."
                            sleep 10
                            return (it.object().status.phase == 'Complete')
                        }

                    }
                }
        }

        stage('Bootstrap Deployment') {
            echo "Bootstrapping the environment for deployment '${deploymentId}'..."
            openshift.withCluster() {
                openshift.withProject() {

                /*
                    I see no way of being able to reference an image from an ImageStream in a DeploymentConfig without
                    having automatic build triggers.
                */
                    def imageStream = openshift.selector("is/rhdp-drupal")
                    internalDockerRegistry = imageStream.object().status['dockerImageRepository']

                    def deployJob = openshift.create(openshift.process(readFile(file:'openshift/drupal-deployment-job.yml'),'-p', "DEPLOYMENT_ID=${deploymentId}", "IMAGE_STREAM=${internalDockerRegistry}"))
                    waitUntil() {
                        echo "Waiting for the completion of job 'drupal-deployment-job-${deploymentId}. This may take some time..."
                        sleep 15
                        openshift.raw("get job drupal-deployment-job-${deploymentId} -o jsonpath='{.status.conditions[?(@.type==\"Complete\")].status}'").out.toString().trim().toBoolean()
                    }
                }
            }
        }

        stage('Deploy Drupal') {
            echo "Deploying Drupal for deployment '${deploymentId}'..."
            openshift.withCluster() {
                openshift.withProject() {
                def deploymentConfig = openshift.create(openshift.process(readFile(file:'openshift/drupal-deployment.yml'), '-p', "DEPLOYMENT_ID=${deploymentId}", , "IMAGE_STREAM=${internalDockerRegistry}"));
                   deploymentConfig.rollout().status()
                }
            }
        }

        stage("Expose Endpoint") {
            echo "Exposing HTTP endpoint for Drupal deployment '${deploymentId}..."
            openshift.withCluster() {
                openshift.withProject() {
                    /*
                        This should be using openshift.verifyService(), but it's not in the plugin version installed here.
                    */
                    openshift.create(openshift.process(readFile(file:'openshift/drupal-http-service.yml'), '-p', "DEPLOYMENT_ID=${deploymentId}"));
                }
            }
        }


        stage("Update 'next' route") {
            openshift.withCluster() {
               openshift.withProject() {
                   def nextServiceName = "drupal-http-${deploymentId}"
                   echo "Updating 'next' route to send traffic to service '${nextServiceName}'.."
                   openshift.raw("patch route/next -p '{\"spec\":{\"to\":{\"name\":\"${nextServiceName}\"}}}'")
               }
            }
        }

        stage("Acceptance Tests") {
            echo "Here we run acceptance tests!"
        }

        stage("Promote To Live") {
            openshift.withCluster() {
                openshift.withProject() {

                    def previousServiceName = "drupal-http-${currentDeploymentId}"
                    def currentServiceName = "drupal-http-${deploymentId}"

                    def configMap = openshift.selector('configmap/drupal-deployments').object()
                    configMap.data['PREVIOUS_DEPLOYMENT_ID'] = "${currentDeploymentId}"
                    configMap.data['CURRENT_DEPLOYMENT_ID'] = "${deploymentId}"
                    openshift.apply(configMap)


                    echo "Updating 'previous' route to send traffic to service '${previousServiceName}'.."
                    openshift.raw("patch route/previous -p '{\"spec\":{\"to\":{\"name\":\"${previousServiceName}\"}}}'")

                    echo "Updating 'current' route to send traffic to service '${currentServiceName}'.."
                    openshift.raw("patch route/current -p '{\"spec\":{\"to\":{\"name\":\"${currentServiceName}\"}}}'")
                }
            }
        }
   }
}