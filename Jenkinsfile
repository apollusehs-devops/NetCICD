// This Source Code Form is subject to the terms of the Mozilla Public
// License, v. 2.0. If a copy of the MPL was not distributed with this
// file, You can obtain one at http://mozilla.org/MPL/2.0/.

pipeline {
    agent none
    
    environment {
        JENKINS_CRED = credentials('Jenkins-SIM')
        CML_CRED = credentials('CML-SIM-CRED')
        CML_URL = credentials('CML-URL')
    }
    stages {
        stage('Prepare Jenkins...') {
            agent { 
                node { 
                    label 'swarm_node' 
                } 
            }
            steps {
                echo 'Collecting variables'
                echo '--------------------'
                echo "${CML_URL}"
                script { 
                    gitCommit = sh(returnStdout: true, script: 'git rev-parse --short  HEAD').trim()
                    jc = sh(returnStdout: true, script: 'curl -u ' + "${JENKINS_CRED}" + ' \'' + "${env.JENKINS_URL}" + 'crumbIssuer/api/xml?xpath=concat(//crumbRequestField,":",//crumb)\'').trim()
                    JenkinsCrumb = jc.substring(14)
                }
            }
        }
        stage('Deploying Stage 0 simulation (Box) in CML') {
            agent { 
                node { 
                    label 'swarm_node' 
                } 
            }
            steps {
                echo "Prepare stage 0 simulation environment"
                echo "--------------------------------------"

                checkout scm

                startsim(0)
            
            }
        }
        stage('Stage 0: Deploying and testing Box in CML') {
            steps {
                node ("stage0-" + gitCommit as String) {
                    echo "Switching to jenkins agent: stage0-" + "${gitCommit}"

                    checkout scm

                    echo "Start stage 0 playbook"
                    ansiblePlaybook installation: 'ansible', inventory: 'vars/stage0', playbook: 'stage0.yml', extraVars: ["stage": "0"], extras: '-vvvv'
                }
                node ('swarm_node') {
                    echo "Switching to jenkins agent: swarm_node"

                    echo 'Stopping CML simulation'
                    sh 'curl -X GET -u ' + "${CML_CRED}" + ' ' + "${CML_URL}"  + '/simengine/rest/stop/stage0-' + "${gitCommit}"

                    echo 'Removing Jenkins Agent'
                    sh 'curl -L -s -o /dev/null -u ' + "${JENKINS_CRED}" + ' -H "Content-Type:application/x-www-form-urlencoded" -H "' + "${jc}" + '" -X POST "' + "${env.JENKINS_URL}" + 'computer/stage0' + "-" + "${gitCommit}" + '/doDelete"'
                }
            }
        }
     }
    post {
        success {
            echo 'I succeeded!'
            //mail to: 'architects@example.com',
            //    subject: "Success for Pipeline: ${gitCommit}",
            //    body: "Success for ${env.BUILD_URL}"
        }
        failure {
            echo 'You are not a Jedi yet....I failed :('
            //    mail to: 'architects@example.com',
            //    subject: "Failed Pipeline: ${gitCommit}",
            //    body: "Something is wrong with ${env.BUILD_URL}"
        }
        changed {
            echo 'Things were different before...'
        }
        cleanup {
            echo 'Cleaning up....'
        }
    }
}


def startsim(stage) {
    echo 'Creating Jenkins build node for commit: ' + "${gitCommit}"
    sh 'curl -L -s -o /dev/null -u ' + "${JENKINS_CRED}" + ' -H Content-Type:application/x-www-form-urlencoded -H "'+ "${jc}" + '" -X POST -d \'json={"name":+"stage' + "${stage}" + "-" + "${gitCommit}" + '",+"nodeDescription":+"NetCICD+host+for+commit+is+stage'  + "${stage}" + "-"+ "${gitCommit}" + '",+"numExecutors":+"1",+"remoteFS":+"/root",+"labelString":+"slave' + "${stage}" + "-"+ "${gitCommit}" + '",+"mode":+"EXCLUSIVE",+"":+["hudson.slaves.JNLPLauncher",+"hudson.slaves.RetentionStrategy$Always"],+"launcher":+{"stapler-class":+"hudson.slaves.JNLPLauncher",+"$class":+"hudson.slaves.JNLPLauncher",+"workDirSettings":+{"disabled":+false,+"workDirPath":+"",+"internalDir":+"remoting",+"failIfWorkDirIsMissing":+false},+"tunnel":+"",+"vmargs":+""},+"retentionStrategy":+{"stapler-class":+"hudson.slaves.RetentionStrategy$Always",+"$class":+"hudson.slaves.RetentionStrategy$Always"},+"nodeProperties":+{"stapler-class-bag":+"true"},+"type":+"hudson.slaves.DumbSlave",+"Jenkins-Crumb":+"'+ "${JenkinsCrumb}" + '"}\' "' + "${env.JENKINS_URL}" + 'computer/doCreateItem?name="stage' + "${stage}" + "-" + "${gitCommit}" + '"&type=hudson.slaves.DumbSlave"'
  
    echo 'Retrieving Agent Secret'
    script {
        agentSecret = jenkins.model.Jenkins.getInstance().getComputer("stage" + "${stage}" + "-" + "$gitCommit").getJnlpMac()
    }
    echo "secret = " + "${agentSecret}"

    echo "Inserting jenkins url in docker jumphost configuration"
    sh "sed -i 's%jenkins_url%" + "${env.JENKINS_URL}" + "%g' virl/stage" + "${stage}" + ".virl"

    echo "Inserting agent secret in docker jumphost configuration"
    sh "sed -i 's/jenkins_secret/" + "${agentSecret}" + "/g' virl/stage" + "${stage}" + ".virl"
   
    echo "Configuring Jenkins agent to use"
    sh "sed -i 's/jenkins_agent/stage" + "${stage}" + "-" + "${gitCommit}" + "/g' virl/stage" + "${stage}" + ".virl"

    echo 'Starting CML simulation for stage ' + "${stage}"
    sh 'curl -X POST -u ' + "${CML_CRED}" + ' --header "Content-Type:text/xml;charset=UTF-8" --data-binary @virl/stage' + "${stage}" + '.virl ' + "${CML_URL}" + '/simengine/rest/launch?session=stage' + "${stage}" + '-' + "${gitCommit}"

    timeout(time: 30, unit: "MINUTES") {
        script {
            waitUntil {
                sleep 60
                cml_state_json = sh(returnStdout: true, script: 'curl -X GET -u ' + "${CML_CRED}" + ' ' + "${CML_URL}" + '/simengine/rest/nodes/stage' + "${stage}" + '-' + "${gitCommit}").trim()
                def c_state = readJSON text: "${cml_state_json}"
                cml_state = c_state["stage" + "${stage}" + "-"+"${gitCommit}"]
                //echo "${cml_state}"
                cs = cml_state.collect {it.value.reachable}
                echo "Node reachability: " + "${cs}"
                test =  cs.every {element -> element == true}
                echo "Simulation ready? " + "${test}"
                if (test) {
                    return true
                } else {
                    return false
                }
            }
        }
    }
    return null
}