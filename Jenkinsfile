
pipeline {
    agent  { label 'master' }
    tools {
        maven 'Maven 3.6.0'
        jdk 'jdk8'
    }
    environment {
        VERSION="0.1"
        APP_NAME = "carts"
        TAG = "neotysdevopsdemo/${APP_NAME}"
        TAG_DEV = "${TAG}:DEV-${VERSION}"
        NL_DT_TAG = "app:${env.APP_NAME},environment:dev"
        CARTS_ANOMALIEFILE = "$WORKSPACE/monspec/carts_anomalieDection.json"
        TAG_STAGING = "${TAG}-stagging:${VERSION}"
        DYNATRACEID = "${env.DT_ACCOUNTID}.live.dynatrace.com"
        DYNATRACEAPIKEY = "${env.DT_API_TOKEN}"
        NLAPIKEY = "${env.NL_WEB_API_KEY}"
        OUTPUTSANITYCHECK = "$WORKSPACE/infrastructure/sanitycheck.json"
        DYNATRACEPLUGINPATH = "$WORKSPACE/lib/DynatraceIntegration-3.0.1-SNAPSHOT.jar"
        GROUP = "neotysdevopsdemo"
        COMMIT = "DEV-${VERSION}"

    }
    stages {
        stage('Checkout') {
            agent { label 'master' }
            steps {
                git  url:"https://github.com/${GROUP}/${APP_NAME}.git",
                        branch :'master'
            }
        }
        stage('Maven build') {
            steps {
                sh "mvn -B clean package -DdynatraceId=$DYNATRACEID -DneoLoadWebAPIKey=$NLAPIKEY -DdynatraceApiKey=$DYNATRACEAPIKEY -Dtags=${NL_DT_TAG} -DoutPutReferenceFile=$OUTPUTSANITYCHECK -DcustomActionPath=$DYNATRACEPLUGINPATH -DjsonAnomalieDetectionFile=$CARTS_ANOMALIEFILE"
                // cette ligne est pour license ...mais il me semble que tu as license avec ton container  sh "chmod -R 777 $WORKSPACE/target/neoload/"
            }
        }
        stage('Docker build') {
            when {
                expression {
                    return env.BRANCH_NAME ==~ 'release/.*' || env.BRANCH_NAME ==~ 'master'
                }
            }
            steps {
                withCredentials([usernamePassword(credentialsId: 'dockerHub', passwordVariable: 'TOKEN', usernameVariable: 'USER')]) {
                    sh "cp ./target/*.jar ./docker/carts"
                    sh "docker build --build-arg BUILD_VERSION=${VERSION} --build-arg COMMIT=$COMMIT -t ${TAG_DEV} $WORKSPACE/docker/carts/"
                    sh "docker login --username=${USER} --password=${TOKEN}"
                    sh "docker push ${TAG_DEV}"
                }

            }
        }


        stage('Deploy to dev ') {
            when {
                expression {
                    return env.BRANCH_NAME ==~ 'release/.*' || env.BRANCH_NAME ==~ 'master'
                }
            }
            steps {
                sh "sed -i 's,TAG_TO_REPLACE,${TAG_DEV},' $WORKSPACE/docker-compose.yml"
                sh 'docker-compose -f $WORKSPACE/docker-compose.yml up -d'

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
                sh 'docker-compose -f $WORKSPACE/infrastructure/infrastructure/neoload/lg/doker-compose.yml up -d'

            }

        }
        stage('Join Load Generators to Application') {
            steps {
                sh 'docker network connect carts_master_default docker-lg1'
            }
        }
        stage('Run health check in dev') {
            agent {
                dockerfile {
                    args '--user root -v /tmp:/tmp --network=carts_master_default'
                    dir 'infrastructure/infrastructure/neoload/controller'
                    reuseNode true
                }
            }


            steps {


                echo "Waiting for the service to start..."
                sleep 250
                script {
                    neoloadRun executable: '/home/neoload/neoload/bin/NeoLoadCmd',
                            project: "$WORKSPACE/target/neoload/Carts_NeoLoad/Carts_NeoLoad.nlp",
                            testName: 'HealthCheck_carts_${VERSION}_${BUILD_NUMBER}',
                            testDescription: 'HealthCheck_carts_${VERSION}_${BUILD_NUMBER}',
                            commandLineOption: "-nlweb -L Population_BasicCheckTesting=$WORKSPACE/infrastructure/infrastructure/neoload/lg/remote.txt -L Population_Dynatrace_Integration=$WORKSPACE/infrastructure/infrastructure/neoload/lg/local.txt  -nlwebToken $NLAPIKEY -variables host=carts,port=80,basicPath=/health",
                            scenario: 'DynatraceSanityCheck', sharedLicense: [server: 'NeoLoad Demo License', duration: 2, vuCount: 200],
                            trendGraphs: [
                                    [name: 'Limit test Check API Response time', curve: ['BasicCheckTesting>Actions>BasicCheck'], statistic: 'average'],
                                    'ErrorRate'
                            ]
                }


            }
        }
        stage('Sanity Check') {
            agent {
                dockerfile {
                    args '--user root -v /tmp:/tmp --network=carts_master_default'
                    dir 'infrastructure/infrastructure/neoload/controller'
                    reuseNode true
                }
            }
            steps {
                script {
                    neoloadRun executable: '/home/neoload/neoload/bin/NeoLoadCmd',
                            project: "$WORKSPACE/target/neoload/Carts_NeoLoad/Carts_NeoLoad.nlp",
                            testName: 'DynatraceSanityCheck_carts_${VERSION}_${BUILD_NUMBER}',
                            testDescription: 'DynatraceSanityCheck_carts_${VERSION}_${BUILD_NUMBER}',
                            commandLineOption: "-nlweb -L  Population_Dynatrace_SanityCheck=$WORKSPACE/infrastructure/infrastructure/neoload/lg/local.txt -nlwebToken $NLAPIKEY -variables host=carts,port=80",
                            scenario: 'DYNATRACE_SANITYCHECK', sharedLicense: [server: 'NeoLoad Demo License', duration: 2, vuCount: 200],
                            trendGraphs: [
                                                      'ErrorRate'
                            ]
                }



                echo "push ${OUTPUTSANITYCHECK}"
                //---add the push of the sanity check---
                withCredentials([usernamePassword(credentialsId: 'git-credentials', passwordVariable: 'GIT_PASSWORD', usernameVariable: 'GIT_USERNAME')]) {
                    sh "git config --global user.email ${env.GITHUB_USER_EMAIL}"
                    sh "git config remote.origin.url https://github.com/${env.GITHUB_ORGANIZATION}/carts"
                    sh "git config --add remote.origin.fetch +refs/heads/*:refs/remotes/origin/*"
                    sh "git config remote.origin.url https://github.com/${env.GITHUB_ORGANIZATION}/carts"
                  //  sh "git add ${OUTPUTSANITYCHECK}"
                   // sh "git commit -m 'Update Sanity_Check_${BUILD_NUMBER} ${env.APP_NAME} '"
                    //  sh "git pull -r origin master"
                    //#TODO handle this exeption
                 //   sh "git push origin HEAD:master"

                }

            }
        }
        stage('Run functional check in dev') {
            when {
                expression {
                    return env.BRANCH_NAME ==~ 'release/.*' || env.BRANCH_NAME ==~ 'master'
                }
            }
            agent {
                dockerfile {
                    args '--user root -v /tmp:/tmp --network=carts_master_default'
                    dir 'infrastructure/infrastructure/neoload/controller'
                    reuseNode true
                }
            }

            steps {
                script {
                    neoloadRun executable: '/home/neoload/neoload/bin/NeoLoadCmd',
                            project: "$WORKSPACE/target/neoload/Carts_NeoLoad/Carts_NeoLoad.nlp",
                            testName: 'FuncCheck_carts__${VERSION}_${BUILD_NUMBER}',
                            testDescription: 'FuncCheck_carts__${VERSION}_${BUILD_NUMBER}',
                            commandLineOption: "-nlweb Population_AddItemToCart=$WORKSPACE/infrastructure/infrastructure/neoload/lg/remote.txt -L Population_Dynatrace_Integration=$WORKSPACE/infrastructure/infrastructure/neoload/lg/local.txt  -nlwebToken $NLAPIKEY -variables carts_host=carts,carts_port=80",
                            scenario: 'Cart_Load', sharedLicense: [server: 'NeoLoad Demo License', duration: 2, vuCount: 200],
                            trendGraphs: [
                                    [name: 'Limit test Carts API Response time', curve: ['AddItemToCart>Actions>AddItemToCart'], statistic: 'average'],
                                    'ErrorRate'
                            ]
                }


            }
        }

        stage('Mark artifact for staging namespace') {

            steps {

                withCredentials([usernamePassword(credentialsId: 'dockerHub', passwordVariable: 'TOKEN', usernameVariable: 'USER')]) {
                    sh "docker login --username=${USER} --password=${TOKEN}"
                    sh "docker tag ${TAG_DEV} ${TAG_STAGING}"
                    sh "docker push ${TAG_STAGING}"
                }

            }
        }
    }
    post {

        always {

                sh 'docker-compose -f $WORKSPACE/infrastructure/infrastructure/neoload/lg/doker-compose.yml down'
                sh 'docker-compose -f $WORKSPACE/docker-compose.yml down'
                  cleanWs()
                sh 'docker volume prune'
        }

    }

}

