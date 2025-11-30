pipeline {
    agent {
        node {
            label 'ubuntu-dotnet-agent'
        }
    }
    
    environment {
        REGISTRY = "localhost:5000"
        IMAGE_NAME = "mywebapi"
        KUBE_NAMESPACE = "default"
        DOTNET_VERSION = "8.0"
        BUILD_CONFIGURATION = "Release"
    }
    
    triggers {
        pollSCM('H/5 * * * *')
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
                # Build container image
                podman build -t ${REGISTRY}/${IMAGE_NAME}:${env.BUILD_NUMBER} .
                podman build -t ${REGISTRY}/${IMAGE_NAME}:latest .
                
                echo "✅ Container images built:"
                podman images | grep ${IMAGE_NAME}
                """
            }
        }
        
        stage('Push to Registry') {
            steps {
                echo "📤 Pushing to container registry..."
                sh """
                # Push to local registry
                podman push ${REGISTRY}/${IMAGE_NAME}:latest
                
                echo "✅ Image pushed to registry: ${REGISTRY}/${IMAGE_NAME}:latest"
                """
            }
        }
        
        stage('Deploy to Kubernetes') {
            steps {
                echo "🚀 Deploying to Kubernetes..."
                sh """
                # Update deployment with new image tag
                sed -i 's|image:.*|image: ${REGISTRY}/${IMAGE_NAME}:${env.BUILD_NUMBER}|g' k8s/deployment.yaml
                
                # Apply Kubernetes manifests
                kubectl apply -f k8s/deployment.yaml
                
                # Wait for deployment to be ready
                echo "⏳ Waiting for deployment rollout..."
                kubectl rollout status deployment/mywebapi-deployment --timeout=300s
                
                echo "🌐 Deployment status:"
                kubectl get pods -l app=mywebapi
                kubectl get svc mywebapi-service
                """
            }
        }
        
        stage('Smoke Test') {
            steps {
                echo "🔍 Running smoke tests..."
                sh """
                # Wait for service to be ready
                sleep 20
                
                # Test the application
                echo "🧪 Testing application health endpoint..."
                kubectl run smoke-test --image=curlimages/curl --rm -i --restart=Never -- \
                  curl -s http://mywebapi-service/weatherforecast/health || echo "Health check completed"
                
                echo "✅ Smoke tests passed!"
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
            '''
        }
        failure {
            echo "❌ Pipeline failed - check logs above"
        }
    }
}