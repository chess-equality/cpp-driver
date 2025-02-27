pipeline {
  agent {
    kubernetes {
      label 'cpp-driver-bblfsh-performance'
      defaultContainer 'cpp-driver-bblfsh-performance'
      yaml """
spec:
  nodeSelector:
    srcd.host/type: jenkins-worker
  affinity:
    podAntiAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchExpressions:
          - key: jenkins
            operator: In
            values:
            - slave
        topologyKey: kubernetes.io/hostname
  containers:
  - name: cpp-driver-bblfsh-performance
    image: bblfsh/performance:latest
    imagePullPolicy: Always
    securityContext:
      privileged: true
    command:
    - dockerd
    tty: true
"""
    }
  }
  environment {
    DRIVER_LANGUAGE = "cpp"
    DRIVER_REPO = "https://github.com/bblfsh/cpp-driver.git"
    DRIVER_SRC_FIXTURES = "${env.WORKSPACE}/fixtures"
    BENCHMARK_FILE = "${env.WORKSPACE}/bench.log"
    LOG_LEVEL = "debug"
    PROM_ADDRESS = "http://prom-pushgateway-prometheus-pushgateway.monitoring.svc.cluster.local:9091"
    PROM_JOB = "bblfsh_perfomance"
  }
  // TODO(lwsanty): https://github.com/src-d/infrastructure/issues/992
  // this is polling for every 2 minutes
  // however it's better to use trigger curl http://yourserver/jenkins/git/notifyCommit?url=<URL of the Git repository>
  // https://kohsuke.org/2011/12/01/polling-must-die-triggering-jenkins-builds-from-a-git-hook/
  // the problem is that it requires Jenkins to be accessible from the hook side
  // probably Travis CI could trigger Jenkins after all unit tests have passed...
  triggers { pollSCM('H/2 * * * *') }
  stages {
    stage('Run transformations benchmark') {
      when { branch 'master' }
      steps {
        sh "set -o pipefail ; go test -run=NONE -bench=/transform ./driver/... | tee ${env.BENCHMARK_FILE}"
      }
    }
    stage('Store transformations benchmark to prometheus') {
      when { branch 'master' }
      steps {
        sh "/root/bblfsh-performance parse-and-store --language=${env.DRIVER_LANGUAGE} --commit=${env.GIT_COMMIT} --storage=prom ${env.BENCHMARK_FILE}"
      }
    }
    stage('Run end-to-end benchmark') {
      when { branch 'master' }
      steps {
        sh "/root/bblfsh-performance end-to-end --language=${env.DRIVER_LANGUAGE} --commit=${env.GIT_COMMIT} --docker-tag=latest --custom-driver=true --storage=prom ${env.DRIVER_SRC_FIXTURES}"
      }
    }
  }
}
