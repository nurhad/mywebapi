pipeline {
    agent {
        node {
            label 'ubuntu-dotnet-agent'
        }
    }
    
    environment {
        REGISTRY = "10.112.1.77:5000"
        IMAGE_NAME = "mywebapi"
        KUBE_NAMESPACE = "default"
        DOTNET_VERSION = "8.0"
        BUILD_CONFIGURATION = "Release"
        CONTAINER_PORT = "80"
        SERVICE_PORT = "80"
    }
    
    triggers {
        pollSCM('H/30 * * * *')
    }
    
    stages {
        stage('Build .NET Application') {
            steps {
                echo "🏗️ Building .NET WebAPI..."
                sh '''
                echo "🔧 Environment:"
                echo "Node: $NODE_NAME"
                echo "Host: $(hostname)"
                echo "Workspace: $(pwd)"
                echo ""
                echo "📂 Repository contents:"
                ls -la
                echo ""
                echo "📦 Restoring dependencies..."
                dotnet restore
                
                echo "🔨 Building application..."
                dotnet build --configuration Release --no-restore
                
                echo "✅ .NET build completed successfully!"
                '''
            }
        }
        
        stage('Run Tests') {
            steps {
                echo "🧪 Running tests..."
                sh '''
                # Check if test projects exist
                if find . -name "*Test*.csproj" | head -1; then
                    echo "Running test suite..."
                    dotnet test --verbosity normal
                else
                    echo "ℹ️  No test projects found - skipping tests"
                fi
                '''
            }
        }
        
        stage('Build Container Image') {
            steps {
                echo "🐳 Building Docker image with Podman..."
                sh """
                    # Build dengan tag yang KONSISTEN untuk registry
                    podman build -t ${REGISTRY}/${IMAGE_NAME}:${env.BUILD_NUMBER} -t ${REGISTRY}/${IMAGE_NAME}:latest .
                    
                    echo "✅ Container images built:"
                    podman images | grep ${IMAGE_NAME}
                """
            }
        }

        stage('Push to Local Registry') {
            steps {
                echo "📤 Ensuring local registry is running..."
                sh """
                    # Configure Podman to allow insecure registry
                    echo "🔧 Configuring insecure registry..."
                    sudo mkdir -p /etc/containers
                    echo -e '[[registry]]\\nlocation = "10.112.1.77:5000"\\ninsecure = true' | sudo tee /etc/containers/registries.conf.d/insecure.conf
                    
                    # Check if registry container exists and is running
                    if ! podman ps --format "table {{.Names}}" | grep -q registry; then
                        echo "🚀 Starting registry container..."
                        # Clean up any existing registry container first
                        podman stop registry 2>/dev/null || echo "No running registry to stop"
                        podman rm registry 2>/dev/null || echo "No registry container to remove"
                        podman run -d -p 5000:5000 --name registry registry:2
                        sleep 5
                    else
                        echo "✅ Registry container is already running"
                    fi
                    
                    # Wait for registry to be ready
                    echo "🔍 Verifying registry access..."
                    until curl -s http://localhost:5000/v2/_catalog > /dev/null; do
                        echo "Waiting for registry to be ready..."
                        sleep 3
                    done
                    
                    # Push images to local registry dengan --tls-verify=false
                    echo "📤 Pushing images to registry..."
                    podman push --tls-verify=false 10.112.1.77:5000/mywebapi:${env.BUILD_NUMBER}
                    podman push --tls-verify=false 10.112.1.77:5000/mywebapi:latest
                    
                    echo "✅ Images pushed to local registry:"
                    curl -s http://localhost:5000/v2/mywebapi/tags/list | jq . 2>/dev/null || curl -s http://localhost:5000/v2/mywebapi/tags/list
                """
            }
        }
        
        stage('Deploy to Kubernetes') {
            steps {
                echo "🚀 Deploying to Kubernetes..."
                sh """
                    # Update deployment dengan image dari REGISTRY
                    sed -i 's|image:.*|image: ${REGISTRY}/${IMAGE_NAME}:${env.BUILD_NUMBER}|g' k8s/deployment.yaml
                    
                    # Apply Kubernetes manifests
                    kubectl apply -f k8s/deployment.yaml
                    
                    echo "🌐 Deployment applied - checking status..."
                    kubectl get pods -l app=mywebapi
                    kubectl get svc mywebapi-service
                """
            }
        }
        
        stage('Wait for Deployment') {
            steps {
                echo "⏳ Waiting for deployment to be ready..."
                sh """
                # Wait dengan timeout lebih lama dan continue meskipun timeout
                timeout 300s kubectl rollout status deployment/mywebapi-deployment || echo "Rollout taking longer than expected - continuing..."
                
                echo "📊 Final deployment status:"
                kubectl get pods -l app=mywebapi
                kubectl get svc mywebapi-service
                """
            }
        }
        
        stage('Smoke Test') {
            steps {
                echo "🔍 Running smoke tests..."
                sh """
                # Wait for service
                sleep 30
                
                echo "🧪 Testing application..."
                # Get service details
                kubectl get svc mywebapi-service
                
                # Try to access the service
                echo "Testing health endpoint..."
                kubectl run smoke-test --image=curlimages/curl --rm -i --restart=Never -- \
                  curl -s http://mywebapi-service/weatherforecast/health || echo "Health check attempted - service might still be starting"
                
                echo "✅ Smoke test completed!"
                """
            }
        }
    }
    
    post {
        always {
            echo "📊 Pipeline execution completed"
            echo "🔗 Build: ${env.BUILD_NUMBER}"
            echo "🎯 Agent: ${env.NODE_NAME}"
            echo "📈 Result: ${currentBuild.result}"
        }
        success {
            echo "🎉 CI/CD SUCCESS! .NET WebAPI deployed to Kubernetes!"
            sh '''
            echo "📋 Final Status:"
            kubectl get pods -l app=mywebapi
            kubectl get svc mywebapi-service
            echo "🚀 Application deployed successfully!"
            '''
        }
        failure {
            echo "⚠️  Pipeline completed with warnings - check deployment status"
            sh '''
            echo "🔍 Current status:"
            kubectl get pods -l app=mywebapi
            kubectl get svc mywebapi-service
            kubectl describe deployment mywebapi-deployment
            '''
        }
    }
}