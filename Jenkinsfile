// Author: Dang Thanh Phat
// Email: thanhphatit95@gmail.com
// Blog: itblognote.com
// Description: Code pipeline for Jenkins all service Pharmacity
// ============================START============================
properties([pipelineTriggers([githubPush()])])
node {
  try {
    def namespace = 'pharmacy'
    def imageName = 'pmc-devops-nginx-sample'
    def releaseName = 'pmc-devops-nginx-sample'
    def chartName = 'general-application'
    def repository = 'https://github.com/Pharmacity-JSC/pmc-devops-nginx-sample'
    def buildDockerENV = 'false'
    def environment
    def myRepo
    def gitBranchName
    def shortGitCommit 
    def dockerImageTag
    def dockerImageTagLatest
    def eksClusterDefault
    def awsAccessKeyID 
    def awsSecretKeyID
    def awsAccount
    def dockerImage
    def userBuild
    def mailUserBuild
    // notifyBuild('STARTED')
    stage('Checkout') {
      echo "before checkout"
      myRepo = checkout scm
      echo "after chekout scm"
      gitBranchName = myRepo.GIT_BRANCH
            echo "$gitBranchName"
      // gitBranchName2 = scm.branches[0].name.split("/")[1]
      gitBranchName = gitBranchName.substring(gitBranchName.lastIndexOf('/')+1, gitBranchName.length())
      echo "$gitBranchName"
      shortGitCommit = "${myRepo.GIT_COMMIT[0..10]}"
      echo "shortGitCommit $shortGitCommit"
      if(gitBranchName == 'master' || gitBranchName == 'main' || gitBranchName == 'production')
      {
        echo "Working on branch ${gitBranchName}...!"
        environment = 'production'
        eksClusterDefault = 'prod-eks-main'
        awsAccount = 'aws_account_prod'
        dockerImageTag = "${environment}.${shortGitCommit}"
        dockerImageTagLatest = "${environment}.latest"
        withCredentials([aws(credentialsId: 'aws_account_prod', accessKeyVariable: 'aws_access_key_id', secretKeyVariable: 'aws_secret_access_key')]) {
          awsAccessKeyID = "${aws_access_key_id}"
          awsSecretKeyID = "${aws_secret_access_key}"
        }
      }  
      else if(gitBranchName == 'staging' || gitBranchName == 'stag' || gitBranchName == 'stg')
      {
        echo "Working on branch ${gitBranchName}...!"
        environment = 'staging'
        eksClusterDefault = 'stg-eks-main'
        awsAccount = 'aws_account_stag'
        dockerImageTag = "${environment}.${shortGitCommit}"
        dockerImageTagLatest = "${environment}.latest"
        withCredentials([aws(credentialsId: 'aws_account_stag', accessKeyVariable: 'aws_access_key_id', secretKeyVariable: 'aws_secret_access_key')]) {
          awsAccessKeyID = "${aws_access_key_id}"
          awsSecretKeyID = "${aws_secret_access_key}"
        }
      }
      else if(gitBranchName == 'training' || gitBranchName == 'train')
      {
        echo "Working on branch ${gitBranchName}...!"
        environment = 'training'
        namespace = 'pmc-training'
        eksClusterDefault = 'stg-eks-main'
        awsAccount = 'aws_account_stag'
        dockerImageTag = "${environment}.${shortGitCommit}"
        dockerImageTagLatest = "${environment}.latest"
        withCredentials([aws(credentialsId: 'aws_account_stag', accessKeyVariable: 'aws_access_key_id', secretKeyVariable: 'aws_secret_access_key')]) {
          awsAccessKeyID = "${aws_access_key_id}"
          awsSecretKeyID = "${aws_secret_access_key}"
        }
      }
      else
      {
        echo "Working on branch ${gitBranchName}...!"
        environment = 'development'
        namespace = 'pmc-testing'
        eksClusterDefault = 'stg-eks-main'
        awsAccount = 'aws_account_stag'
        dockerImageTag = "${environment}.${shortGitCommit}"
        dockerImageTagLatest = "${environment}.latest"
        withCredentials([aws(credentialsId: 'aws_account_stag', accessKeyVariable: 'aws_access_key_id', secretKeyVariable: 'aws_secret_access_key')]) {
          awsAccessKeyID = "${aws_access_key_id}"
          awsSecretKeyID = "${aws_secret_access_key}"
        }
      }
    }

    stage("Slack") {
      userBuild = sh(script: 'set +x; echo $(git show -s --pretty=%an)', returnStdout: true).trim()
      mailUserBuild = sh(script: 'set +x; echo $(git show -s --pretty=%ae)', returnStdout: true).trim()
      slackSend(
        color: "good", 
        message: "Project: ${repository} is building...!\nOn Branch: ${gitBranchName}\nBy User: ${userBuild}\nMail: ${mailUserBuild}"
      )
    }

    stage('Build docker') {
      withCredentials([string(credentialsId: 'aws_ecr_account_url', variable: 'ecr_url'), string(credentialsId: 'aws_default_region', variable: 'aws_region')]) {
        withEnv(["AWS_DEFAULT_REGION=${aws_region}",
          "AWS_ACCESS_KEY_ID=${awsAccessKeyID}",
          "AWS_SECRET_ACCESS_KEY=${awsSecretKeyID}"]){
          sh "set +x; aws ecr describe-repositories --repository-names ${imageName} 2>&1 > /dev/null || aws ecr create-repository --repository-name ${imageName} --image-scanning-configuration scanOnPush=true"
          docker.withRegistry("https://${ecr_url}", "ecr:${aws_region}:${awsAccount}") {
            if(buildDockerENV == 'false')
            {
              dockerImage = docker.build("${imageName}:${dockerImageTag}","-t ${imageName}:${dockerImageTagLatest} .")
            }
            else
            {
              dockerImage = docker.build("${imageName}:${dockerImageTag}","--build-arg ENV_DEPLOY=${environment} -t ${imageName}:${dockerImageTagLatest} .")
            }
            dockerImage.push()
            dockerImage.push("${dockerImageTagLatest}")
          }
          echo "[SUCCESS] Push image ${imageName}:${dockerImageTag} and ${imageName}:${dockerImageTagLatest} to ECR"  
        }
      }
    }

    stage("Deploy helm ${environment}") {
      withCredentials([string(credentialsId: 'aws_default_region', variable: 'aws_region'), 
        string(credentialsId: 'aws_s3_bucket_name_helm_chart', variable: 's3_bucket_name_helm_chart'),
        string(credentialsId: 'aws_ecr_account_url', variable: 'aws_ecr_url')]) {
        withEnv(["AWS_DEFAULT_REGION=${aws_region}", 
          "AWS_ACCESS_KEY_ID=${awsAccessKeyID}", 
          "AWS_SECRET_ACCESS_KEY=${awsSecretKeyID}", 
          "EKS_CLUSTER=${eksClusterDefault}",
          "S3_BUCKET_NAME=${s3_bucket_name_helm_chart}",
          'PRIVATE_HELM_REPO_NAME=pharmacity-charts',
          "HELM_NAMESPACE_NAME=${namespace}",
          "HELM_RELEASE_NAME=${releaseName}",
          "HELM_CHART_NAME=${chartName}", 
          "IMAGE_NAME=${imageName}", 
          "IMAGE_TAG_BUILD=${dockerImageTag}", 
          "AWS_ECR_ACCOUNT_URL=${aws_ecr_url}"]) {
          println("Kubernetes: Config account ${eksClusterDefault}")  
          sh '''#!/bin/bash
              create_config_k8s(){
                AWS_ACCOUNT_ID=$(aws sts get-caller-identity --output text  | awk '{print $1}' | tr -d ' ')
                EKS_CLUSTER_ASSUME_ROLE_ARN="arn:aws:iam::${AWS_ACCOUNT_ID}:role/role-eks-deployment-${EKS_CLUSTER}"
                # Show information
                echo "[+] EKS_CLUSTER=$EKS_CLUSTER"
                echo "[+] EKS_CLUSTER_ASSUME_ROLE_ARN=$EKS_CLUSTER_ASSUME_ROLE_ARN"

                # Authen AWS and generate kube-config for EKS_CLUSTER
                aws eks update-kubeconfig \
                  --name ${EKS_CLUSTER} \
                  --region ${AWS_DEFAULT_REGION} \
                  --role-arn ${EKS_CLUSTER_ASSUME_ROLE_ARN}

                echo ${EKS_CLUSTER} - ${AWS_DEFAULT_REGION} - ${EKS_CLUSTER_ASSUME_ROLE_ARN}
                # Show current kubeconfig context
                echo "[+] Kubeconfig current-context:"
                kubectl config current-context
              }

              create_config_k8s

              until $(kubectl cluster-info &>/dev/null)
              do
                [ -d "$HOME/.kube" ] && rm -rf "$HOME/.kube" || echo "Folder is not exists !"
                create_config_k8s
              done
              echo ""
              echo "Kubernetes control plane is running !"
          '''
          println("Helm: Add Helm Repository")  
          sh '''#!/bin/bash
              echo "[+] Helm client version"
              helm version

              echo "[+] Check helm-s3 exist"
              helm plugin list

              echo "[+] Helm add repository of company"
              echo "PRIVATE_HELM_REPO_NAME: ${PRIVATE_HELM_REPO_NAME}"
              echo "S3_BUCKET_NAME: ${S3_BUCKET_NAME}"

              if [[ "$(helm repo list 2> /dev/null | grep -i "${PRIVATE_HELM_REPO_NAME}")" ]];then
                  # Remove current setting Helm Repo to add new
                  helm repo remove ${PRIVATE_HELM_REPO_NAME} 2> /dev/null
              fi

              helm repo add ${PRIVATE_HELM_REPO_NAME} ${S3_BUCKET_NAME}
              helm repo update
              helm repo list
              
              echo "[+] List active Charts in Helm Chart Repository: ${PRIVATE_HELM_REPO_NAME}"
              helm search repo ${PRIVATE_HELM_REPO_NAME}
          '''
          println("Helm: Deploy Application")
          sh '''#!/bin/bash
              echo -e "\n[+] Start deployment with helm"
              echo "HELM_NAMESPACE_NAME: ${HELM_NAMESPACE_NAME}"
              echo "HELM_RELEASE_NAME: ${HELM_RELEASE_NAME}"
              echo "HELM_CHART_NAME: ${PRIVATE_HELM_REPO_NAME}/${HELM_CHART_NAME}"
              echo "IMAGE_URL: ${AWS_ECR_ACCOUNT_URL}/${IMAGE_NAME}"
              echo "IMAGE_TAG_BUILD: ${IMAGE_TAG_BUILD}"

              # We upgrade helm only, no install helm release from app-repo
              # We define all settings for helm release application in other repository
              CURRENT_UNIXTIME=$(date +%s)
              
              upgrade_helm(){
                helm upgrade ${HELM_RELEASE_NAME} ${PRIVATE_HELM_REPO_NAME}/${HELM_CHART_NAME} \
                  --reuse-values \
                  --namespace ${HELM_NAMESPACE_NAME} \
                  --set image.repository="${AWS_ECR_ACCOUNT_URL}/${IMAGE_NAME}" \
                  --set image.tag="${IMAGE_TAG_BUILD}" \
                  --set timestamp="${CURRENT_UNIXTIME}"
              }

              check_helm(){
                helmReleaseName=$(helm list -n ${HELM_NAMESPACE_NAME} | awk '{print $1}' | grep -i ${HELM_RELEASE_NAME} | tr -d ' ' | head -n1)
                if [[ "${helmReleaseName}" == "${HELM_RELEASE_NAME}" ]];then
                  upgrade_helm
                else
                  echo "[WARNING] Sorry, The repo doesn't exist in list !"
                fi 
              }    

              check_helm
              until $(kubectl cluster-info &>/dev/null)
              do
                check_helm
              done          
          '''
        }
      }
      sh 'set +x; [ -d "$HOME/.kube" ] && rm -rf "$HOME/.kube" || echo "Folder is not exists !"'
      println("[SUCCESS] Deploy to ${environment} !") 
    } 
  } catch (e) {
    currentBuild.result = "FAILED"
    // Only failure send notifications
    notifyBuild(currentBuild.result)
    throw e
  } finally {
    // Success or failure, always send notifications
    println("Build status: ${currentBuild.result}") 
    //notifyBuild(currentBuild.result)
  }
}

def notifyBuild(String buildStatus = 'STARTED') {
  // build status of null means successful
  buildStatus = buildStatus ?: 'SUCCESS'

  def mailSubject = "[${buildStatus}]: Jenkins Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]'"
  def mailTo  = 'phat.dangthanh@pharmacity.vn'
  def mailDetails = """
      Jobs you building is failed. Please, check it !
[+] Project: ${env.JOB_NAME} [${env.BUILD_NUMBER}]
[+] URL: ${env.BUILD_URL}
  """

  emailext (
    subject: mailSubject,
    to: mailTo,
    body: mailDetails
  ) 
}
