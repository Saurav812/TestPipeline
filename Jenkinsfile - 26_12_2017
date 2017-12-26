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

    // This might work if we move to CloudBees, a much cleaner way of obtaining the pom.version - TODO
    //environment {
    //    VERSION = readMavenPom().getVersion()
    //}

    environment {
	    // the idea is to encapsulate all variables here.
        def server_ip_address_or_name = ""

		def server_username_name = ""

		def server_password = "" // there is a way to get this from jenkins please google.
	    // todo_2nd : ip address or server name of the server where we will deploy tomcat
		// i expect these def to be available in stages below , can you please see how to do that. or in worst case we will define as jenkins parameter. like ${Branch_Name} below
    }


stages {

	stage ('checkout') {
		steps {
			echo "checking out branch ${Branch_Name}"
			// NOTE : ${Branch_Name} is defined as pipeline parameter
			git branch: '${Branch_Name}', url: 'https://gitlab.putnaminv.com/public_cloud/SB_Reference_App.git'
		}
	}
        // Build and unit test the application
        stage('Build') {
		// todo_1st_2 : earlier this stage was 'Build/Unit/Integration Test' now separate it ... make 3 phases Build & Unit Test & integration test :
		// in build call mvn "clean install -DskipTests" not mvn clean package , see stage ('runTests') below to see how to run unit tests .... need to find how to run Integration ( i.e. selenium only later : as it needs to be run after deployment )
        //*******************************************
        // Build Seperated - Build, Unit Test and Integration Test ( steps to be put, pending)
        // FOr Integration Testing -
        	// We need to use Surefire to run Unit Test and Integration Test {Structure - Test.java(Unit Testing), Integration.java(ntegration test)}, Attached file for reference

            steps {
				          deleteDir()
                  dir('SB_Reference_App') {
                    // git url: 'https://gitlab.putnaminv.com/public_cloud/SB_Reference_App.git'
                    //sh "mvn -B -f ./product-api -Dmaven.test.failure.ignore=true -Dspring.profiles.active=integration clean package"
                    //sh "mvn -B -f ./product-api clean package"
                    sh mvn "clean install -DskipTests"

                    // Construct the imageVersion of the build from the pom.version and BUILD_NUMBER, this will be used later for tagging the docker image
// TODO_3rd : if Time taken by Unit test of previous build is MUCH MUCH LOWER ( by 50%) than current set warning message ( as some inefficient code is there )
                }
            }

        }

		stage ('Unit Test') {
			/* Call the Maven build with tests. */
			steps {
				mvn "install -Dmaven.test.failure.ignore=true"

				/* Archive the test results */
				junit '**/target/surefire-reports/TEST-*.xml'
        // Below will show the report on UI
        step([$class: 'JUnitResultArchiver', testResults: 'target/surefire-reports/TEST-*.xml'])

        //Parallel Execution of Tests in Chrome and Firefox. there will be failure here, reason steps parallel needs to be called and test steps
        //inside the parllel execution
            parallel (
              "Firefox" : {
                //sh "echo Firefox test steps"
              },
              "Chrome" : {
                //sh "echo Chrome test steps"
              }
              )
			}
			post {
          always {
                  junit 'target/surefire-reports/*.xml'
                  // Publish Reports
                  step([$class: 'XUnitBuilder',
                  thresholds: [[$class: 'FailedThreshold', unstableThreshold: '1']],
                  tools: [[$class: 'JUnitType', pattern: 'target/surefire-reports/**']]])

                  publishHTML([
                    allowMissing: true,
                    alwaysLinkToLastBuild: true,
                    keepAll: true, reportDir: 'target/surefire-reports',
                    reportFiles: '*.xml',
                    reportName: 'Coverage Report',
                    reportTitles: '']
          )
                  //Sending an email
                  emailext attachLog: true, body: 'This is a test Job ', subject: 'Passed', to: 'xyz@domain.com'
        }

          failure {
                  //TODO_1st - notify users when the Pipeline fails
      			    	emailext attachLog: true, body: '', subject: 'Failures', to: 'xyz@domain.com'
          }

			}

			// todo_1st_2 : use xUnit not junit plugin : step([$class: 'XUnitBuilder',thresholds: [ [$class: 'SkippedThreshold', failureThreshold: '0'], [$class: 'FailedThreshold', failureThreshold: '10']], tools: [[$class: 'JUnitType', pattern: 'reports/**']]]) see https://jenkins.io/blog/2016/10/31/xunit-reporting/

			// can we run this in parallel later ?
			// todo_2nd : run tests in parallel https://jenkins.io/blog/2016/06/16/parallel-test-executor-plugin/
		}

		stage ('Integration Test') {
		     steps {
		         //test to be put here
		         // Ex - mvn clean verify -P integration-test
		     }
      			post {
      			    failure
      			    	// notify users when the Integration Test fails
      			    	emailext attachLog: true, body: '', subject: 'Failures', to: 'xyz@domain.com'
      			}

		  }


        // Run the code coverage report
        stage('JaCoCo Code Coverage') {
            steps {
                dir('SB_Reference_App') {
                    //sh "${mvnHome}/bin/mvn -B -f ./product-api -Dspring.profiles.active=integration -Dmaven.test.failure.ignore=true prepare-package"
                    jacoco(classPattern: './product-api/target/**/classes', execPattern: './product-api/target/jacoco.exec', sourcePattern: './product-api/src/main/java')
                    // TODO_1st : if code coverage  is below threshold stop build & fail , is there a way show these numbers on UI  or record and send at end of email
                    jacoco buildOverBuild: true, changeBuildStatus: true, deltaBranchCoverage: '15', deltaClassCoverage: '20',
                    deltaComplexityCoverage: '20', deltaInstructionCoverage: '6', deltaLineCoverage: '25', deltaMethodCoverage: '35',
                    maximumBranchCoverage: '30', maximumClassCoverage: '50', maximumComplexityCoverage: '40', maximumInstructionCoverage: '10',
                    maximumLineCoverage: '50', maximumMethodCoverage: '60', minimumBranchCoverage: '20', minimumClassCoverage: '25',
                    minimumComplexityCoverage: '25', minimumInstructionCoverage: '7', minimumLineCoverage: '30', minimumMethodCoverage: '40'
                }
                  post {
                      // Notify user for the failure
                      failure {
                        emailext attachLog: true, body: '', subject: 'Failures', to: 'xyz@domain.com'
                      }
                  }

            }
        }

        // Run the Sonar code quality scan
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
                    withSonarQubeEnv('SonarQube') {
                        sh "mvn -B -f ./product-api -Dproject.settings=./product-api/sonar-project.properties -DskipTests=true -DBUILD_NUMBER=${imageVersion} sonar:sonar"
                    }
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
            }
            post {

                failure
                	emailext attachLog: true, body: '', subject: 'Failures', to: 'xyz@domain.com'
            }


        }

		stage('archive') {
			// todo_1st_2 : archive in JFrog if all selenium test cases pass , in putnam JFrog is used its Open source.
			// https://www.jfrog.com/confluence/display/RTF/Working+With+Pipeline+Jobs+in+Jenkins
		}

    }

	post {
       			always {
					// TODO_1st : send summary email with all Junit , Selenium & JAcoco numbers
					publishHTML([allowMissing: true, alwaysLinkToLastBuild: false, keepAll: false, reportDir: 'location of test report dir', reportFiles: 'index.html', reportName: 'Selenium HTML Report', reportTitles: ''])
           			echo 'Post Build Steps !'
					deleteDir()
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
