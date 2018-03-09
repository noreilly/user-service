#!/usr/bin/groovy

@Library('github.com/noreilly/jenkins-pipeline@master')

def pipeline = new io.estrado.Pipeline()

podTemplate(label: 'jenkins-pipeline', containers: [
        containerTemplate(name: 'jnlp', image: 'jenkins/jnlp-slave:3.10-1-alpine', args: '${computer.jnlpmac} ${computer.name}', workingDir: '/home/jenkins', resourceRequestCpu: '200m', resourceLimitCpu: '300m', resourceRequestMemory: '256Mi', resourceLimitMemory: '512Mi', ttyEnabled: true),
        containerTemplate(name: 'mvn', image: 'maven:3.3.3', command: 'cat', ttyEnabled: true),
        containerTemplate(name: 'docker', image: 'docker:1.12.6', command: 'cat', ttyEnabled: true),
        containerTemplate(name: 'helm', image: 'lachlanevenson/k8s-helm:v2.6.0', command: 'cat', ttyEnabled: true)
],
        volumes: [
                hostPathVolume(mountPath: '/var/run/docker.sock', hostPath: '/var/run/docker.sock'),
        ]) {

    node('jenkins-pipeline') {

        checkout scm

        def inputFile = readFile('Jenkinsfile.json')
        def config = new groovy.json.JsonSlurperClassic().parseText(inputFile)
        def pwd = pwd()
        def chart_dir = "${pwd}/charts/user-service"
        println "directory: ${chart_dir} "
        sh "ls"

        def image_tags_map = pipeline.getContainerTags(config)

        // compile tag list
        def image_tags_list = pipeline.getMapValues(image_tags_map)

        stage('Compile and Verify') {

            container('mvn') {
                sh "mvn clean compile"
            }

        }

        stage('Security Scan') {
            container('mvn') {
                sh "echo 'mvn org.owasp:dependency-check-maven:aggregate'"

            }
        }

        stage('Test') {
            container('mvn') {
                sh "mvn clean install"

            }
        }

        stage('build image') {
            container('docker') {
                sh "docker build -t user-service:${env.BUILD_NUMBER} ."

            }
        }

        stage('test deployment') {
            container('helm') {

                // run helm chart linter
                pipeline.helmLint(chart_dir)

                // run dry-run helm chart installation
                pipeline.helmDeploy(
                        dry_run: true,
                        name: config.app.name,
                        namespace: config.app.name,
                        chart_dir: chart_dir,
                        version_tag: env.BUILD_NUMBER,
                        set: [
                                "imageTag"       : env.BUILD_NUMBER,
                                "replicas"        : config.app.replicas,
                                "cpu"             : config.app.cpu,
                                "memory"          : config.app.memory,
                                "ingress.hostname": config.app.hostname,
                        ]
                )

            }
        }

        // deploy only the master branch
        if (env.BRANCH_NAME == 'master') {
            stage('deploy to k8s') {
                container('helm') {
                    // Deploy using Helm chart
                    pipeline.helmDeploy(
                            dry_run: false,
                            name: config.app.name,
                            namespace: config.app.name,
                            chart_dir: chart_dir,
                            version_tag: env.BUILD_NUMBER,
                            set: [
                                    "imageTag"       : env.BUILD_NUMBER,
                                    "replicas"        : config.app.replicas,
                                    "cpu"             : config.app.cpu,
                                    "memory"          : config.app.memory,
                                    "ingress.hostname": config.app.hostname,
                            ]
                    )

                    //  Run helm tests
                    if (config.app.test) {
                        pipeline.helmTest(
                                name: config.app.name
                        )
                    }
                }
            }
        }
    }
}
