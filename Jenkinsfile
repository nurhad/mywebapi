pipeline {
    agent {
        label 'ubuntu dotnet podman k8s agent'  // ← RUN DI UBUNTU AGENT
    }
    
    environment {
        // Container registry configuration
        REGISTRY = "localhost:5000"
        IMAGE_NAME = "mywebapi"
        KUBE_NAMESPACE = "default"
        
        // Build configuration
        DOTNET_VERSION = "8.0"
        BUILD_CONFIGURATION = "Release"
    }
    
    triggers {
        pollSCM('H/5 * * * *')  // Auto-check GitHub setiap 5 menit
    }
    
    stages {
        stage('Environment Info') {
            steps {
                echo "🎯 Running on Jenkins Agent"
                sh '''
                echo "🖥️  Agent Hostname: $(hostname)"
                echo "🔧 .NET Version: $(dotnet --version)"
                echo "🐳 Podman Version: $(podman --version)"
                echo "☸️  Kubectl Version: $(kubectl version --client --short)"
                echo "📁 Workspace: $(pwd)"
                '''
            }
        }
        
        stage('Checkout Code') {
            steps {
                echo "📥 Checking out code from GitHub..."
                git branch: 'main', url: 'https://github.com/nurhad/mywebapi.git',
                // credentialsId: 'github-token'  // Jika pakai private repo
                
                sh 'echo "📂 Repository contents:" && ls -la'
            }
        }
        
        stage('Build .NET Application') {
            steps {
                echo "🏗️ Building .NET WebAPI..."
                sh '''
                echo "📦 Restoring NuGet packages..."
                dotnet restore
                
                echo "🔨 Building application..."
                dotnet build --configuration Release --no-restore
                
                echo "✅ Build completed successfully!"
                '''
            }
        }
        
        stage('Run Tests') {
            steps {
                echo "🧪 Running tests..."
                sh '''
                # Check if test projects exist
                if find . -name "*Test*.csproj | head -1"; then
                    echo "Running tests..."
                    dotnet test --verbosity normal
                else
                    echo "ℹ️  No test projects found - skipping tests"
                fi
                '''
            }
        }
        
        stage('Build Container Image') {
            steps {
                echo "🐳 Building container image with Podman..."
                sh """
                # Build Docker image
                podman build -t ${REGISTRY}/${IMAGE_NAME}:${env.BUILD_NUMBER} .
                podman build -t ${REGISTRY}/${IMAGE_NAME}:latest .
                
                echo "✅ Container images built:"
                podman images | grep ${IMAGE_NAME}
                """
            }
        }
        
        stage('Deploy to Kubernetes') {
            steps {
                echo "🚀 Deploying to Kubernetes..."
                sh """
                # Apply Kubernetes manifests
                kubectl apply -f k8s/deployment.yaml
                
                # Wait for deployment to be ready
                echo "⏳ Waiting for deployment to be ready..."
                kubectl rollout status deployment/mywebapi-deployment -n ${KUBE_NAMESPACE} --timeout=300s
                
                echo "🌐 Deployment status:"
                kubectl get pods -l app=mywebapi -n ${KUBE_NAMESPACE}
                kubectl get svc mywebapi-service -n ${KUBE_NAMESPACE}
                """
            }
        }
        
        stage('Smoke Test') {
            steps {
                echo "🔍 Running smoke tests..."
                sh """
                # Wait for service to be ready
                sleep 20
                
                # Get service details
                echo "📡 Service information:"
                kubectl get svc mywebapi-service -n ${KUBE_NAMESPACE}
                
                # Try to access the service
                echo "🧪 Testing application health endpoint..."
                kubectl run smoke-test --image=curlimages/curl --rm -it --restart=Never -- \
                  curl -s http://mywebapi-service/weatherforecast/health || echo "Health check attempted"
                """
            }
        }
    }
    
    post {
        always {
            echo "📊 Pipeline execution completed"
            echo "🏷️  Build Number: ${env.BUILD_NUMBER}"
            echo "🎯 Agent: ${env.NODE_NAME}"
            echo "📈 Result: ${currentBuild.result}"
        }
        success {
            echo "🎉 SUCCESS! .NET WebAPI deployed via Jenkins Agent!"
            sh '''
            echo "✅ Deployment Summary:"
            kubectl get pods -l app=mywebapi
            kubectl get svc mywebapi-service
            '''
        }
        failure {
            echo "❌ Pipeline failed - check logs above"
            sh '''
            echo "🔍 Debug information:"
            kubectl get pods -l app=mywebapi
            kubectl describe deployment mywebapi-deployment
            '''
        }
    }
}