pipeline {
    agent any

    environment {
        DOCKER_IMAGE = "2022bcs0206kishan/wine_predict_jenkins:latest"
        CONTAINER_NAME = "test_inference_2022BCS0206"
        API_PORT = "8000"
        API_URL = "http://localhost:${API_PORT}"
    }

    stages {

        stage('Pull Image') {
            steps {
                script {
                    echo "2022BCS0206: Pulling Docker image from Docker Hub..."
                    sh "docker pull ${DOCKER_IMAGE}"
                    echo "2022BCS0206: Image pulled successfully."
                }
            }
        }

        stage('Run Container') {
            steps {
                script {
                    echo "2022BCS0206: Starting inference container..."
                    sh """
                        docker stop ${CONTAINER_NAME} || true
                        docker rm ${CONTAINER_NAME} || true
                        docker run -d \
                            --name ${CONTAINER_NAME} \
                            -p ${API_PORT}:${API_PORT} \
                            ${DOCKER_IMAGE}
                    """
                    echo "2022BCS0206: Container started: ${CONTAINER_NAME}"
                }
            }
        }

        stage('Wait for Service Readiness') {
            steps {
                script {
                    echo "2022BCS0206: Waiting for API to be ready..."
                    def maxRetries = 15
                    def retries = 0
                    def ready = false

                    while (retries < maxRetries && !ready) {
                        try {
                            def response = sh(
                                script: "curl -s -o /dev/null -w '%{http_code}' ${API_URL}/",
                                returnStdout: true
                            ).trim()

                            if (response == "200") {
                                ready = true
                                echo "2022BCS0206: API is ready! Status: ${response}"
                            } else {
                                echo "2022BCS0206: API not ready yet. Status: ${response}. Retry ${retries + 1}/${maxRetries}"
                                sleep(2)
                                retries++
                            }
                        } catch (Exception e) {
                            echo "2022BCS0206: Waiting... Retry ${retries + 1}/${maxRetries}"
                            sleep(2)
                            retries++
                        }
                    }

                    if (!ready) {
                        error("2022BCS0206: API did not become ready within timeout. Pipeline FAILED.")
                    }
                }
            }
        }

        stage('Send Valid Inference Request') {
            steps {
                script {
                    echo "2022BCS0206: Sending valid inference request..."

                    def validPayload = '''{
                        "fixed_acidity": 7.4,
                        "volatile_acidity": 0.70,
                        "citric_acid": 0.00,
                        "residual_sugar": 1.9,
                        "chlorides": 0.076,
                        "free_sulfur_dioxide": 11.0,
                        "total_sulfur_dioxide": 34.0,
                        "density": 0.9978,
                        "pH": 3.51,
                        "sulphates": 0.56,
                        "alcohol": 9.4
                    }'''

                    def response = sh(
                        script: """curl -s -w "\\nHTTP_STATUS:%{http_code}" \
                            -X POST \
                            -H "Content-Type: application/json" \
                            -d '${validPayload}' \
                            ${API_URL}/predict""",
                        returnStdout: true
                    ).trim()

                    echo "2022BCS0206: Raw response: ${response}"

                    def parts = response.split("HTTP_STATUS:")
                    def body = parts[0].trim()
                    def httpStatus = parts[1].trim()

                    echo "2022BCS0206: Response body: ${body}"
                    echo "2022BCS0206: HTTP Status: ${httpStatus}"

                    // Validate HTTP status
                    if (httpStatus != "200") {
                        error("2022BCS0206: Valid request FAILED. Expected HTTP 200, got ${httpStatus}")
                    }
                    echo "2022BCS0206: ✓ HTTP Status check PASSED: ${httpStatus}"

                    // Validate prediction field exists
                    if (!body.contains("wine_quality")) {
                        error("2022BCS0206: Valid request FAILED. 'wine_quality' field missing in response.")
                    }
                    echo "2022BCS0206: ✓ Prediction field check PASSED: 'wine_quality' found"

                    // Validate prediction is numeric
                    def predMatch = body =~ /"wine_quality"\s*:\s*(\d+)/
                    if (!predMatch) {
                        error("2022BCS0206: Valid request FAILED. 'wine_quality' value is not numeric.")
                    }
                    echo "2022BCS0206: ✓ Numeric value check PASSED: wine_quality = ${predMatch[0][1]}"

                    echo "2022BCS0206: ✓ ALL valid request validations PASSED"
                }
            }
        }

        stage('Send Invalid Request') {
            steps {
                script {
                    echo "2022BCS0206: Sending invalid inference request..."

                    def invalidPayload = '''{
                        "fixed_acidity": "not_a_number",
                        "volatile_acidity": "invalid"
                    }'''

                    def response = sh(
                        script: """curl -s -w "\\nHTTP_STATUS:%{http_code}" \
                            -X POST \
                            -H "Content-Type: application/json" \
                            -d '${invalidPayload}' \
                            ${API_URL}/predict""",
                        returnStdout: true
                    ).trim()

                    echo "2022BCS0206: Raw response: ${response}"

                    def parts = response.split("HTTP_STATUS:")
                    def body = parts[0].trim()
                    def httpStatus = parts[1].trim()

                    echo "2022BCS0206: Response body: ${body}"
                    echo "2022BCS0206: HTTP Status: ${httpStatus}"

                    // Validate API returns error (422 Unprocessable Entity for FastAPI)
                    if (httpStatus == "200") {
                        error("2022BCS0206: Invalid request test FAILED. API should return error but returned 200.")
                    }
                    echo "2022BCS0206: ✓ Error response check PASSED: Got expected error status ${httpStatus}"

                    // Validate error message is meaningful
                    if (!body.contains("detail") && !body.contains("error") && !body.contains("value")) {
                        error("2022BCS0206: Invalid request test FAILED. Error message is not meaningful.")
                    }
                    echo "2022BCS0206: ✓ Meaningful error message check PASSED"

                    echo "2022BCS0206: ✓ ALL invalid request validations PASSED"
                }
            }
        }

        stage('Stop Container') {
            steps {
                script {
                    echo "2022BCS0206: Stopping and removing container..."
                    sh """
                        docker stop ${CONTAINER_NAME} || true
                        docker rm ${CONTAINER_NAME} || true
                    """
                    // Verify no leftover containers
                    def running = sh(
                        script: "docker ps --filter name=${CONTAINER_NAME} --format '{{.Names}}'",
                        returnStdout: true
                    ).trim()

                    if (running) {
                        error("2022BCS0206: Container still running after stop! ${running}")
                    }
                    echo "2022BCS0206: ✓ Container stopped and removed cleanly."
                }
            }
        }

        stage('Pipeline Result') {
            steps {
                script {
                    echo "================================================"
                    echo "2022BCS0206: PIPELINE RESULT SUMMARY"
                    echo "================================================"
                    echo "2022BCS0206: ✓ Stage 1 - Pull Image         : PASSED"
                    echo "2022BCS0206: ✓ Stage 2 - Run Container      : PASSED"
                    echo "2022BCS0206: ✓ Stage 3 - Service Readiness  : PASSED"
                    echo "2022BCS0206: ✓ Stage 4 - Valid Request      : PASSED"
                    echo "2022BCS0206: ✓ Stage 5 - Invalid Request    : PASSED"
                    echo "2022BCS0206: ✓ Stage 6 - Stop Container     : PASSED"
                    echo "================================================"
                    echo "2022BCS0206: ALL VALIDATIONS PASSED - PIPELINE SUCCESS"
                    echo "================================================"
                }
            }
        }
    }

    post {
        always {
            script {
                echo "2022BCS0206: Post-build cleanup..."
                sh "docker stop ${CONTAINER_NAME} || true"
                sh "docker rm ${CONTAINER_NAME} || true"
            }
        }
        success {
            echo "2022BCS0206: Pipeline completed SUCCESSFULLY. All inference validations passed."
        }
        failure {
            echo "2022BCS0206: Pipeline FAILED. Check logs above for details."
        }
    }
}
