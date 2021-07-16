node {
    wrap([$class: "BuildUser"]) {
        checkout scm
        def buildlib = load("pipeline-scripts/buildlib.groovy")
        def commonlib = buildlib.commonlib

        commonlib.describeJob("rebuild", """
            <h2>Rebuild an image, rpm, or RHCOS for an assembly.</h2>
            <h3>This job is used to patch one or more images with an updated RPM dependency or upstream code commit</h3>
        """)

        properties(
            [
                disableResume(),
                buildDiscarder(
                    logRotator(
                        artifactDaysToKeepStr: "",
                        artifactNumToKeepStr: "",
                        daysToKeepStr: "",
                        numToKeepStr: "")),
                [
                    $class: "ParametersDefinitionProperty",
                    parameterDefinitions: [
                        commonlib.ocpVersionParam('BUILD_VERSION', '4'),
                        string(
                            name: "ASSEMBLY",
                            description: "The name of an assembly to rebase & build for. e.g. 4.9.1",
                            trim: true
                        ),
                        choice(
                            name: 'TYPE',
                            description: 'image or rpm or rhcos',
                            choices: [
                                    'image',
                                    'rpm',
                                    'rhcos',
                                ].join('\n'),
                        ),
                        string(
                            name: "DISTGIT_KEY",
                            description: "(Optional) The name of a component to rebase & build for. e.g. openshift-enterprise-cli; leave this empty when rebuilding rhcos",
                            defaultValue: "",
                            trim: true
                        ),
                        booleanParam(
                            name: "DRY_RUN",
                            description: "Take no action, just echo what the job would have done.",
                            defaultValue: false
                        ),
                        commonlib.mockParam(),
                    ]
                ],
            ]
        )   // Please update README.md if modifying parameter names or semantics

        commonlib.checkMock()
        stage("initialize") {
            buildlib.initialize()
            buildlib.registry_quay_dev_login()
            if (params.DRY_RUN) {
                currentBuild.displayName += " - [DRY RUN]"
            }
            currentBuild.displayName += " - $params.ASSEMBLY - $params.TYPE - ${params.DISTGIT_KEY?: '(N/A)'}"
            commonlib.shell(script: "pip install -e ./pyartcd")
        }
        stage ("Notify release channel") {
            if (params.DRY_RUN) {
                return
            }
            slackChannel = slacklib.to(params.BUILD_VERSION)
            slackChannel.say(":construction: Rebuilding $params.TYPE $params.DISTGIT_KEY for assembly $params.ASSEMBLY :construction:")
        }

        stage("rebuild") {
            sh "mkdir -p ./artcd_working"
            def cmd = [
                "artcd",
                "-vv",
                "--working-dir=./artcd_working",
                "--config", "./config/artcd.toml",
            ]

            if (params.DRY_RUN) {
                cmd << "--dry-run"
            }
            cmd += [
                "rebuild",
                "-g", "openshift-$params.BUILD_VERSION",
                "--assembly", params.ASSEMBLY,
                "--type", params.TYPE
            ]
            if (params.DISTGIT_KEY) {
                cmd << "--component"
                cmd << params.DISTGIT_KEY
            }
            sshagent(["openshift-bot"]) {
                echo "Will run ${cmd}"
                lock("github-activity-lock-${params.BUILD_VERSION}") {
                    commonlib.shell(script: cmd.join(' '))
                }
            }
        }
        stage("save artifacts") {
            commonlib.safeArchiveArtifacts([
                "artcd_working/email/**",
                "artcd_working/**/*.json",
                "artcd_working/**/*.log",
            ])
        }
        stage ("Notify release channel") {
            if (params.DRY_RUN) {
                return
            }
            slackChannel = slacklib.to(params.BUILD_VERSION)
            slackChannel.say("Hi @release-artists , rebuilding $params.TYPE $params.DISTGIT_KEY for assembly $params.ASSEMBLY is done. Please check instructions in the build log for the next step.")
        }
        buildlib.cleanWorkspace()
    }
}
