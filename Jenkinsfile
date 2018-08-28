#!groovy

def kubectlTest() {
    // Test that kubectl can correctly communication with the Kubernetes API
    echo "running kubectl test"
    sh "kubectl get nodes"

}

def helmLint(String chart_dir) {
    // lint helm chart
    sh "/usr/local/bin/helm lint ${chart_dir}"

}

def helmDeploy(Map args) {
    //configure helm client and confirm tiller process is installed

    if (args.dry_run) {
        println "Running dry-run deployment"

        sh "/usr/local/bin/helm upgrade --dry-run --debug --install ${args.name} ${args.chart_dir} --set ImageTag=${args.tag},Replicas=${args.replicas},Cpu=${args.cpu},Memory=${args.memory},DomainName=${args.name} --namespace=${args.name}"
    } else {
        println "Running deployment"
        sh "/usr/local/bin/helm upgrade --install ${args.name} ${args.chart_dir} --set ImageTag=${args.tag},Replicas=${args.replicas},Cpu=${args.cpu},Memory=${args.memory},DomainName=${args.name} --namespace=${args.name}"

        echo "Application ${args.name} successfully deployed. Use helm status ${args.name} to check"
    }
}



timeout(time: 1000, unit: 'SECONDS') {
    node {
        stage 'Check out pipeline from GitHub Repo'
        git url: 'https://github.com/showerlee/Jenkins-Pipeline-CI-CD-with-Helm-on-Kubernetes.git'
        // Setup the Docker Registry (Docker Hub) + Credentials 
        registry_url = "https://index.docker.io/v1/" // Docker Hub
        docker_creds_id = "showerlee-dockerhub" // name of the Jenkins Credentials ID
        //build_tag = "1.0" // default tag to push for to the registry       
        def pwd = pwd()
        def chart_dir = "${pwd}/charts/newegg-nginx"
        Properties props = new Properties()
        File propsFile = new File("${pwd}/promote.properties")
        props.load(propsFile.newDataInputStream())
        def build_tag_raw = props.getProperty('BUILD_TAG')
        float build_tag = Float.parseFloat(build_tag_raw)+0.1;
        println("Set current build_tag="+build_tag)
        
        def inputFile = readFile('config.json')
        def config = new groovy.json.JsonSlurperClassic().parseText(inputFile)
        println "pipeline config ==> ${config}"
        println "----------------------------------------------------------------------------"
        
        stage 'Register DockerHub'
        echo "[INFO] Register Dockerhub"
        docker.withRegistry("${registry_url}", "${docker_creds_id}") {
        
            // Set up the container to build 
            maintainer_name = "showerlee"
            container_name = "nginx-test"
            println "----------------------------------------------------------------------------"

            stage "Build Nginx Container"
            echo "[INFO] Building Nginx with docker.build(${maintainer_name}/${container_name}:${build_tag})"
            container = docker.build("${maintainer_name}/${container_name}:${build_tag}", '.')
            println "----------------------------------------------------------------------------"
            try {
                
                // Start Testing
                stage "Spin up Nginx Container"
                echo "[INFO] Spin up Nginx Container"
                
                // Run the container with the env file, mounted volumes and the ports:
                docker.image("${maintainer_name}/${container_name}:${build_tag}").withRun("--name=${container_name}  -p 80:80 ")  { c ->
                       
                    // wait for the django server to be ready for testing
                    // the 'waitUntil' block needs to return true to stop waiting
                    // in the future this will be handy to specify waiting for a max interval: 
                    // https://issues.jenkins-ci.org/browse/JENKINS-29037
                    //
                    waitUntil {
                        sh """
                        set +x
                        ss -antup | grep :::80[^0-9] | grep LISTEN | wc -l | tr -d '\n' > /tmp/wait_results
                        set -x
                        """
                        wait_results = readFile '/tmp/wait_results'

                        echo "[INFO] Wait Results(${wait_results})"
                        if ("${wait_results}" == "1")
                        {
                            echo "[INFO] Nginx is listening on port 80"
                            sh "rm -f /tmp/wait_results"
                            return true
                        }
                        else
                        {
                            echo "[INFO] Nginx is not listening on port 80 yet"
                            return false
                        }
                    } // end of waitUntil
                    
                    // At this point Nginx is running
                    echo "[INFO] Docker Container is running"
                    input 'You can check the running container on docker build server now! Click Proceed to next stage...'    
                    // this pipeline is using 3 tests 
                    // by setting it to more than 3 you can test the error handling and see the pipeline Stage View error message
                    MAX_TESTS = 3
                    for (test_num = 1; test_num <= MAX_TESTS; test_num++) {     
                        println "----------------------------------------------------------------------------"   
                        echo "Running Test(${test_num})"
                    
                        expected_results = 0
                        if (test_num == 1 ) 
                        {
                            // Test we can download the home page from the running docker container
                            echo "[INFO] Check validation of home page"
                            sh """
                            set +x
                            docker exec -t ${container_name} curl -s http://localhost | grep Welcome | wc -l | tr -d '\n' > /tmp/test_results
                            set -x
                            """
                            expected_results = 1
                        }
                        else if (test_num == 2)
                        {
                            // Test if port 80 is exposed
                            echo "[INFO] Check if port 80 is exposed"
                            sh """
                            set +x
                            docker inspect --format '{{ (.NetworkSettings.Ports) }}' ${container_name}
                            docker inspect --format '{{ (.NetworkSettings.Ports) }}' ${container_name} | grep map | grep '80/tcp:' | wc -l | tr -d '\n' > /tmp/test_results
                            set -x
                            """
                            expected_results = 1
                        }
                        else if (test_num == 3)
                        {
                            // Test there's nothing established on the port since nginx is not running:
                            echo "[INFO] Check if nothing established from nginx container"
                            sh """
                            set +x
                            docker exec -t ${container_name} ss -apn | grep 80 | grep ESTABLISHED | wc -l | tr -d '\n' > /tmp/test_results
                            set -x
                            """
                            expected_results = 0
                        }
                        else
                        {
                            err_msg = "Missing Test(${test_num})"
                            echo "[ERROR] ${err_msg}"
                            currentBuild.result = 'FAILURE'
                            error "Failed to finish container testing with Message(${err_msg})"
                        }
                        
                        // Now validate the results match the expected results
                        stage "Test(${test_num}) - Validate Results"
                        test_results = readFile '/tmp/test_results'
                        echo "[INFO] Test(${test_num}) Results($test_results) == Expected(${expected_results})"
                        sh """
                        set +x
                        if [ \"${test_results}\" != \"${expected_results}\" ]; 
                        then 
                            echo \" --------------------- Test(${test_num}) Failed--------------------\"
                            echo \" - Test(${test_num}) Failed\"
                            exit 1
                        else 
                            echo \" - Test(${test_num}) Passed\"
                            exit 0
                        fi
                        set -x
                        """

                        echo "[INFO] Finished Running Test(${test_num})"
                    
                        // cleanup after the test run
                        sh "rm -f /tmp/test_results"
                        currentBuild.result = 'SUCCESS'
                    }
                }
                
            } catch (Exception err) {
                err_msg = "Test had Exception(${err})"
                currentBuild.result = 'FAILURE'
                error "FAILED - Stopping build for Error(${err_msg})"
            }
            println "----------------------------------------------------------------------------"
            stage "Push to DockerHub"
            input 'Do you approve to push?'
            container.push()
            sh """
            echo 'BUILD_TAG=${build_tag}' > ${pwd}/promote.properties
            git add ${pwd}/promote.properties
            git commit -m"Update docker tag to ${build_tag}"
            git push origin master

            """
            currentBuild.result = 'SUCCESS'
            println "----------------------------------------------------------------------------"
            
        }
        
        stage ('helm test') { 
            echo "[INFO] Start helm test"   
            // run helm chart linter
            echo "[INFO] Run helm chart linter"
            helmLint(chart_dir)

            // dry-run helm chart installation
            echo "[INFO] Dry-run helm chart installation"
            helmDeploy(
                dry_run       : true,
                name          : config.app.name,
                chart_dir     : chart_dir,
                tag           : build_tag,
                replicas      : config.app.replicas,
                cpu           : config.app.cpu,
                memory        : config.app.memory
            )
            println "----------------------------------------------------------------------------"
        }
        
        stage ('helm deploy') {
            echo "[INFO] Start helm deployment"
            // Deploy using Helm chart
            helmDeploy(
                dry_run       : false,
                name          : config.app.name,
                chart_dir     : chart_dir,
                tag           : build_tag,
                replicas      : config.app.replicas,
                cpu           : config.app.cpu,
                memory        : config.app.memory
            )
            echo "[INFO] Deployment Finished..."
        }
        
        ///////////////////////////////////////
        //
        // Coming Soon Feature Enhancements
        //
        // 1. Add Docker Compose testing as a new Pipeline item that is initiated after this one for "Integration" testing
        // 2. Make sure to set the Pipeline's "Throttle builds" to 1 because the docker containers will collide on resources like ports and names
        // 3. Should be able to parallelize the docker.withRegistry() methods to ensure the container is running on the slave
        // 4. After the tests finish (and before they start), clean up container images to prevent stale docker image builds from affecting the current test run
    }
}
