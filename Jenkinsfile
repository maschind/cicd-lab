// Set your project Prefix using your GUID
def prefix      = "9597"

// Set variable globally to be available in all stages
// Set Maven command to always include Nexus Settings
def mvnCmd      = "mvn -s ./nexus_openshift_settings.xml"
// Set Development and Production Project Names
def devProject  = "${prefix}-tasks-dev"
def prodProject = "${prefix}-tasks-prod"
// Set the tag for the development image: version + build number
def devTag      = "0.0-0"
// Set the tag for the production image: version
def prodTag     = "0.0"
def destApp     = "tasks-green"
def activeApp   = ""

pipeline {
  agent {
    // Using the Jenkins Agent Pod that we defined earlier
    label "maven-appdev"
  }
  stages {
    // Checkout Source Code and calculate Version Numbers and Tags
    stage('Checkout Source') {
      steps {
        // TBD: Get code from protected Git repository
        git url: "https://maschind:password@gogs-gogs-${prefix}-gogs.apps.cluster-c78c.c78c.example.opentlc.com/CICDLabs/openshift-tasks-private.git"


       script {
          def pom = readMavenPom file: 'pom.xml'
          def version = pom.version

          // Set the tag for the development image: version + build number.
          devTag = pom.version + "-" + currentBuild.number            
          // Set the tag for the production image: version           

          prodTag = pom.version                      
          echo "Dev build tag is ${devTag}"           
          echo "Prod build tag is ${prodTag}"

        }
      }
    }

    // Using Maven build the war file
    // Do not run tests in this step
    stage('Build War File') {
      steps {
        echo "Building version ${devTag}"

        //withMaven {
          //sh mvnCmd
          sh mvnCmd + " clean package -DskipTests"
        //}

      }
    }

    // Using Maven run the unit tests
    stage('Unit Tests') {
      steps {
        echo "Running Unit Tests"

        sh mvnCmd + " test"
      }
    }

    //Using Maven call SonarQube for Code Analysis
    stage('Code Analysis') {
      steps {
        echo "Running Code Analysis"

        // TBD 4C730552-AXYjfwRXlWI5vWMcliFy
        // sh 'mvn clean package sonar:sonar -Dsonar.host.url=http://sonarqube.9597-sonarqube.svc.cluster.local:9000 -Dsonar.projectName=${JOB_BASE_NAME}'
        sh mvnCmd + " sonar:sonar -Dsonar.host.url=http://sonarqube-${prefix}-sonarqube.apps.cluster-c78c.c78c.example.opentlc.com -Dsonar.projectName=${JOB_BASE_NAME} -Dsonar.projectVersion=${devTag}"
              
      }
    }

    // Publish the built war file to Nexus
    stage('Publish to Nexus') {
      steps {
        echo "Publish to Nexus"

        sh mvnCmd + ' deploy -Dsonatype-nexus-staging.username'

      }
    }

    // Build the OpenShift Image in OpenShift and tag it.
    stage('Build and Tag OpenShift Image') {
      steps {
        echo "Building OpenShift container image tasks:${devTag}"
        script {
          openshift.withCluster(){
            openshift.withProject("${prefix}-tasks-dev"){
                def bc = openshift.selector("buildconfig/tasks")
                bc.describe()
                bc.startBuild("--from-file=http://nexus.${prefix}-nexus.svc.cluster.local:8081/repository/releases/org/jboss/quickstarts/eap/tasks/${prodTag}/tasks-${prodTag}.war")

                def builds = bc.related('builds')

                timeout(10) {
                    echo "Waiting for builds to complete..."

                    builds.untilEach(1) {
                        return it.object().status.phase == "Complete"
                    }
                }

                echo "Tagging tasks:latest with tasks:${devTag}"
                openshift.tag("tasks:latest", "tasks:${devTag}")
            }
          }
        }

        // TBD: Start binary build in OpenShift using the file we just published.
        // Either use the file from your
        // workspace (filename to pass into the binary build
        // is openshift-tasks.war in the 'target' directory of
        // your current Jenkins workspace).
        // OR use the file you just published into Nexus:
        // "--from-file=http://nexus.${prefix}-nexus.svc.cluster.local:8081/repository/releases/org/jboss/quickstarts/eap/tasks/${prodTag}/tasks-${prodTag}.war"

      }
    }

    // Deploy the built image to the Development Environment.
    stage('Deploy to Dev') {
      steps {
        echo "Deploy container image to Development Project"
        script {
          openshift.withCluster(){
            openshift.withProject("${prefix}-tasks-dev"){
              // 1. Update the image on the dev deployment config
              // def bc = openshift.selector("buildconfig/tasks")
              echo "Modifying tasks deployment to deploy image with tag ${devTag}"
              openshift.set("image", "dc/tasks", "tasks=image-registry.openshift-image-registry.svc:5000/${prefix}-tasks-dev/tasks:${devTag}")

              // 2. Recreate the config maps with the potentially changed properties files
              openshift.selector('configmap', 'tasks-config').delete()
              def configmap = openshift.create('configmap', 'tasks-config', '--from-file=./configuration/application-users.properties', '--from-file=./configuration/application-roles.properties' )

              // 3. Redeploy the dev deployment
              // Deploy the development application.
              openshift.selector("dc", "tasks").rollout().latest();
              
            }
          }
        }

        // TBD: Deploy the image
        // 3. Redeploy the dev deployment
        // 4. Wait until the deployment is running
        //    The following code will accomplish that by
        //    comparing the requested replicas
        //    (rc.spec.replicas) with the running replicas
        //    (rc.status.readyReplicas)

        // sleep 5
        // def dc = openshift.selector("dc", "tasks").object()
        // def dc_version = dc.status.latestVersion
        // def rc = openshift.selector("rc", "tasks-${dc_version}").object()

        // echo "Waiting for ReplicationController tasks-${dc_version} to be ready"
        // while (rc.spec.replicas != rc.status.readyReplicas) {
        //   sleep 5
        //   rc = openshift.selector("rc", "tasks-${dc_version}").object()
        // }

      }
    }

    // Run Integration Tests in the Development Environment.
    stage('Integration Tests') {
      steps {
        echo "Running Integration Tests"
        script {
          def status = "000"

          // Create a new task called "integration_test_1"
          echo "Creating task"
          // The next bit works - but only after the application
          // has been deployed successfully
          // status = sh(returnStdout: true, script: "curl -sw '%{response_code}' -o /dev/null -u 'tasks:redhat1' -H 'Content-Length: 0' -X POST http://tasks.${prefix}-tasks-dev.svc.cluster.local:8080/ws/tasks/integration_test_1").trim()
          // echo "Status: " + status
          // if (status != "201") {
          //     error 'Integration Create Test Failed!'
          // }

          echo "Retrieving tasks"
          // TBD: Implement check to retrieve the task

          echo "Deleting tasks"
          // TBD: Implement check to delete the task

        }
      }
    }

    // Copy Image to Nexus Container Registry
    stage('Copy Image to Nexus Container Registry') {
      steps {
        echo "Copy image to Nexus Container Registry"
        // TBD. Use skopeo to copy


        // TBD: Tag the built image with the production tag.

      }
    }

    // Blue/Green Deployment into Production
    // -------------------------------------
    // Do not activate the new version yet.
    stage('Blue/Green Production Deployment') {
      steps {
        echo "Blue/Green Deployment"

        // TBD: 1. Determine which application is active
        //      2. Update the image for the other application
        //      3. Deploy into the other application
        //      4. Recreate Config maps for other application
        //      5. Wait until application is running
        //         See above for example code

      }
    }

    stage('Switch over to new Version') {
      steps {
        // TBD: Stop for approval


        echo "Executing production switch"
        // TBD: After approval execute the switch

      }
    }
  }

  post {
        always {
            junit '**/target/surefire-reports/*.xml'
        }
    }

}
