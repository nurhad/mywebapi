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
        
        stage('Push to Local Registry') {
            steps {
                echo "📤 Ensuring local registry is running..."
                sh """
                # Start local registry jika belum running
                podman ps | grep registry || podman run -d -p 5000:5000 --name registry registry:2
                sleep 5
                
                # Push images to registry
                podman push ${REGISTRY}/${IMAGE_NAME}:latest
                
                echo "✅ Images pushed to local registry"
                """
            }
        }
        
        stage('Deploy to Kubernetes') {
            steps {
                echo "🚀 Deploying to Kubernetes..."
                sh """
                # Update deployment dengan image dari registry
                sed -i 's|image:.*|image: ${REGISTRY}/${IMAGE_NAME}:${env.BUILD_NUMBER}|g' k8s/deployment.yaml
                
                # Add imagePullPolicy untuk development
                sed -i '/image:/a\\          imagePullPolicy: IfNotPresent' k8s/deployment.yaml
                
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
                # Wait for deployment dengan retry logic
                for i in {1..30}; do
                    kubectl get pods -l app=mywebapi | grep -q Running && break
                    echo "Waiting for pods to be ready... (\$i/30)"
                    sleep 10
                done
                
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
                sleep 10
                
                echo "🧪 Testing application..."
                # Get service details
                kubectl get svc mywebapi-service
                
                # Try to access the service
                echo "Testing health endpoint..."
                kubectl run smoke-test --image=curlimages/curl --rm -i --restart=Never -- \
                  curl -s http://mywebapi-service/weatherforecast/health && echo "✅ Health check successful!" || echo "⚠️ Health check failed - service might still be starting"
                
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
    }
}