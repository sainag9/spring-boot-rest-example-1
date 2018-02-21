// Run this node on a Maven Slave
	// Maven Slaves have JDK and Maven already installed
	node('maven') {
	  // Make sure your nexus_openshift_settings.xml
	  // Is pointing to your nexus instance
	  def mvnCmd = "mvn"
	
	  stage('Checkout Source') {
	    // Get Source Code from SCM (Git) as configured in the Jenkins Project
	    // Next line for inline script, "checkout scm" for Jenkinsfile from Gogs
	    //git 'http://gogs.xyz-gogs.svc.cluster.local:3000/CICDLabs/openshift-tasks.git'
	    checkout scm
	  }
	
	  // The following variables need to be defined at the top level and not inside
	  // the scope of a stage - otherwise they would not be accessible from other stages.
	  // Extract version and other properties from the pom.xml
	  def groupId    = getGroupIdFromPom("pom.xml")
	  def artifactId = getArtifactIdFromPom("pom.xml")
	  def version    = getVersionFromPom("pom.xml")
	
	  stage('Build war') {
	    echo "Building version ${version}"
	
	    sh "${mvnCmd} clean package -DskipTests"
	  }
	  stage('Unit Tests') {
	    echo "Unit Tests"
	    sh "${mvnCmd} test"
	  }
	  
	
	  stage('Build OpenShift Image') {
	    def newTag = "TestingCandidate-${version}"
	    echo "New Tag: ${newTag}"
	
	    // Copy the war file we just built and rename to ROOT.war
	    sh "cp ./target/spring-boot-rest-example-0.0.1-SNAPSHOT.war ./ROOT.war"
	
	    // Start Binary Build in OpenShift using the file we just published
	    // Replace xyz-tasks-dev with the name of your dev project
	    sh "oc project xyz-tasks-dev"
	    sh "oc start-build tasks --follow --from-file=./ROOT.war -n xyz-tasks-dev"
	
	    openshiftTag alias: 'false', destStream: 'tasks', destTag: newTag, destinationNamespace: 'xyz-tasks-dev', namespace: 'xyz-tasks-dev', srcStream: 'tasks', srcTag: 'latest', verbose: 'false'
	  }
	
	  stage('Deploy to Dev') {
	    // Patch the DeploymentConfig so that it points to the latest TestingCandidate-${version} Image.
	    // Replace xyz-tasks-dev with the name of your dev project
	    sh "oc project xyz-tasks-dev"
	    sh "oc patch dc tasks --patch '{\"spec\": { \"triggers\": [ { \"type\": \"ImageChange\", \"imageChangeParams\": { \"containerNames\": [ \"tasks\" ], \"from\": { \"kind\": \"ImageStreamTag\", \"namespace\": \"xyz-tasks-dev\", \"name\": \"tasks:TestingCandidate-$version\"}}}]}}' -n xyz-tasks-dev"
	
	    openshiftDeploy depCfg: 'tasks', namespace: 'xyz-tasks-dev', verbose: 'false', waitTime: '', waitUnit: 'sec'
	    openshiftVerifyDeployment depCfg: 'tasks', namespace: 'xyz-tasks-dev', replicaCount: '1', verbose: 'false', verifyReplicaCount: 'false', waitTime: '', waitUnit: 'sec'
	    openshiftVerifyService namespace: 'xyz-tasks-dev', svcName: 'tasks', verbose: 'false'
	  }
	
	  stage('Integration Test') {
	    // TBD: Proper test
	    // Could use the OpenShift-Tasks REST APIs to make sure it is working as expected.
	
	    def newTag = "ProdReady-${version}"
	    echo "New Tag: ${newTag}"
	
	    // Replace xyz-tasks-dev with the name of your dev project
	    openshiftTag alias: 'false', destStream: 'tasks', destTag: newTag, destinationNamespace: 'xyz-tasks-dev', namespace: 'xyz-tasks-dev', srcStream: 'tasks', srcTag: 'latest', verbose: 'false'
	  }// Blue/Green Deployment into Production
	  // -------------------------------------
	  def dest   = "tasks-green"
	  def active = ""
	
	  stage('Prep Production Deployment') {
	    // Replace xyz-tasks-dev and xyz-tasks-prod with
	    // your project names
	    sh "oc project xyz-tasks-prod"
	    sh "oc get route tasks -n xyz-tasks-prod -o jsonpath='{ .spec.to.name }' > activesvc.txt"
	    active = readFile('activesvc.txt').trim()
	    if (active == "tasks-green") {
	      dest = "tasks-blue"
	    }
	    echo "Active svc: " + active
	    echo "Dest svc:   " + dest
	  }
	  stage('Deploy new Version') {
	    echo "Deploying to ${dest}"
	
	    // Patch the DeploymentConfig so that it points to
	    // the latest ProdReady-${version} Image.
	    // Replace xyz-tasks-dev and xyz-tasks-prod with
	    // your project names.
	    sh "oc patch dc ${dest} --patch '{\"spec\": { \"triggers\": [ { \"type\": \"ImageChange\", \"imageChangeParams\": { \"containerNames\": [ \"$dest\" ], \"from\": { \"kind\": \"ImageStreamTag\", \"namespace\": \"xyz-tasks-dev\", \"name\": \"tasks:ProdReady-$version\"}}}]}}' -n xyz-tasks-prod"
	
	    openshiftDeploy depCfg: dest, namespace: 'xyz-tasks-prod', verbose: 'false', waitTime: '', waitUnit: 'sec'
	    openshiftVerifyDeployment depCfg: dest, namespace: 'xyz-tasks-prod', replicaCount: '1', verbose: 'false', verifyReplicaCount: 'true', waitTime: '', waitUnit: 'sec'
	    openshiftVerifyService namespace: 'xyz-tasks-prod', svcName: dest, verbose: 'false'
	  }
	  stage('Switch over to new Version') {
	    input "Switch Production?"
	
	    // Replace xyz-tasks-prod with the name of your
	    // production project
	    sh 'oc patch route tasks -n xyz-tasks-prod -p \'{"spec":{"to":{"name":"' + dest + '"}}}\''
	    sh 'oc get route tasks -n xyz-tasks-prod > oc_out.txt'
	    oc_out = readFile('oc_out.txt')
	    echo "Current route configuration: " + oc_out
	  }}
	
	// Convenience Functions to read variables from the pom.xml
	def getVersionFromPom(pom) {
	  def matcher = readFile(pom) =~ '<version>(.+)</version>'
	  matcher ? matcher[0][1] : null
	}
	def getGroupIdFromPom(pom) {
	  def matcher = readFile(pom) =~ '<groupId>(.+)</groupId>'
	  matcher ? matcher[0][1] : null
	}
	def getArtifactIdFromPom(pom) {
	  def matcher = readFile(pom) =~ '<artifactId>(.+)</artifactId>'
	  matcher ? matcher[0][1] : null
}
