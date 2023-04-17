void setBuildStatus(String message, String state, String repo_url) {
  step([
      $class: "GitHubCommitStatusSetter",
      reposSource: [$class: "ManuallyEnteredRepositorySource", url: repo_url],
      contextSource: [$class: "ManuallyEnteredCommitContextSource", context: "ci/jenkins/build-status"],
      errorHandlers: [[$class: "ChangingBuildStatusErrorHandler", result: "UNSTABLE"]],
      statusResultSource: [ $class: "ConditionalStatusResultSource", results: [[$class: "AnyBuildResult", message: message, state: state]] ]
  ]);
}

pipeline {
    agent any

    environment {
        AWS_ROLE_ARN = 'arn:aws:iam::190655619732:role/CFAssumeFromFortinet'
        TEMPLATE = '$(pwd)/pipeline/...'
        TEMPLATE_PARAMS = '$(pwd)/pipeline/...'
    }

    stages {
        stage('Run cfn-lint linter') {
            steps {
                echo "Running cfn-lint..."
                sh 'cfn-lint $(pwd)/pipeline/webhosting/cloudfront-s3-website.yaml'
                sh 'cfn-lint $(pwd)/pipeline/webhosting/pipeline.yaml'

            }
        }
        stage('Run cfn-nag linter'){
            when { expression { false } }
            steps {
                sh 'docker pull stelligent/cfn_nag:latest'
                sh 'docker run -v $(pwd)/pipeline/webhosting:/templates -t stelligent/cfn_nag /templates/cloudfront-s3-website.yaml'
                sh 'docker run -v $(pwd)/pipeline/webhosting:/templates -t stelligent/cfn_nag /templates/pipeline.yaml'
            }
        } 
        stage('Running FortiDevSec scans...') {
            when { expression { false } }
            steps {
                echo "Running SAST scan..."
                sh 'env | grep -E "JENKINS_HOME|BUILD_ID|GIT_BRANCH|GIT_COMMIT" > /tmp/env'
                sh 'docker pull registry.fortidevsec.forticloud.com/fdevsec_sast:latest'
                sh 'docker run --rm --env-file /tmp/env --mount type=bind,source=$PWD,target=/scan registry.fortidevsec.forticloud.com/fdevsec_sast:latest'
            }
        }
        stage('Cloudformation Template Test Deployment'){
            when { expression { false } }
            steps {
               sh '''
                echo "Getting AWS account credentials..."
                set +x
                EXTERNAL_ID=\$(aws ssm get-parameter --name /jenkins/external-id --region us-east-1 --with-decryption | jq -r '.Parameter | .Value ')
                ASSUME_ROLE_OUTPUT=\$(aws sts assume-role --role-arn ${AWS_ROLE_ARN} --role-session-name jenkins --external-id \$EXTERNAL_ID)
                ASSUME_ROLE_ENVIRONMENT=\$(echo \$ASSUME_ROLE_OUTPUT | jq -r \'.Credentials | .["AWS_ACCESS_KEY_ID"] = .AccessKeyId | .["AWS_SECRET_ACCESS_KEY"] = .SecretAccessKey | .["AWS_SESSION_TOKEN"] = .SessionToken | del(.AccessKeyId, .SecretAccessKey, .SessionToken, .Expiration) | to_entries[] | "export \\(.key)=\\(.value)"\')
                eval \$ASSUME_ROLE_ENVIRONMENT 
                ERROR_CREDS="\$?"
                set -x
                [[ "ERROR_CREDS" == "0" ]] || echo "Error setting AWS credentials..."
                echo "Building stack..."
                stack_id=\$(aws cloudformation create-stack --stack-name stack-build-test --template-body file://$TEMPLATE --parameters file://$TEMPLATE_PARAMS --capabilities CAPABILITY_NAMED_IAM | jq .[] -r)
                stack_name=\$(aws cloudformation describe-stacks --query "Stacks[?StackId==\'\$stack_id\'][StackName]" --output text)
                echo "waiting on stack build to complete..."
                aws cloudformation wait stack-create-complete --stack-name \$stack_name
               '''
            }
        }
    }
    post {
     success {
        setBuildStatus("Build succeeded", "SUCCESS", "${GIT_URL}");
     }
     failure {
        setBuildStatus("Build failed", "FAILURE", "${GIT_URL}");
     }
  }
}
