//Groovy Pipeline
node () { //node('worker_node')
   properties([
      parameters([
           gitParameter(branchFilter: 'origin/(.*)', defaultValue: 'development', name: 'BRANCH', type: 'PT_BRANCH'),
           choice(choices: ['DEV', 'QA' , 'BOTH'], name: 'SERVER_DEPLOYMENT_OPTION'),
           booleanParam(defaultValue: false,  name: 'RELEASE'),
           booleanParam(defaultValue: false,  name: 'WITH_ERROR')
      ]),
      disableConcurrentBuilds()
   ])
   
   def repoUrl = 'https://github.com/d-synchronized/ci-cd-demo.git'
   def previousPomVersion = ''
   def tagVersionCreated = ''
   try {
      stage('Checkout Source Code') { 
          echo "***Checking out source code from repo url ${repoUrl},branchName ${params.BRANCH}***"
          bat "git config user.name 'Dishant Anand'"
          bat "git config user.email d.synchronized@gmail.com"
          
          checkout([$class: 'GitSCM', 
                    branches: [[name: "*/${params.BRANCH}"]], 
                    extensions: [], 
                    userRemoteConfigs: [[credentialsId: 'github-credentials', url: "${repoUrl}"]]])
      }
      
      
      stage('Drop SNAPSHOT') {
          projectVersion = readMavenPom().getVersion()
          previousPomVersion = projectVersion
          if("${params.RELEASE}" ==  'true'){
             echo "About to release ${projectVersion}"
             bat "mvn versions:set -DremoveSnapshot -DgenerateBackupPoms=false"
             
             NEW_VERSION = readMavenPom().getVersion()
             
             if(NEW_VERSION != projectVersion){
                echo "Changed from ${projectVersion} to ${NEW_VERSION}"
                bat "mvn -U versions:set -DnewVersion=${NEW_VERSION}"
          
                withCredentials([usernamePassword(credentialsId: 'github-account', passwordVariable: 'GIT_PASSWORD', usernameVariable: 'GIT_USERNAME')]) {
                  bat "git add pom.xml"
                  bat "git commit -m \"Incrementing pom version from ${projectVersion} to ${NEW_VERSION}\""
                  bat "git push https://${env.GIT_USERNAME}:${env.GIT_PASSWORD}@github.com/d-synchronized/ci-cd-demo.git HEAD:${BRANCH}"
                }
             }else {
                echo "New Version matches the POM Version"
             }
          } else {
             echo "***DROP SNAPSHOT skipped for releaseType? ${params.RELEASE}***"
          } 
      }
      
      stage('Create TAG'){
          if("${params.RELEASE}" ==  'true'){
             echo "***Creating tag for the RELEASE***"
             
             LATEST_TAG_VERSION = sh(returnStdout:  true, script: 'git describe --abbrev=0 --tags').trim()
             echo "Latest tag version is ${LATEST_TAG_VERSION}"
             TAG_VERSION_BITS=LATEST_TAG_VERSION.tokenize("-")
             PORTION_AFTER_HYPHEN="${TAG_VERSION_BITS[1]}"
             PORTION_AFTER_HYPHEN_BITS=PORTION_AFTER_HYPHEN.tokenize(".")
             TAG_VERSION_MAJOR_BIT="${PORTION_AFTER_HYPHEN_BITS[0]}"
             TAG_VERSION_MAJOR_BIT = TAG_VERSION_MAJOR_BIT?.isInteger() ? TAG_VERSION_MAJOR_BIT.toInteger() : null
             echo "TAG_VERSION_MAJOR_BIT is ${TAG_VERSION_MAJOR_BIT}"
             
             projectVersion = readMavenPom().getVersion()
             VERSION_BITS=projectVersion.tokenize(".")
             VNUM1="${VERSION_BITS[0]}"
             VERSION_MAJOR_BIT = VNUM1?.isInteger() ? VNUM1.toInteger() : null
             echo "VERSION_MAJOR_BIT is ${TAG_VERSION_MAJOR_BIT}"
             
             NEW_TAG_VERSION = TAG_VERSION_MAJOR_BIT >= VERSION_MAJOR_BIT ? (VERSION_MAJOR_BIT + 1) + '.0.0' : projectVersion
             NEW_TAG = "RELEASE-${NEW_TAG_VERSION}"
             
             tagVersionCreated = NEW_TAG
             
             echo "NEW_TAG is ${NEW_TAG}"
             withCredentials([usernamePassword(credentialsId: 'github-account', passwordVariable: 'GIT_PASSWORD', usernameVariable: 'GIT_USERNAME')]) {
                bat "git tag -a ${NEW_TAG} -m \"pushing TAG VERSION ${NEW_TAG}\""
                bat "git push https://${env.GIT_USERNAME}:${env.GIT_PASSWORD}@github.com/d-synchronized/ci-cd-demo.git --tags"
             }//with credentials ends here
             echo "***Successfully created TAG ${NEW_TAG} in the repository for the current release***"
          } else {
              echo "***TAG creation skipped for the environment ${params.ENVIRONMENT}***"
          }
          
      }
      
   
     stage('Build & Deploy Artifact') {
          bat([script: 'mvn clean install deploy']) 
          milestone label: 'After Build', ordinal: 1
     }
     
     stage('Deploy Artifact') {
         DEPLOY_TO_QA = "${params.SERVER_DEPLOYMENT_OPTION}" == 'BOTH' || "${params.SERVER_DEPLOYMENT_OPTION}" == 'QA' ? true : false
         DEPLOY_TO_DEV = "${params.SERVER_DEPLOYMENT_OPTION}" == 'BOTH' || "${params.SERVER_DEPLOYMENT_OPTION}" == 'DEV' ? true : false
         if(DEPLOY_TO_DEV){
            echo "Deploying to Dev servers"
         }
         if(DEPLOY_TO_QA){
            echo "Deploying to QA servers"
         }
     }
     
     stage('Increment Development Version'){
          projectVersion = readMavenPom().getVersion()
          VERSION_BITS=projectVersion.tokenize(".")
          VNUM1="${VERSION_BITS[0]}"
          VNUM2="${VERSION_BITS[1]}"
          VNUM3="${VERSION_BITS[2]}"
          NEW_VERSION = ""
          if("${params.RELEASE}" ==  'true'){
             VNUM1= VNUM1?.isInteger() ? VNUM1.toInteger() + 1 : null
             NEW_VERSION="${VNUM1}.0.0-SNAPSHOT"
          } else {
             VNUM3_BITS = VNUM3.tokenize("-")
             VNUM3_INCR = VNUM3_BITS[0]?.isInteger() ? VNUM3_BITS[0].toInteger() + 1 : VNUM3_BITS[0]
             NEW_VERSION="${VNUM1}.${VNUM2}.${VNUM3_INCR}-SNAPSHOT"
          } 
          
          echo "Changed from ${projectVersion} to ${NEW_VERSION}"
          bat "mvn -U versions:set -DnewVersion=${NEW_VERSION}"
          
          withCredentials([usernamePassword(credentialsId: 'github-account', passwordVariable: 'GIT_PASSWORD', usernameVariable: 'GIT_USERNAME')]) {
              bat "git add pom.xml"
              bat "git commit -m \"Incrementing pom version/appending snapshot from ${projectVersion} to ${NEW_VERSION}\""
              bat "git push https://${env.GIT_USERNAME}:${env.GIT_PASSWORD}@github.com/d-synchronized/ci-cd-demo.git HEAD:${BRANCH}"
          }
       }
       
     
       currentBuild.result = 'SUCCESS'
       if("${WITH_ERROR}" == 'true'){
          error("Build failed because of this and that..")
       }
   } catch(Exception err) {
      echo "Error occurred while running the job '${env.JOB_NAME}'"
      currentBuild.result = 'FALIURE'
      revertParentPOM("${previousPomVersion}")
      if("${params.RELEASE}" == 'true'){
         deleteTag("${tagVersionCreated}")
      }
   } finally {
       //deleteDir()
       echo '***************************************************'
       echo '***************************************************'
       echo '****POST******BUILD*****ACTION*********START*******'
       //mail to: 'd.synchronized@gmail.com', cc: 'vision4cloud@gmail.com,d.xcption13@gmail.com', bcc: 'slayer4cloud@gmail.com', 
       //     body: "Status for ${env.JOB_NAME} (${env.JOB_URL}) is ${currentBuild.result}", 
       //     subject: "Status of pipeline : ${currentBuild.fullDisplayName}"
       emailext attachLog: true, 
                body: 'Status for ${env.JOB_NAME} (${env.JOB_URL}) is ${currentBuild.result}', 
                subject: 'Status of pipeline : ${currentBuild.fullDisplayName',
                to: 'd.synchronized@gmail.com,d.xcption13@gmail.com'
       echo '****POST******BUILD*****ACTION*********END*********'
       echo '***************************************************'
       echo '***************************************************'
   }
   
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
