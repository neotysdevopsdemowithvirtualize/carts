@Library('dynatrace@master') _
pipeline {
  agent none
 tools {
       maven 'Maven 3.6.0'
       jdk 'jdk8'
   }
  environment {
    APP_NAME = "carts"
    TAG = "${env.DOCKER_REGISTRY_URL}:5000/neotysdevopdemo/${APP_NAME}"
    TAG_DEV = "${env.TAG}:${env.VERSION}-${env.BUILD_NUMBER}"
    NL_DT_TAG="app:${env.APP_NAME},environment:dev"
    CARTS_ANOMALIEFILE="$WORKSPACE/monspec/carts_anomalieDection.json"
    TAG_STAGING = "${env.TAG}:${env.VERSION}"
    DYNATRACEID="${env.DT_ACCOUNTID}.live.dynatrace.com"
    DYNATRACEAPIKEY="${env.DT_API_TOKEN}"
    NLAPIKEY="${env.NL_WEB_API_KEY}"
    OUTPUTSANITYCHECK="$WORKSPACE/infrastructure/sanitycheck.json"
    DYNATRACEPLUGINPATH="$WORKSPACE/lib/DynatraceIntegration-3.0.1-SNAPSHOT.jar"
    GITORIGIN="neotyslab"
	GROUP=neotysdevopdemo
	COMMIT=DEV-${env.BUILD_NUMBER}

  }
  stages {
    stage('Maven build') {
      steps {
        
         agent { label 'master'} {
		  git(branch:'master',
              url:'https://github.com/neotysdevopsdemo/carts')
         checkout scm
          sh "mvn -B clean package -DdynatraceId=$DYNATRACEID -DneoLoadWebAPIKey=$NLAPIKEY -DdynatraceApiKey=$DYNATRACEAPIKEY -Dtags=${NL_DT_TAG} -DoutPutReferenceFile=$OUTPUTSANITYCHECK -DcustomActionPath=$DYNATRACEPLUGINPATH -DjsonAnomalieDetectionFile=$CARTS_ANOMALIEFILE"
        // cette ligne est pour license ...mais il me semble que tu as license avec ton container  sh "chmod -R 777 $WORKSPACE/target/neoload/"
        }
      }
    }
    stage('Docker build') {
      when {
        expression {
          return env.BRANCH_NAME ==~ 'release/.*' || env.BRANCH_NAME ==~'master'
        }
      }
      steps {
      withCredentials([usernamePassword(credentialsId: 'dockerHub', passwordVariable: 'TOKEN', usernameVariable: 'USER')]) {

         sh "./scripts/build.sh GROUP=${GROUP} COMMIT=${COMMIT}"
         sh "docker login --username=${USER} --password=${TOKEN}"
         sh "docker push ${TAG_DEV}"
		 }
         sh "sed -i 's/TAG_TO_REPLACE/${TAG_DEV}/'  docker-compose.yml"
      }
    }
    
    
    stage('Deploy to dev ') {
      when {
        expression {
          return env.BRANCH_NAME ==~ 'release/.*' || env.BRANCH_NAME ==~'master'
        }
      }
      steps {
        
           sh 'docker-compose -f docker-compose.yml up -d'
       
      }
    }

    /*stage('DT Deploy Event') {
      when {
          expression {
          return env.BRANCH_NAME ==~ 'release/.*' || env.BRANCH_NAME ==~'master'
          }
      }
      steps {
          createDynatraceDeploymentEvent(
          envId: 'Dynatrace Tenant',
          tagMatchRules: [
              [
              meTypes: [
                  [meType: 'SERVICE']
              ],
              tags: [
                  [context: 'CONTEXTLESS', key: 'app', value: "${env.APP_NAME}"],
                  [context: 'CONTEXTLESS', key: 'environment', value: 'dev']
              ]
              ]
          ])
      }
    }*/
    stage('Start NeoLoad infrastructure') {

            steps {
                     sh 'docker-compose -f infrastructure/infrastructure/neoload/lg/docker-compose.yml up -d'
            stash includes: 'neoload/lg/lg.yaml', name: 'LG'
            stash includes: 'neoload/test/microservices.yaml', name: 'microservices'
            stash includes: 'neoload/test/data/tokens.csv', name: 'tokens'
                    }

    }
	stage('Join Load Generators to Application') {
      agent{label 'master'}
      steps{
        sh 'docker network connect docker-compose_default docker-lg1'
      }
    }
    stage('Run health check in dev') {
      agent {
        dockerfile {
          args '--user root -v /tmp:/tmp --network=lg_default'
          dir 'neoload/controller'
        }
      }
    

       steps {

         container('neoload') {
             echo "Waiting for the service to start..."
             sleep 100
             script {
				  neoloadRun executable: '/home/neoload/neoload/bin/NeoLoadCmd',
					project: "$WORKSPACE/target/neoload/Carts_NeoLoad/Carts_NeoLoad.nlp",
					testName: 'HealthCheck_carts_${BUILD_NUMBER}',
					testDescription: 'HealthCheck_carts_${BUILD_NUMBER}',
					commandLineOption: "-nlweb -loadGenerators $WORKSPACE/infrastructure/infrastructure/neoload/lg/lg.yaml -nlwebToken $NLAPIKEY -variables host=${env.APP_NAME}.dev.svc,port=80,basicPath=/carts/health",
					scenario: 'DynatraceSanityCheck', sharedLicense: [server: 'NeoLoad Demo License', duration: 2, vuCount: 200],
					trendGraphs: [
						[name: 'Limit test Catalogue API Response time', curve: ['CatalogueList>Actions>Get Catalogue List'], statistic: 'average'],
						'ErrorRate'
						]
			}
			 
			
         }
      }
    }
    stage('Sanity Check') {
		agent {
			dockerfile {
			  args '--user root -v /tmp:/tmp --network=lg_default'
			  dir 'neoload/controller'
			}
       }
          steps {
			  script {
				  neoloadRun executable: '/home/neoload/neoload/bin/NeoLoadCmd',
					project: "$WORKSPACE/target/neoload/Carts_NeoLoad/Carts_NeoLoad.nlp",
					testName: 'DynatraceSanityCheck_carts_${BUILD_NUMBER}',
					testDescription: 'DynatraceSanityCheck_carts_${BUILD_NUMBER}',
					commandLineOption: "-nlweb -loadGenerators $WORKSPACE/infrastructure/infrastructure/neoload/lg/lg.yaml -nlwebToken $NLAPIKEY -variables host=${env.APP_NAME}.dev,port=80",
					scenario: 'DYNATRACE_SANITYCHECK', sharedLicense: [server: 'NeoLoad Demo License', duration: 2, vuCount: 200],
					trendGraphs: [
						[name: 'Limit test Catalogue API Response time', curve: ['CatalogueList>Actions>Get Catalogue List'], statistic: 'average'],
						'ErrorRate'
						]
				}
              
            
            
               echo "push ${OUTPUTSANITYCHECK}"
               //---add the push of the sanity check---
               withCredentials([usernamePassword(credentialsId: 'git-credentials-acm', passwordVariable: 'GIT_PASSWORD', usernameVariable: 'GIT_USERNAME')]) {
                   sh "git config --global user.email ${env.GITHUB_USER_EMAIL}"
                   sh "git config remote.origin.url https://github.com/${env.GITHUB_ORGANIZATION}/carts"
                   sh "git config --add remote.origin.fetch +refs/heads/*:refs/remotes/origin/*"
                   sh "git config remote.origin.url https://github.com/${env.GITHUB_ORGANIZATION}/carts"
                   sh "git add ${OUTPUTSANITYCHECK}"
                   sh "git commit -m 'Update Sanity_Check_${BUILD_NUMBER} ${env.APP_NAME} version ${env.VERSION}'"
                 //  sh "git pull -r origin master"
                 //#TODO handle this exeption
                  // sh "git push origin HEAD:master"
               
             }

          }
    }
    stage('Run functional check in dev') {
          when {
            expression {
              return env.BRANCH_NAME ==~ 'release/.*' || env.BRANCH_NAME ==~'master'
            }
          }

          steps {
               container('neoload') {
                 script {

                     status =sh(script:"/neoload/bin/NeoLoadCmd -project $WORKSPACE/target/neoload/Carts_NeoLoad/Carts_NeoLoad.nlp -testResultName FuncCheck_carts__${BUILD_NUMBER} -description FuncCheck_carts__${BUILD_NUMBER} -nlweb -L  Population_AddItemToCart=$WORKSPACE/infrastructure/infrastructure/neoload/lg/remote.txt -L Population_Dynatrace_Integration=$WORKSPACE/infrastructure/infrastructure/neoload/lg/local.txt -nlwebToken $NLAPIKEY -variables carts_host=${env.APP_NAME}.dev.svc,carts_port=80 -launch Cart_Load -noGUI", returnStatus: true)
                      if (status != 0) {
                        currentBuild.result = 'FAILED'
                        error "Load Test on cart."
                      }
                 }
             }


          }
    }

    stage('Mark artifact for staging namespace') {
      when {
        expression {
          return env.BRANCH_NAME ==~ 'release/.*'
        }
      }
      steps {
         
          withCredentials([usernamePassword(credentialsId: 'dockerHub', passwordVariable: 'TOKEN', usernameVariable: 'USER')]) {
            sh "docker login --username=${USER} --password=${TOKEN}"
			sh "GROUP=${GROUP} COMMIT=${env.TAG_STAGING} ./scripts/push.sh
            sh "docker tag ${env.TAG_DEV} ${env.TAG_STAGING}"
            sh "docker push ${env.TAG_STAGING}"
          }
        
      }
    }
    stage('Deploy to staging') {
      when {
        beforeAgent true
        expression {
          return env.BRANCH_NAME ==~ 'release/.*'
        }
      }
      steps {
        build job: "k8s-deploy-staging",
          parameters: [
            string(name: 'APP_NAME', value: "${env.APP_NAME}"),
            string(name: 'TAG_STAGING', value: "${env.TAG_STAGING}"),
            string(name: 'VERSION', value: "${env.VERSION}")
          ]
      }
    }
  }
  post {
      always {
        stage('Stop Infrastructure') {
      parallel{
        stage('Stop NeoLoad infrastructure') {
          agent { label 'master' }
          steps {
            sh 'docker-compose -f neoload/lg/docker-compose.yml down'
          }
        }
        stage('Stop application'){
          agent { label 'master'}
          steps {
            git(branch:'master',
              url:'https://github.com/microservices-demo/microservices-demo')
            sh 'docker-compose -f deploy/docker-compose/docker-compose.yml down'
          }
        }
      }
    }
      }

    }
}

