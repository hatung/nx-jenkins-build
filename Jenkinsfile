
podTemplate(inheritFrom: 'default', containers: [
    containerTemplate(name: 'docker-dind', image: 'docker:dind', privileged:true, ttyEnabled: true),
    containerTemplate(name: 'nodejs', image: 'node:latest', ttyEnabled: true)
  ],
  //volumes: [hostPathVolume(hostPath: '/var/run/docker.sock', mountPath: '/var/run/docker.sock')]
  ) {


    node(POD_LABEL) {
        stage('Preparation') {
             container('nodejs') {
                stage('Install dependencies') {
                    sh 'echo hello from node'
                    checkout scm
                    sh 'npm config set fetch-timeout=600000'
                    sh 'npm install'
                }
            }

        }
        container('docker-dind') {

            def distributedTasks = [:]

            stage("Building Distributed Tasks") {
            jsTask {
                //checkout scm
                //sh 'npm install --network-timeout 1000000000'

                distributedTasks << distributed('test', 3)
                distributedTasks << distributed('lint', 3)
                distributedTasks << distributed('build', 3)
            }
            }

            stage("Run Distributed Tasks") {
            parallel distributedTasks
            }
        }

    }
}

def jsTask(Closure cl) {
    withEnv(["HOME=${workspace}"]) {
      docker.image('node:latest').inside('--tmpfs /.config', cl)
    }
}

def distributed(String target, int bins) {
  def jobs = splitJobs(target, bins)
  def tasks = [:]

  jobs.eachWithIndex { jobRun, i ->
    def list = jobRun.join(',')
    def title = "${target} - ${i}"

    tasks[title] = {
      jsTask {
        stage(title) {
          sh "npx nx run-many --target=${target} --projects=${list} --parallel"
        }
      }
    }
  }

  return tasks
}

def splitJobs(String target, int bins) {
  def String baseSha = env.CHANGE_ID ? 'origin/master' : 'origin/master~1'
  def String raw
  raw = sh(script: "npx nx print-affected --base=${baseSha} --target=${target}", returnStdout: true)
  def data = readJSON(text: raw)

  def tasks = data['tasks'].collect { it['target']['project'] }

  if (tasks.size() == 0) {
    return tasks
  }

  // this has to happen because Math.ceil is not allowed by jenkins sandbox (╯°□°）╯︵ ┻━┻
  def c = sh(script: "echo \$(( ${tasks.size()} / ${bins} ))", returnStdout: true).toInteger()
  def split = tasks.collate(c)

  return split
}
