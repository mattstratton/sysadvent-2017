pipeline {
  agent any

  

  stages {
    stage("Lint") {
      steps {
        sh 'foodcritic'
        sh 'cookstyle'
      }
    }
    stage("Smoke") {
      steps {
        sh 'kitchen test'
      }
    }
    stage("Compliance") {
      steps {
        writeFile file: '.kitchen.jenkins.yml', text: '''---
driver:
  name: dokken

provisioner:
  name: dokken

verifier:
  name: inspec

transport:
  name: dokken

platforms:
  - name: centos-7
    driver:
      image: dokken/centos-7

suites:
  - name: default
    run_list:
      - recipe[$JOB_BASE_NAME::default]
    verifier:
      inspec_tests:
        - test/smoke/default
        - https://github.com/mattstratton/sa2017-compliance.git
    attributes:
'''
        sh 'KITCHEN_YAML=".kitchen.jenkins.yml" kitchen test'
      }
    }

  }


}




