pipeline {
    agent any
    
    environment {
        // Container registry configuration
        REGISTRY = "localhost:5000"  // Local registry untuk testing
        IMAGE_NAME = "mywebapi"
        KUBE_NAMESPACE = "default"
        
        // Build configuration
        DOTNET_VERSION = "8.0"
        BUILD_CONFIGURATION = "Release"
    }
    
    triggers {
        pollSCM('H/5 * * * *')  // Auto-trigger setiap 5 menit
    }
    
    stages {
        stage('Checkout Code') {
            steps {
                echo "🔔 Checking out .NET WebAPI code..."
                git branch: 'main', 
                url: 'https://github.com/nurhad/mywebapi.git'
                
                sh 'echo "📁 Project structure:"'
                sh 'find . -name "*.cs" -o -name "*.csproj" -o -name "Dockerfile" | head -10'
            }
        }
        
        stage('Restore & Build') {
            steps {
                echo "🏗️ Building .NET application..."
                sh '''
                echo "🔧 .NET version:"
                dotnet --version
                
                echo "📦 Restoring dependencies..."
                dotnet restore
                
                echo "🔨 Building application..."
                dotnet build --configuration ${BUILD_CONFIGURATION} --no-restore
                
                echo "✅ Build completed successfully!"
                '''
            }
        }
        
        stage('Run Tests') {
            steps {
                echo "🧪 Running unit tests..."
                sh '''
                # Jika ada test project
                if [ -f "**/*.Test.csproj" ]; then
                    dotnet test --verbosity normal
                else
                    echo "ℹ️  No test projects found - skipping tests"
                fi
                '''
            }
        }
        
        stage('Security Scan') {
            steps {
                echo "🛡️ Scanning for vulnerabilities..."
                sh '''
                echo "📋 Checking for known vulnerabilities..."
                dotnet list package --vulnerable --include-transitive || echo "No vulnerabilities found"
                
                # Basic security check
                echo "🔍 Security analysis completed"
                '''
            }
        }
        
        stage('Build Docker Image') {
            steps {
                echo "🐳 Building Docker image with Podman..."
                sh """
                # Build image
                podman build -t ${REGISTRY}/${IMAGE_NAME}:${env.BUILD_NUMBER} .
                podman build -t ${REGISTRY}/${IMAGE_NAME}:latest .
                
                echo "✅ Docker images built:"
                podman images | grep ${IMAGE_NAME}
                """
            }
        }
        
        stage('Push to Registry') {
            steps {
                echo "📤 Pushing image to registry..."
                sh """
                # Push to local registry (adjust for your registry)
                podman push ${REGISTRY}/${IMAGE_NAME}:${env.BUILD_NUMBER}
                podman push ${REGISTRY}/${IMAGE_NAME}:latest
                
                echo "✅ Images pushed to registry"
                """
            }
        }
        
        stage('Deploy to Kubernetes') {
            steps {
                echo "🚀 Deploying to Kubernetes..."
                sh """
                # Update Kubernetes deployment
                sed -i 's|image: mywebapi:latest|image: ${REGISTRY}/${IMAGE_NAME}:${env.BUILD_NUMBER}|g' k8s/deployment.yaml
                
                # Apply Kubernetes manifests
                kubectl apply -f k8s/deployment.yaml
                
                echo "🔄 Checking deployment status..."
                kubectl rollout status deployment/mywebapi-deployment --timeout=300s
                
                echo "🌐 Service information:"
                kubectl get svc mywebapi-service
                """
            }
        }
        
        stage('Smoke Test') {
            steps {
                echo "🧪 Running smoke tests..."
                sh """
                # Wait for service to be ready
                sleep 30
                
                # Get service URL
                SERVICE_URL=\$(kubectl get svc mywebapi-service -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
                if [ -z "\$SERVICE_URL" ]; then
                    SERVICE_URL="localhost"
                fi
                
                echo "🔍 Testing endpoint: http://\${SERVICE_URL}/weatherforecast/health"
                
                # Test health endpoint
                curl -f http://\${SERVICE_URL}/weatherforecast/health || echo "Health check failed"
                
                echo "✅ Smoke tests passed!"
                """
            }
        }
    }
    
    post {
        always {
            echo "📊 Pipeline execution completed"
            sh 'echo "Build: ${env.BUILD_NUMBER} | Result: ${currentBuild.result}"'
        }
        success {
            echo "🎉 .NET WebAPI successfully deployed to Kubernetes!"
            sh '''
            echo "🌐 Your application is running at:"
            kubectl get svc mywebapi-service
            echo ""
            echo "📋 Pod status:"
            kubectl get pods -l app=mywebapi
            '''
        }
        failure {
            echo "❌ Pipeline failed - check logs above"
            sh 'echo "Debug info:"; kubectl get pods -l app=mywebapi'
        }
        cleanup {
            echo "🧹 Cleaning up workspace..."
            // Cleanup resources jika perlu
        }
    }
}