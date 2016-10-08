
# Openshift 3.3 instructions

## add new CICD jenkins 
```
oc new-project cicd --display-name='CICD Jenkins' --description='CICD Jenkins'
oc new-app --template=jenkins-persistent -p JENKINS_IMAGE_STREAM_TAG=jenkins-1-rhel7:latest,NAMESPACE=openshift,MEMORY_LIMIT=2048Mi,JENKINS_PASSWORD=password
```

## pipline raw groovy for reference
```
node('maven') {
  stage 'build & deploy in dev'
  openshiftBuild(namespace: 'development',
  			    buildConfig: 'myapp',
			    showBuildLogs: 'true',
			    waitTime: '3000000')
  stage 'verify deploy in dev'
  openshiftVerifyDeployment(namespace: 'development',
				       depCfg: 'myapp',
				       replicaCount:'1',
				       verifyReplicaCount: 'true',
				       waitTime: '300000')
  stage 'deploy in test'
  openshiftTag(namespace: 'development',
  			  sourceStream: 'myapp',
			  sourceTag: 'latest',
			  destinationStream: 'myapp',
			  destinationTag: 'promoteQA')

  openshiftDeploy(namespace: 'testing',
  			     deploymentConfig: 'myapp',
			     waitTime: '300000')

  openshiftScale(namespace: 'testing',
  			     deploymentConfig: 'myapp',
			     waitTime: '300000',
			     replicaCount: '2')

  stage 'verify deploy in test'
  openshiftVerifyDeployment(namespace: 'testing',
				       depCfg: 'myapp',
				       replicaCount:'2',
				       verifyReplicaCount: 'true',
				       waitTime: '300000')

}
```

## create pipeline build config
```
cat <<EOF | oc create -f -
apiVersion: v1
kind: BuildConfig
metadata:
  creationTimestamp: null
  labels:
    app: pipeline
    name: pipeline
  name: pipeline
spec:
  output: {}
  postCommit: {}
  resources: {}
  runPolicy: Serial
  source:
    type: None
  strategy:
    jenkinsPipelineStrategy:
      jenkinsfile: "#!groovy\nnode('maven') {\n  stage 'build & deploy in dev'\n  openshiftBuild(namespace:
        'development',\n  \t\t\t    buildConfig: 'myapp',\n\t\t\t    showBuildLogs:
        'true',\n\t\t\t    waitTime: '3000000')\n  stage 'verify deploy in dev'\n
        \ openshiftVerifyDeployment(namespace: 'development',\n\t\t\t\t       depCfg:
        'myapp',\n\t\t\t\t       replicaCount:'1',\n\t\t\t\t       verifyReplicaCount:
        'true',\n\t\t\t\t       waitTime: '300000')\n  stage 'deploy in test'\n  openshiftTag(namespace:
        'development',\n  \t\t\t  sourceStream: 'myapp',\n\t\t\t  sourceTag: 'latest',\n\t\t\t
        \ destinationStream: 'myapp',\n\t\t\t  destinationTag: 'promoteQA')\n\n  openshiftDeploy(namespace:
        'testing',\n  \t\t\t     deploymentConfig: 'myapp',\n\t\t\t     waitTime:
        '300000')\n\n  openshiftScale(namespace: 'testing',\n  \t\t\t     deploymentConfig:
        'myapp',\n\t\t\t     waitTime: '300000',\n\t\t\t     replicaCount: '2')\n\n
        \ stage 'verify deploy in test'\n  openshiftVerifyDeployment(namespace: 'testing',\n\t\t\t\t
        \      depCfg: 'myapp',\n\t\t\t\t       replicaCount:'2',\n\t\t\t\t       verifyReplicaCount:
        'true',\n\t\t\t\t       waitTime: '300000')\n\n}\n"
    type: JenkinsPipeline
  triggers:
  - github:
      secret: secret101
    type: GitHub
  - generic:
      secret: secret101
    type: Generic
status:
  lastVersion: 0
EOF
```

## create projects and permissions
```
oc new-project development --display-name='MyApp Development' --description='MyApp Development'
oc policy add-role-to-user edit system:serviceaccount:cicd:jenkins -n development

oc new-project testing --display-name='MyApp Testing' --description='MyApp Testing'
oc policy add-role-to-user edit system:serviceaccount:cicd:jenkins -n testing
oc policy add-role-to-group system:image-puller system:serviceaccount:testing -n development
```

# create config for myapp from webui if you want to disbale first build/deploy
```
oc project development
oc new-app openshift/php:5.6~https://github.com/eformat/bgdemo --name myapp
oc expose svc myapp
```

#create DC in testing project testing
```
oc project testing
oc create dc myapp --image=172.30.148.12:5000/development/myapp:promoteQA
oc expose dc myapp --port=8080
oc expose svc myapp
```

#add pipeline webhook to git
Add into github project webhooks (point to pipline webhook)
