pipeline {
    agent any
    parameters {
        string(name: 'INSTANCE_1_URL', defaultValue: 'http://75.101.217.176:8080', description: 'Jenkins Instance 1 URL')
        string(name: 'INSTANCE_2_URL', defaultValue: 'http://54.80.55.7:8080', description: 'Jenkins Instance 2 URL')
        string(name: 'INSTANCE_1_JOBS', defaultValue: 'job1', description: 'Comma-separated job names for Instance 1')
        string(name: 'INSTANCE_2_JOBS', defaultValue: 'job2,job3', description: 'Comma-separated job names for Instance 2')
    }
    stages {
        stage('Trigger Jobs') {
            steps {
                script {
                    // Initialize a list to collect job results
                    def jobResults = []
                    // Trigger jobs for each instance in parallel
                    parallel (
                        "Instance 1 Jobs": {
                            withCredentials([usernamePassword(credentialsId: 'instance1-credentials', usernameVariable: 'USERNAME', passwordVariable: 'TOKEN')]) {
                                jobResults += triggerJobs(
                                    params.INSTANCE_1_URL,
                                    env.USERNAME,
                                    env.TOKEN,
                                    params.INSTANCE_1_JOBS
                                )
                            }
                        },
                        "Instance 2 Jobs": {
                            withCredentials([usernamePassword(credentialsId: 'instance2-credentials', usernameVariable: 'USERNAME', passwordVariable: 'TOKEN')]) {
                                jobResults += triggerJobs(
                                    params.INSTANCE_2_URL,
                                    env.USERNAME,
                                    env.TOKEN,
                                    params.INSTANCE_2_JOBS
                                )
                            }
                        }
                    )
                    // Log all job results
                    jobResults.each { echo it }
                }
            }
        }
    }
    post {
        always {
            echo "Pipeline execution completed."
        }
    }
}
def triggerJobs(instanceUrl, user, token, jobs) {
    // Split comma-separated job names into a list
    def jobList = jobs.split(',')
    def jobResults = [] // Track results for each job
    for (job in jobList) {
        echo "Triggering job: ${job} on ${instanceUrl}"
        def triggerUrl = "${instanceUrl}/job/${job}/build"
        def responseCode = sh(
            script: """
                curl -s -o /dev/null -w "%{http_code}" -X POST "${triggerUrl}" \
                --user ${user}:${token}
            """,
            returnStdout: true
        ).trim()
        if (responseCode == '201') {
            echo "Successfully triggered job: ${job}"
            // Wait for the job to complete and retrieve its status
            def jobStatus = getJobStatus(instanceUrl, user, token, job)
            jobResults << "Job: ${job} Status: ${jobStatus.status} - ${jobStatus.message}"
        } else {
            echo "Failed to trigger job: ${job}. Response code: ${responseCode}"
            jobResults << "Job: ${job} could not be triggered."
        }
    }
    return jobResults // Return all results
}
def getJobStatus(instanceUrl, user, token, job) {
    def jobInfoUrl = "${instanceUrl}/job/${job}/lastBuild/api/json"
    // Poll for job completion
    while (true) {
        def jobResponse = sh(
            script: """
                curl -s --user ${user}:${token} "${jobInfoUrl}"
            """,
            returnStdout: true
        ).trim()
        def json = readJSON(text: jobResponse)
        // Check the status of the last build
        def jobStatus = json.result
        if (jobStatus != null) {
            return [status: jobStatus, message: jobStatus] // Return status if completed
        }
        echo "Job: ${job} is still running. Waiting for completion..."
        sleep(time: 10, unit: 'SECONDS') // Wait before checking again
    }
}
