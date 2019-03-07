
pipeline {
    agent  { label 'master' }
    tools {
        maven 'Maven 3.6.0'
        jdk 'jdk8'
    }
    environment {
        APP_NAME = "carts"
        TAG = "neotysdevopdemo/${APP_NAME}"
        TAG_DEV = "${TAG}:${env.VERSION}-${env.BUILD_NUMBER}"
        NL_DT_TAG = "app:${env.APP_NAME},environment:dev"
        CARTS_ANOMALIEFILE = "$WORKSPACE/monspec/carts_anomalieDection.json"
        TAG_STAGING = "${TAG}-stagging:${env.VERSION}"
        DYNATRACEID = "${env.DT_ACCOUNTID}.live.dynatrace.com"
        DYNATRACEAPIKEY = "${env.DT_API_TOKEN}"
        NLAPIKEY = "${env.NL_WEB_API_KEY}"
        OUTPUTSANITYCHECK = "$WORKSPACE/infrastructure/sanitycheck.json"
        DYNATRACEPLUGINPATH = "$WORKSPACE/lib/DynatraceIntegration-3.0.1-SNAPSHOT.jar"
        GITORIGIN = "neotyslab"
        GROUP = "neotysdevopdemo"
        COMMIT = "DEV - ${ env.BUILD_NUMBER}"

    }
    stages {
        stage('Checkout') {
            agent { label 'master' }
            steps {
                git  url:'https://github.com/neotysdevopdemo/carts.git',
                        branch :'master'
            }
        }
        stage('Maven build') {
            agent { label 'master' }
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
            agent { label 'master' }
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
                    return env.BRANCH_NAME ==~ 'release/.*' || env.BRANCH_NAME ==~ 'master'
                }
            }
            agent { label 'master' }
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
            agent { label 'master' }
            steps {
                sh 'docker-compose -f infrastructure/infrastructure/neoload/lg/docker-compose.yml up -d'

            }

        }
        stage('Join Load Generators to Application') {
            agent { label 'master' }
            steps {
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


                echo "Waiting for the service to start..."
                sleep 100
                script {
                    neoloadRun executable: '/home/neoload/neoload/bin/NeoLoadCmd',
                            project: "$WORKSPACE/target/neoload/Carts_NeoLoad/Carts_NeoLoad.nlp",
                            testName: 'HealthCheck_carts_${BUILD_NUMBER}',
                            testDescription: 'HealthCheck_carts_${BUILD_NUMBER}',
                            commandLineOption: "-nlweb -loadGenerators $WORKSPACE/infrastructure/infrastructure/neoload/lg/lg.yaml -nlwebToken $NLAPIKEY -variables host=carts,port=80,basicPath=/carts/health",
                            scenario: 'DynatraceSanityCheck', sharedLicense: [server: 'NeoLoad Demo License', duration: 2, vuCount: 200],
                            trendGraphs: [
                                    [name: 'Limit test Catalogue API Response time', curve: ['CatalogueList>Actions>Get Catalogue List'], statistic: 'average'],
                                    'ErrorRate'
                            ]
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
                            commandLineOption: "-nlweb -loadGenerators $WORKSPACE/infrastructure/infrastructure/neoload/lg/lg.yaml -nlwebToken $NLAPIKEY -variables host=carts,port=80",
                            scenario: 'DYNATRACE_SANITYCHECK', sharedLicense: [server: 'NeoLoad Demo License', duration: 2, vuCount: 200],
                            trendGraphs: [
                                    [name: 'Limit test Catalogue API Response time', curve: ['CatalogueList>Actions>Get Catalogue List'], statistic: 'average'],
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
                    sh "git add ${OUTPUTSANITYCHECK}"
                    sh "git commit -m 'Update Sanity_Check_${BUILD_NUMBER} ${env.APP_NAME} version ${env.VERSION}'"
                    //  sh "git pull -r origin master"
                    //#TODO handle this exeption
                    sh "git push origin HEAD:master"

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
                    args '--user root -v /tmp:/tmp --network=lg_default'
                    dir 'neoload/controller'
                }
            }

            steps {
                script {
                    neoloadRun executable: '/home/neoload/neoload/bin/NeoLoadCmd',
                            project: "$WORKSPACE/target/neoload/Carts_NeoLoad/Carts_NeoLoad.nlp",
                            testName: 'FuncCheck_carts__${BUILD_NUMBER}',
                            testDescription: 'FuncCheck_carts__${BUILD_NUMBER}',
                            commandLineOption: "-nlweb -loadGenerators $WORKSPACE/infrastructure/infrastructure/neoload/lg/lg.yaml -nlwebToken $NLAPIKEY -variables host=carts,port=80",
                            scenario: 'Cart_Load', sharedLicense: [server: 'NeoLoad Demo License', duration: 2, vuCount: 200],
                            trendGraphs: [
                                    [name: 'Limit test Catalogue API Response time', curve: ['CatalogueList>Actions>Get Catalogue List'], statistic: 'average'],
                                    'ErrorRate'
                            ]
                }


            }
        }

        stage('Mark artifact for staging namespace') {
            when {
                expression {
                    return env.BRANCH_NAME ==~ 'release/.*'
                }
            }
            agent { label 'master' }
            steps {

                withCredentials([usernamePassword(credentialsId: 'dockerHub', passwordVariable: 'TOKEN', usernameVariable: 'USER')]) {
                    sh "docker login --username=${USER} --password=${TOKEN}"
                    sh "GROUP=${GROUP} COMMIT=${TAG_STAGING} ./scripts/push.sh"
                    sh "docker tag $TAG_DEV} ${TAG_STAGING}"
                    sh "docker push ${TAG_STAGING}"
                }

            }
        }
    }
    post {

        always {

                sh 'docker-compose -f infrastructure/infrastructure/neoload/lg/docker-compose.yml down'
                sh 'docker-compose -f docker-compose.yml down'

        }

    }

}

