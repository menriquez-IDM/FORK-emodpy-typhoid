podTemplate(
    //idleMinutes : 30,
    podRetention : onFailure(),
    activeDeadlineSeconds : 3600,
    containers: [
        containerTemplate(
            name: 'dtk-rpm-builder', 
            image: 'docker-production.packages.idmod.org/idm/dtk-rpm-builder:0.1',
            command: 'sleep', 
            args: '30d'
            )
  ]) {
  node(POD_LABEL) {
    container('dtk-rpm-builder'){
		def build_ok = true
		stage('Cleanup Workspace') {
			cleanWs()
			echo "Cleaned Up Workspace For Project"
			echo "${params.BRANCH}"
		}
		stage('Code Checkout') {
			if (env.CHANGE_ID) {
				echo "I execute on the pull request ${env.CHANGE_ID}"
				checkout([$class: 'GitSCM',
							branches: [[name: "pr/${env.CHANGE_ID}/head"]],
							doGenerateSubmoduleConfigurations: false,
							extensions: [],
							gitTool: 'Default',
							submoduleCfg: [],
							userRemoteConfigs: [[refspec: '+refs/pull/*:refs/remotes/origin/pr/*', credentialsId: '704061ca-54ca-4aec-b5ce-ddc7e9eab0f2', url: 'git@github.com:InstituteforDiseaseModeling/emodpy-typhoid.git']]])
			} else {
				echo "I execute on the ${env.BRANCH_NAME} branch"
				git branch: "${env.BRANCH_NAME}",
				credentialsId: '704061ca-54ca-4aec-b5ce-ddc7e9eab0f2',
				url: 'git@github.com:InstituteforDiseaseModeling/emodpy-typhoid.git'
			}
		}
		stage('Prepare') {
			sh 'python3 -m pip install --upgrade pip'
			sh "pip3 install wheel"
			sh 'python3 -m pip install --upgrade setuptools'
			sh 'pip3 freeze'
		}
		stage('Build') {
			sh 'pwd'
			sh 'ls -a'
			sh 'python3 package_setup.py bdist_wheel'
			 
		}
		stage('Install') {
			def curDate = sh(returnStdout: true, script: "date").trim()
			echo "The current date is ${curDate}"
			
			echo "I am installing emodpy-typhoid from wheel file built from code"
			def wheelFile = sh(returnStdout: true, script: "find ./dist -name '*.whl'").toString().trim()
			//def wheelFile = sh(returnStdout: true, script: "python3 ./.github/scripts/get_wheel_filename.py --package-file package_setup.py").toString().trim()
			echo "This is the package file: ${wheelFile}"
			sh "pip3 install $wheelFile --index-url=https://packages.idmod.org/api/pypi/pypi-production/simple"
			
			sh "pip3 install dataclasses"
			sh 'pip3 install keyrings.alt'
			sh "pip3 freeze"
		}
		stage('Login') {
				withCredentials([string(credentialsId: 'Comps_emodpy_user', variable: 'user'), string(credentialsId: 'Comps_emodpy_password', variable: 'password'),
								 string(credentialsId: 'Bamboo_id', variable: 'bamboo_user'), string(credentialsId: 'Bamboo', variable: 'bamboo_password')]) {
					sh 'python3 .dev_scripts/create_auth_token_args.py --comps_url https://comps2.idmod.org --username $user --password $password'
					sh 'python3 .dev_scripts/create_auth_token_args.py --comps_url https://comps.idmod.org --username yechen --password $password'
					dir('tests/workflow_testing') {
						sh 'python3 bamboo_login_with_arguments.py -u $bamboo_user -p $bamboo_password'
					}
				}
			}
			
		try{
			stage('Unit Test') {
				echo "Running Unit test Tests"
				dir('tests/unittests') {
					sh "pip3 install unittest-xml-reporting"
					sh 'python3 -m xmlrunner discover'
					junit '*.xml'
				}
			}
		} catch(e) {
			build_ok = false
			echo e.toString()  
		}
		
		try{
			stage('Workflow Tests') {
				echo "Running Workflow Tests"
				dir('tests/workflow_testing') {
					sh 'python3 -m xmlrunner discover'
					junit '*.xml'
				}
			}
		} catch(e) {
			build_ok = false
			echo e.toString()  
		}
		
		stage('Run Examples') {		    
			echo "Running examples"
			//sh 'yum -y remove mpich'
			//sh 'yum -y install mpich-3.2'
			//sh 'export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:/usr/lib64/mpich/lib'
            sh 'pip3 install snakemake'
			dir('examples') {
				sh 'snakemake --cores=4 --config python_version=python3'
			}
			}
		if(build_ok) {
			currentBuild.result = "SUCCESS"
		} else {
			currentBuild.result = "FAILURE"
		}
	}
 }
}
