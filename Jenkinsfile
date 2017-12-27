// lavnish : todo_1st : convert to scriplet pipeline
pipeline {

	// We are binding to a single node at this point but we will expand this to use an agent label in the future - TODO
    agent {
        node {
            label ''
        }
    }
    // Specify Pipeline Specific Options
    options {
        // we will like to make sure that we only keep 20 builds at a time.
        buildDiscarder(logRotator(numToKeepStr:'20'))
        // make sure these builds dont hang on for ever .... so timebound it to 60 minutes
        timeout(time: 60, unit: 'MINUTES')
    }

    environment {
	    // the idea is to encapsulate all variables here.
      // Exactly do we want to display all the values here, as this is used to use the varriable in the script. Please clarify
            def server_ip_address_or_name = ""
            def server_username_name = ""
            def server_password = "" // there is a way to get this from jenkins please google.
		          def git_url = ""
	    // todo_2nd : ip address or server name of the server where we will deploy tomcat
		// i expect these def to be available in stages below , can you please see how to do that. or in worst case we will define as jenkins parameter. like ${Branch_Name} below
		// lavnish : todo_2nd : define project_folder="SB_Reference_App" and then use it instead of  dir('SB_Reference_App') { in all of below stages
    }


	stages {
		stage ('checkout') {
			steps {
				// start with clean workspace , sometimes pipeline are stopped manually
				//deleteDir() // lavnish : must delete workspace at end of job , else space on jenkins server goes full. Post build cleanup code placed
				echo "checking out branch ${Branch_Name}"
				// NOTE : ${Branch_Name} is defined as pipeline parameter
				git branch: '${Branch_Name}', url: 'https://gitlab.putnaminv.com/public_cloud/SB_Reference_App.git' // todo_2nd : use git_url from environment section
			}
		}

		// Just Build no Unit Test
        stage('Build') {
			// todo_1st_2 : earlier this stage was 'Build/Unit/Integration Test' now separate it ... make 3 phases Build & Unit Test & integration test :
			// in build call mvn "clean install -DskipTests" not mvn clean package , see stage ('runTests') below to see how to run unit tests .... need to find how to run Integration ( i.e. selenium only later : as it needs to be run after deployment )
			steps {
               dir('SB_Reference_App') {
                    sh mvn "clean install -DskipTests"
                    // Construct the imageVersion of the build from the pom.version and BUILD_NUMBER, this will be used later for tagging the docker image
                }
            }
        }

		stage ('Unit Test') {
			/* Call the Maven build with tests. */
			steps {
				mvn "install -Dmaven.test.failure.ignore=true"

			}
			 /*lavnish : not sure WHY this section is put in post , xunit should run inside post , let me know if i am missing something , else delete in v4
      // Placed the Xunitbuild in post always section, currently we are having XunitBuilder, after testing this Pipeline we can update by giving
        XunitPublisher
      // Below code has been tested as per the Sample java project
      */
      post {
          always {
                  //junit 'target/surefire-reports/*.xml'
                  script {
                      try {
                        step([$class: 'XUnitBuilder', thresholds: [[$class: 'FailedThreshold',
                        unstableThreshold: '1']],tools: [[$class: 'JUnitType', pattern: 'target/surefire-reports/**']]])
                      } catch (e) {
                          currentBuild.result = 'FAILED'
                        throw e
                      } finally {
                        emailext attachLog: true, body: 'Unit Test has FAILED ', subject: 'FAILED', to: 'abc@xyz.com'
                      }
                  }
                  publishHTML([
                    allowMissing: true,
                    alwaysLinkToLastBuild: true,
                    keepAll: true, reportDir: 'reports/',
                    reportFiles: '*.xml',
                    reportName: 'Coverage Report',
                    reportTitles: '']
                              )

                  }
            }
			// lavnish : can we run this in parallel later ?
          /* If we want to run Parallel Unit test and Integration test below can be used-
                                      stage ('test') {
                                        steps {
                                            parallel (
                                                "unit tests": { sh 'mvn test' },
                                                "integration tests": { sh 'mvn integration-test' }
                                            )
                                        }

          */
			// todo_2nd : run tests in parallel https://jenkins.io/blog/2016/06/16/parallel-test-executor-plugin/
		}

		// Run the code coverage report
        stage('JaCoCo Code Coverage') {
            steps {
                script {
                  try {
                    dir('SB_Reference_App') {
                        //sh "${mvnHome}/bin/mvn -B -f ./product-api -Dspring.profiles.active=integration -Dmaven.test.failure.ignore=true prepare-package"
                        jacoco(classPattern: './product-api/target/**/classes', execPattern: './product-api/target/jacoco.exec', sourcePattern: './product-api/src/main/java')
                        // TODO_1st : if code coverage  is below threshold stop build & fail , is there a way show these numbers on UI  or record and send at end of email
                                            }
                  } catch (e) {
                        currentBuild.result = 'FAILED'
                    throw e
                  } finally {
                        // Email triggered if the build is failed
                        emailext attachLog: true, body: 'JACOCO Code Coverage has FAILED ', subject: 'FAILED', to: 'abc@xyz.com'
                            }

                        }

                    }
                  }
        // Run the Sonar code quality scan
        // Try catch is required here as well ?
        stage('Sonar Code Quality Scan') {
            steps {
                dir('SB_Reference_App') {
                    withSonarQubeEnv('SonarQube') {
                        sh "mvn -B -f ./product-api -Dproject.settings=./product-api/sonar-project.properties -DskipTests=true -DBUILD_NUMBER=${imageVersion} sonar:sonar"
                    }
                }
            }
        }

        // Check to make sure the code passes the quality gate configured in Sonar
        stage('Sonar Code Quality Gate') {
            steps {
                dir('SB_Reference_App') {
                    // Wait a max of 2 minutes for the Quality Check to complete
                    //timeout(2) {
                    //    def qg = waitForQualityGate()
                    //    if (qg.status != 'OK') {
                    //        error "Pipeline aborted due to quality gate failure: ${qg.status}"
                    //    }
                    //}
                }
            }
        }

        // TODO_1st : using SSH plugin , as some teams here use Jboss , some tomcat , lets not use tomcat specirfic plugin , use variables qas diff teams diff values of server_ip , user_name , passwd
		// todo_1st_2 : use concurrency: 1 as shown in https://jenkins.io/doc/book/pipeline-as-code/   search for stage name: 'Production', concurrency: 1 deployment to test , stage or prod.
        stage('Deploy to Tomcat') {
            steps {
                dir('SB_Reference_App') {
                    // TODO_1st
                }
            }
        }

        stage('Run Selenium Test cases') {
            /* TODO_1st : IF FAIL stop set build status to fail , assume its written in JUnit or TestNG */
            steps {
                // task needs to be performed as Slenium test case
				//Parallel Execution of Tests in Chrome and Firefox. there will be failure here, reason steps parallel needs to be called and test steps
				//inside the parllel execution
				parallel (
				"Run_Selenium_Integration_Tests_in_Firefox" : {
					//sh "echo Firefox test steps"
				},
				"Run_Selenium_Integration_Tests_in_Chrome" : {
					//sh "echo Chrome test steps"
				}
              )

            }
            /*lavnish : send email once only at end of pipeline
			post {

                failure
                	emailext attachLog: true, body: '', subject: 'Failures', to: 'xyz@domain.com'
            }*/


        }

		stage('Archive') {
			// todo_1st_2 : archive in JFrog if all selenium test cases pass , in putnam JFrog is used its Open source.
			// https://www.jfrog.com/confluence/display/RTF/Working+With+Pipeline+Jobs+in+Jenkins
      // Below is the code for Jfrog archive, we need to configure the server details in Manage Jenkins
        steps {
          script {
            // Obtain an Artifactory server instance, defined in Jenkins --> Manage:
                    def server = Artifactory.server SERVER_ID

                    def uploadSpec = """{
                          "files": [
                              {
                                  "pattern": "/target/*.war",
                                  "target": "location on server"
                                }
                                    ]
                                      }"""
                    server.upload(uploadSpec)

                    // Publish the merged build-info to Artifactory
                    server.publishBuildInfo buildInfo1
          }
        }
		}

    }

	post {
       			always {
					// TODO_1st : send summary email with all Junit , Selenium & JAcoco numbers // lavnish : do with xunit
					publishHTML([allowMissing: true, alwaysLinkToLastBuild: false, keepAll: false, reportDir: 'location of test report dir', reportFiles: 'index.html', reportName: 'Selenium HTML Report', reportTitles: ''])
           			echo 'Post Build Steps !'
					deleteDir()
          cleanWs cleanWhenAborted: false, cleanWhenFailure: false, cleanWhenNotBuilt: false, cleanWhenUnstable: false
           // lavnish : must delete workspace at end of job , else space on jenkins server goes full
       			}
       			success {
					echo 'Jenkins Pipeline Successful !!'
       			}
       			failure {
					// TODO : lavnish : use extmail , as it allows html email
       				mail bcc: '', body: "<p>Job ${env.JOB_NAME} [${env.BUILD_NUMBER}] Branch [${Branch_Name}] for environment [${environment}]</p> <p> Check console output at <a href='${env.BUILD_URL}'>${env.JOB_NAME} ${env.BUILD_NUMBER}</a>  </p>", cc: '', from: '', replyTo: '', subject: "FAILED : Job ${env.JOB_NAME} [${env.BUILD_NUMBER}] Branch [${Branch_Name}] for environment [${environment}]", to: 'lavnish_lalchandani@putnam.com'
       			}
			}

}
