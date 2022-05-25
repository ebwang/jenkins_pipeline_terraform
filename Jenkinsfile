/*Declarado variavel TARGET vazia pois caso nao escolhido destroyTarget a variavel nao era instanciada*/
def TARGET = ' '

pipeline {

  agent {
    kubernetes {
      label "worker-tf-${UUID.randomUUID().toString()}"
      yamlFile './pipeline/JenkinsContainers.yml'
    }
  }

  /*
  Variable JOB_BASE_NAME is name of JOB in Jenkins, and this name is the same in the structure of directory in repository below:
  */

  stages {
    stage('TF plan'){
      when {
        anyOf {
          environment name: 'ACTION', value: 'create'
          environment name: 'ACTION', value: 'update'
        }
      }
      steps {
        plan(env.ENVIRONMENT, env.REGION, env.ACTION)
      }
    }

    stage('TF plan Destroy all') {
      when {
        environment name: 'ACTION', value: 'destroy'
      }
      steps {
        planDestroy(env.ENVIRONMENT, env.REGION, env.ACTION)
      }
    }

    stage('TF plan Destroy Target') {
      when {
        environment name: 'ACTION', value: 'destroyTarget'
      }
      steps {
        show(env.ENVIRONMENT, env.REGION)
        timeout(time: 10, unit: "MINUTES") {
          script {
            TARGET = input message: 'Digite o resource a ser destruido: ', ok: 'confirma', parameters: [string(defaultValue: 'nulo', description: 'ResourceName', name: 'RSName')]
          }
        }
        planDestroy(env.ENVIRONMENT, env.REGION, env.ACTION, "${TARGET}")
      }
    }

    stage('Approval') {
      steps {
        timeout(time: 10, unit: "MINUTES") {
          script {
            def userInput = input(id: 'confirm', message: 'Apply Terraform?', parameters: [ [$class: 'BooleanParameterDefinition', defaultValue: false, description: 'Apply terraform', name: 'confirm'] ])
          }
        }
      }
    }

    stage('TF Apply') {
      when {
        anyOf {
          environment name: 'ACTION', value: 'create'
          environment name: 'ACTION', value: 'update'
        }
      }
      steps {
        apply(env.ENVIRONMENT)
      }
    }

    stage('TF Destroy') {
      when {
        anyOf {
          environment name: 'ACTION', value: 'destroy'
          environment name: 'ACTION', value: 'destroyTarget'
        }
      }
      steps {
        destroy(env.ENVIRONMENT, env.REGION, "${TARGET}")
      }
    }
  }
}


def install_dependencies() {
  sh """
    apk add curl bash groff py-pip jq
    curl --silent --location 'https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_Linux_amd64.tar.gz' | tar x -z -C /tmp
    mv -v /tmp/eksctl /usr/local/bin
    pip install awscli
  """
}

def plan(String environment, String region, String action) {
  script {
    currentBuild.displayName += "  - ${environment} - ${region} - ${action}}"
    currentBuild.description = "[${environment}] ${region} - ${action}}"
  }
  withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'terraform_' + environment]]) {
    container('terraform') {
      install_dependencies()
      sh 'cd $JOB_BASE_NAME; terraform init -no-color -backend-config="region=' + region + '" -backend-config=env/' + environment + '.backend'
      sh 'cd $JOB_BASE_NAME; terraform plan -no-color -input=false  -var "aws_region=' + region + '" -var-file=env/' + environment + '.tfvars -out $JOB_BASE_NAME-plan'
    }
  }
}

def planDestroy(String environment, String region, String action, String target=null) {
  def targetM = (target == null) ? '' : '-target='+ target
  script{
    currentBuild.displayName += "  - ${environment} - ${region} - ${action}}"
    currentBuild.description = "[${environment}] ${region} - ${action}}"
  }
  withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'terraform_' + environment]]) {
    container('terraform') {
      install_dependencies()
      sh 'cd $JOB_BASE_NAME; terraform init -no-color -backend-config="region=' + region + '" -backend-config=env/' + environment + '.backend'
      sh 'cd $JOB_BASE_NAME; terraform plan -destroy ' + targetM +' -no-color -input=false  -var "aws_region=' + region + '" -var-file=env/' + environment + '.tfvars -out $JOB_BASE_NAME-plan'
    }
  }
}

def apply(String environment) {
  withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'terraform_' + environment]]) {
    container('terraform') {
      install_dependencies()
      sh 'cd $JOB_BASE_NAME;terraform apply -no-color -input=false $JOB_BASE_NAME-plan'
    }
  }
}

def destroy(String environment, String region, String target=' ') {
  def targetM = (target == ' ') ? '' : '-target='+ target
  withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'terraform_' + environment]]) {
    container('terraform') {
      install_dependencies()
      sh 'cd $JOB_BASE_NAME;terraform destroy ' + targetM + ' -no-color -input=false -var "aws_region=' + region + '" -var-file=env/' + environment + '.tfvars -auto-approve'
    }
  }
}

def show(String environment, String region) {
  withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'terraform_' + environment]]) {
    container('terraform') {
      install_dependencies()
      sh 'cd $JOB_BASE_NAME; terraform init -no-color -backend-config="region=' + region + '" -backend-config=env/' + environment + '.backend'
      sh 'cd $JOB_BASE_NAME; terraform show -no-color'
    }
  }
}
