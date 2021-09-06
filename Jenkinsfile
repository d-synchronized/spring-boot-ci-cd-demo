//Groovy Pipeline
node () { //node('worker_node')


   properties([
      parameters([
           gitParameter(branchFilter: 'origin/(.*)', defaultValue: 'development', name: 'BRANCH', type: 'PT_BRANCH'),
           //string(defaultValue: '', name: 'VERSION', trim: true),
           choice(choices: ['DEV', 'QA' , 'PROD'], name: 'ENVIRONMENT'),
           string(defaultValue: 'dapplnl064', name: 'SERVER', trim: true),
           booleanParam(defaultValue: false,  name: 'ROLLBACK'),
           booleanParam(defaultValue: false,  name: 'DEPLOY_FROM_REPO')
      ]),
      disableConcurrentBuilds()
   ])
   
   def server
   def rtMaven = Artifactory.newMavenBuild()
   def buildInfo
   def repoUrl = 'https://github.com/d-synchronized/spring-boot-ci-cd-demo.git'
   try {
      stage('Clone') { 
         echo "***Checking out source code from repo url ${repoUrl},branchName ${params.BRANCH}***"
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
         
         VERSION_SET = "${params.VERSION}" == '' ? false : true
         
         if(DEPLOY_TO_DEV && !DEPLOY_FROM_REPO){
            echo "*******Build & Deploy, Version Set  ${VERSION_SET} , Environment ${params.ENVIRONMENT}********"         
             
            echo "Building SNAPSHOT Artifact"
            rtMaven.run pom: 'pom.xml', goals: 'clean install', buildInfo: buildInfo
            server.publishBuildInfo buildInfo
            
            echo "Dropping SNAPSHOT from the version"
            bat "mvn versions:set -DremoveSnapshot -DgenerateBackupPoms=false"
            
            echo "Building RELEASE Artifact"
            rtMaven.run pom: 'pom.xml', goals: 'clean install', buildInfo: buildInfo
            server.publishBuildInfo buildInfo
         } else if(DEPLOY_TO_QA || DEPLOY_TO_PROD){
              echo "*******Skipping Build & Deploy, Version Set  ${VERSION_SET} , Environment ${params.ENVIRONMENT}********"
              echo "Dropping SNAPSHOT from the version"
              bat "mvn versions:set -DremoveSnapshot -DgenerateBackupPoms=false"
          }     
     }
      
     
     stage('Publish To Application/Web Server') {
         pom = readMavenPom file: 'pom.xml'
         echo "artifact is ${pom.artifactId}"
         echo "artifact is ${pom.version}"
         
         DEPLOY_TO_PROD = "${params.ENVIRONMENT}"  == 'PROD' ? true : false
         DEPLOY_TO_QA = "${params.ENVIRONMENT}" == 'QA' ? true : false
         DEPLOY_TO_DEV = "${params.ENVIRONMENT}"  == 'DEV' ? true : false
         
         def failNoOp
         if(DEPLOY_TO_DEV) {
            def downloadSpec = readFile 'download-snapshots.json'
            buildInfo = server.download spec: downloadSpec, failNoOp: true
            if(!failNoOp){
               def targetFolder = "${pom.artifactId}/SNAPSHOTS/${pom.version}"
               echo "${targetFolder}"
               deploy adapters: [tomcat8(url: 'http://localhost:8082/', credentialsId: 'tomcat')], war: "${targetFolder}/*.war", contextPath: "${pom.artifactId}"
            }
         }else{
            def downloadSpec = readFile 'download-releases.json'
            buildInfo = server.download spec: downloadSpec, failNoOp: true
            if(!failNoOp){
               def targetFolder = "${pom.artifactId}/SNAPSHOTS/${pom.version}"
               deploy adapters: [tomcat8(url: 'http://localhost:8082/', credentialsId: 'tomcat')], war: "${targetFolder}/*.war" , contextPath: "${pom.artifactId}"
            }
         }
     }
     
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

def downloadArtifactory(String localPath, String repository, String remotePath) {
    def server = Artifactory.server 'DSYNC_JFROG_INSTANCE'
    
    //def downloadSpec = readFile 'aql-download.json'
    def downloadSpec = readFile 'download.json'
    def uploadSpec = readFile 'props-upload.json'
    //echo "${downloadSpec}"
    //echo "Artifactory Download: ${repository}/${remotePath} -> ${localPath}"

    def buildInfo2 = server.download spec: downloadSpec, buildInfo: buildInfo
    return buildInfo2
}

def deleteTag(String tagVersionCreated) { 
      echo "deleting the TAG ${tagVersionCreated}"
      withCredentials([usernamePassword(credentialsId: 'github-account', passwordVariable: 'GIT_PASSWORD', usernameVariable: 'GIT_USERNAME')]) {
          bat "git push --delete https://${env.GIT_USERNAME}:${env.GIT_PASSWORD}@github.com/d-synchronized/ci-cd-demo.git ${tagVersionCreated}"
      }
}
   
def revertParentPOM(String previousPomVersion) {
      echo "reverting pom version to ${previousPomVersion}"
      bat "mvn -U versions:set -DnewVersion=${previousPomVersion}"
      withCredentials([usernamePassword(credentialsId: 'github-account', passwordVariable: 'GIT_PASSWORD', usernameVariable: 'GIT_USERNAME')]) {
          bat "git add pom.xml"
          bat "git commit -m \"Reverting pom version from ${NEW_VERSION} to ${projectVersion} \""
          bat "git push https://${env.GIT_USERNAME}:${env.GIT_PASSWORD}@github.com/d-synchronized/ci-cd-demo.git HEAD:${BRANCH}"
      }
}
