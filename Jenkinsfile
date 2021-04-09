def DOCKER_REGISTRY = "choisunguk"
def DEPLOYMNET_NAME = "reactjs-demo"
def DEPLOYMENT_NAMESPACE = "default"

pipeline {
    agent any

    stages {
        stage('build project'){
            steps{
                sh "npm install"
                sh "npm run build"
                // compress artifacts
                dir('build'){
                    sh "tar -cvf ../build.tar ./*"
                }
            }
        }
        stage("build docker image and push"){
            steps{
                script {
                    version = sh(script:""" node -pe "require('./package.json').version"  """, returnStdout:true).trim()
                    println version

                    // build dockerfile
                    docker_imagename = "${DOCKER_REGISTRY}/test-frontend:${version}"
                    def dockerimage = docker.build("${docker_imagename}")

                    // push docker image
                    docker.withRegistry('https://registry.hub.docker.com', 'dockerhub'){
                        dockerimage.push()
                    }

                    // rmi docker image
                    sh(script:"docker rmi ${docker_imagename}")
                }                
            }
        }
        stage('get project version'){
            steps{
                script{
                    image_name = sh(script:""" kubectl get po -A -o json | jq --raw-output '.items[].spec.containers[].image | select(. == "choisunguk/test-frontend:0.1.0")' | sort | uniq """, returnStdout:true )
                    if(!image_name){
                        current_version = ''
                    }else{
                        current_version = image_name.split(":")[1].trim()
                    }
                }    
            }
        }
        stage('change deployment image name'){
            steps{
                dir ('kubernetes_resource'){
                    script {
                        def sed_docker_imagename = docker_imagename.replace("/", "\\/")
                        sh(script:"""sed -i "s/IMAGE_NAME/${sed_docker_imagename}/g" deployment.yaml""")
                    }
                }
            }
        }
        stage('deploy restart same image version'){
            when{
                expression{
                    current_version == version
                }
            }
            steps{
                dir ('kubernetes_resource'){
                    script {
                        sh(script: """kubectl apply -f .""")
                        sh(script: """kubectl rollout restart deployment ${DEPLOYMNET_NAME} -n ${DEPLOYMENT_NAMESPACE}""")
                    }
                }
            }
        }
        stage('deploy on kubernetes'){
            when{
                expression {
                    current_version != version
                }
            }
            steps{
                dir ('kubernetes_resource'){
                    sh 'kubectl apply -f .' 
                }
            }
        }
    }
}