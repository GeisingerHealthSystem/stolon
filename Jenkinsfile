#!groovy
// TODO - test systemd service
//	example: https://github.com/docker/compose/issues/4266
properties([
	parameters([
		string(
			defaultValue: "gdchdpmn04drlx.geisinger.edu",
			description: "Hostname to build on",
			name: 'hostname'
		),
		string(
			defaultValue: "icinga",
			description: 'Git target (such as a branch or tag)',
			name: 'git_target'
		),
		choice(
			choices: ['branch','tag'].join("\n"),
			description: 'What the target type is',
			name: 'git_target_type'
		),
		string(
			defaultValue: "",
			description: "Pass Docker build args",
			name: 'docker_build_args'
		),
		booleanParam(
			defaultValue: false,
			description: "Restart via docker compose only",
			name: "restart_only"
		)
	])
])
node(params.hostname) {
	currentBuild.result = "SUCCESS"
	env.CUSTOM_DOCKER_BUILD_ARGS = params.docker_build_args
	env.CREDENTIALS_STORE = 'uda-github'
	env.PROJECT_NAME = 'stolon-poc'
	env.YAML_FILE = 'docker-compose-etcd.yml'
	// Tag usage: https://issues.jenkins-ci.org/browse/JENKINS-27018
	// Set tag or branch based on request (master is not a valid tag)
	// This allows things to be dynamic enough to work out
	if (params.git_target_type == 'branch') {
		env.GIT_TARGET = '*/' + params.git_target
	}
	else if (params.git_target_type == 'tag') {
		env.GIT_TARGET = 'refs/tags/' + params.git_target
	}
	try {
		stage('Checkout') {
			dir('stolon-repo') {
				checkout([$class: 'GitSCM',
					branches: [[name: env.GIT_TARGET]],
					doGenerateSubmoduleConfigurations: false,
					extensions: [[$class: 'SubmoduleOption',
						disableSubmodules: false,
						parentCredentials: true,
						recursiveSubmodules: true,
						reference: '',
						trackingSubmodules: false]],
					submoduleCfg: [],
					userRemoteConfigs: [[credentialsId: 'udahadoopops',
										url: 'https://github.com/GeisingerHealthSystem/stolon']]])
			}
		}
		// Note: since docker runs as sudo, you MUST have these vars defined in
		// /etc/sudoers.d/cdis_sys_${ENV} to retain them when using sudo
		// See: <PENDING_CONFLUENCE_LINK>
		withCredentials(
			[usernamePassword(
				credentialsId: 'postgres_stolon_master_pw',
				 passwordVariable: 'POSTGRES_PASSWORD',
				 usernameVariable: 'POSTGRES_USER'),
			string(
				credentialsId: 'stolon-master-key',
				variable: 'stolon_MASTER_KEY')
			]) {
				dir('stolon-repo') {
					if(!params.restart_only){
						stage('Docker compose') {
							sh script: """
								sudo docker-compose -f ${YAML_FILE} -p ${PROJECT_NAME} down
								sudo docker-compose -f ${YAML_FILE} -p ${PROJECT_NAME} up --build -d --remove-orphans
							"""
						}
					} else if (params.restart_only){
						stage('Restarting application') {
							sh script: """
								sudo docker-compose -f ${YAML_FILE} -p ${PROJECT_NAME} restart
							"""
						}
					}
					stage('Check services') {
						sh script: """
							sudo docker-compose -f ${YAML_FILE} -p ${PROJECT_NAME} ps
						"""
					}
			}
		}
		stage('Cleanup') {
			cleanWs()
		}
	}
	catch (err) {
		currentBuild.result = "FAILURE"
		throw err
	}
	finally {
		String buildColor = 'YELLOW'
		if (currentBuild.result == "SUCCESS") {
						buildColor = 'GREEN'
		}
		if (currentBuild.result == "FAILURE") {
						buildColor = 'RED'
		}
		//stage('Notify') {
		///	hipchatSend message: '${PROJECT_NAME} - ${BUILD_STATUS} - ${BUILD_URL}', notify: true, color: buildColor
		//}
	}
}
