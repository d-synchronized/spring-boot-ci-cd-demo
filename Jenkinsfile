//Groovy Pipeline
node () { //node('worker_node')
   properties([
      parameters([
           gitParameter(branchFilter: 'origin/(.*)', defaultValue: 'development', name: 'BRANCH', type: 'PT_BRANCH'),
           string(defaultValue: '', name: 'VERSION', trim: true),
           choice(choices: ['DEV', 'QA' , 'PROD'], name: 'ENVIRONMENT'),
           string(defaultValue: 'http://localhost:8082/', name: 'SERVER', trim: true),
           booleanParam(defaultValue: false,  name: 'ROLLBACK'),
           booleanParam(defaultValue: false,  name: 'DEPLOY_FROM_REPO')
      ]),
      disableConcurrentBuilds()
   ])
   
   def server
   def rtMaven = Artifactory.newMavenBuild()
   def buildInfo
   def repoUrl = 'https://github.com/d-synchronized/spring-boot-ci-cd-demo.git'
   def devBuildDownloadFolder
   def qaBuildDownloadFolder
   
   def targetFolder
   def pattern
   try {
      stage('Clone') { 
         echo "***Checking out source code from repo url ${repoUrl},branchName ${params.BRANCH}, deploy from repo ${params.DEPLOY_FROM_REPO}***"
         checkout([$class: 'GitSCM', 
               branches: [[name: "*/${params.BRANCH}"]], 
               extensions: [], 
               userRemoteConfigs: [[credentialsId: 'github-credentials', url: "${repoUrl}"]]])
      }
      
      stage ('Artifactory Configuration') {
        // Obtain an Artifactory server instance, defined in Jenkins --> Manage Jenkins --> Configure System:
        server = Artifactory.server 'DSYNC_JFROG_INSTANCE'

        // Tool name from Jenkins configuration
        rtMaven.tool = 'MAVEN_BUILD_TOOL'
        rtMaven.deployer releaseRepo: 'cetera-maven-releases', snapshotRepo: 'cetera-maven-snapshots', server: server
        //rtMaven.resolver releaseRepo: 'cetera-maven-virtual-releases', snapshotRepo: 'cetera-maven-virtual-snapshots', server: server
        buildInfo = Artifactory.newBuildInfo()
      }
      
      
      stage('Build & Deploy') {
         DEPLOY_TO_PROD = "${params.ENVIRONMENT}"  == 'PROD' ? true : false
         DEPLOY_TO_QA = "${params.ENVIRONMENT}" == 'QA' ? true : false
         DEPLOY_TO_DEV = "${params.ENVIRONMENT}"  == 'DEV' ? true : false
         
         DEPLOY_FROM_REPO = "${params.DEPLOY_FROM_REPO}" == 'false' ? false : true
         pom = readMavenPom file: 'pom.xml'
         if("${params.BRANCH}" == 'development'){
            devBuildDownloadFolder = "${pom.artifactId}/SNAPSHOTS/${pom.version}"
            if(DEPLOY_TO_DEV && !DEPLOY_FROM_REPO){
               echo "Building SNAPSHOT Artifact"
               rtMaven.run pom: 'pom.xml', goals: 'clean install', buildInfo: buildInfo
               server.publishBuildInfo buildInfo
               
               echo "Dropping SNAPSHOT from the version"
               bat "mvn versions:set -DremoveSnapshot -DgenerateBackupPoms=false"
               echo "Building RELEASE Artifact"
               rtMaven.run pom: 'pom.xml', goals: 'clean install', buildInfo: buildInfo
               server.publishBuildInfo buildInfo
            } else if(DEPLOY_TO_QA || DEPLOY_TO_PROD){
               echo "Dropping SNAPSHOT from the version"
               bat "mvn versions:set -DremoveSnapshot -DgenerateBackupPoms=false"
               
               pom = readMavenPom file: 'pom.xml'
               qaBuildDownloadFolder = "${pom.artifactId}/RELEASES/${pom.version}"
            }  
         }//if development branch ends here
         else{
            devBuildDownloadFolder = "${pom.artifactId}/SNAPSHOTS/${pom.version}"
            if(DEPLOY_TO_QA || DEPLOY_TO_PROD){
               echo "Dropping SNAPSHOT from the version"
               bat "mvn versions:set -DremoveSnapshot -DgenerateBackupPoms=false"
               
               pom = readMavenPom file: 'pom.xml'
               qaBuildDownloadFolder = "${pom.artifactId}/RELEASES/${pom.version}"
            }
            rtMaven.deployer.deployArtifacts = false
            rtMaven.run pom: 'pom.xml', goals: 'clean install', buildInfo: buildInfo
         }
     }
     
     
     stage('Download Artifact') {
         pom = readMavenPom file: 'pom.xml'
         VERSION_REQUESTED = "${params.VERSION}"  != '' ? true : false
         
         VERSION_STRING = VERSION_REQUESTED ? "${pom.version}" : "${params.VERSION}"
         echo "downloading artifact"
           
         if(DEPLOY_TO_DEV) {
             targetFolder = "${pom.artifactId}/SNAPSHOTS/${VERSION_STRING}/"
             pattern = "cetera-maven-snapshots/com/example/${pom.artifactId}/${params.VERSION}/${pom.artifactId}-*.war"
         }else{
             targetFolder = "${pom.artifactId}/RELEASES/${VERSION_STRING}/"
             pattern = "cetera-maven-releases/com/example/${pom.artifactId}/${params.VERSION}/${pom.artifactId}-*.war"
         }
         
         echo '${pattern}'
         def downloadSpec = """{
                                  "files": [
                                              {
                                                "pattern": "${pattern}",
                                                "target": "${targetFolder}",
                                                "recursive": "true",
                                                "flat" : "true",
                                                "build" : "${env.JOB_NAME}/LATEST"
                                              }
                                           ]
                            }"""
         def failNoOp
         buildInfo = server.download spec: downloadSpec, failNoOp: true
         if(!failNoOp){
            deploy adapters: [tomcat8(url: "${SERVER}", credentialsId: 'tomcat')], war: "${targetFolder}/*.war", contextPath: "${pom.artifactId}"
         }
     }//Download Artifact ends here
      
     
     stage('Publish To Application/Web Server') {
         pom = readMavenPom file: 'pom.xml'
         DEPLOY_TO_PROD = "${params.ENVIRONMENT}"  == 'PROD' ? true : false
         DEPLOY_TO_QA = "${params.ENVIRONMENT}" == 'QA' ? true : false
         DEPLOY_TO_DEV = "${params.ENVIRONMENT}"  == 'DEV' ? true : false
         
         if(false){
         if("${params.BRANCH}" == 'development'){
            def failNoOp
            if(DEPLOY_TO_DEV) {
               def downloadSpec = readFile 'download-snapshots.json'
               buildInfo = server.download spec: downloadSpec, failNoOp: true
               if(!failNoOp){
                 deploy adapters: [tomcat8(url: "${SERVER}", credentialsId: 'tomcat')], war: "${devBuildDownloadFolder}/*.war", contextPath: "${pom.artifactId}"
               }
           }else{
              def downloadSpec = readFile 'download-releases.json'
              buildInfo = server.download spec: downloadSpec, failNoOp: true
              echo "${buildInfo}"
              if(!failNoOp){
                 echo "${qaBuildDownloadFolder}"
                 deploy adapters: [tomcat8(url: "${SERVER}", credentialsId: 'tomcat')], war: "${qaBuildDownloadFolder}/*.war" , contextPath: "${pom.artifactId}"
              }
           }
         }//if development branch ends here
         else{
            targetFolder = "${pom.artifactId}/RELEASES/${pom.version}"
            deploy adapters: [tomcat8(url: "${SERVER}", credentialsId: 'tomcat')], war: "target/*.war" , contextPath: "${pom.artifactId}"
         }}//false if ends here
     }//publish stage ends here
     
       currentBuild.result = 'SUCCESS'
   } catch(Exception err) {
      echo "Error occurred while running the job '${env.JOB_NAME}' , $err"
      currentBuild.result = 'FALIURE'
      //revertParentPOM("${previousPomVersion}")
      if("${params.RELEASE}" == 'true'){
         deleteTag("${tagVersionCreated}")
      }
   } finally {
       //deleteDir()
   }
   
}