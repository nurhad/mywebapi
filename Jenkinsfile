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
        
        stage('Setup Registry') {
            steps {
                echo "🐳 Setting up Local Registry..."
                sh """
                    # Configure insecure registry untuk Podman (user level)
                    echo "🔧 Configuring insecure registry for Podman..."
                    mkdir -p \$HOME/.config/containers
                    cat > \$HOME/.config/containers/registries.conf << 'EOF'
unqualified-search-registries = ["docker.io"]

[[registry]]
location = "localhost:5000"
insecure = true

[[registry]]
location = "10.112.1.77:5000"
insecure = true
EOF
                    
                    echo "📋 Registry configuration:"
                    cat \$HOME/.config/containers/registries.conf
                    
                    # Start registry container jika belum berjalan
                    echo "🚀 Checking registry container..."
                    if ! podman ps --format "{{.Names}}" | grep -q registry; then
                        echo "Starting registry container..."
                        podman run -d -p 5000:5000 --name registry registry:2
                        echo "✅ Registry container started"
                    else
                        echo "✅ Registry container is already running"
                    fi
                    
                    # Wait for registry to be ready
                    echo "🔍 Verifying registry access..."
                    COUNTER=0
                    while ! curl -s http://localhost:5000/v2/_catalog > /dev/null && [ \$COUNTER -lt 10 ]; do
                        echo "Waiting for registry to be ready... (\$((COUNTER*3)) seconds)"
                        sleep 3
                        COUNTER=\$((COUNTER + 1))
                    done
                    
                    if curl -s http://localhost:5000/v2/_catalog > /dev/null; then
                        echo "✅ Registry is ready and accessible"
                        curl -s http://localhost:5000/v2/_catalog | jq . 2>/dev/null || curl -s http://localhost:5000/v2/_catalog
                    else
                        echo "❌ Registry not accessible after 30 seconds"
                        exit 1
                    fi
                """
            }
        }
        
        stage('Build Container Image') {
            steps {
                echo "🐳 Building Container Image..."
                sh """
                    # Clean old images dengan tag yang sama
                    podman rmi \${REGISTRY}/\${IMAGE_NAME}:\${env.BUILD_NUMBER} 2>/dev/null || true
                    podman rmi \${REGISTRY}/\${IMAGE_NAME}:latest 2>/dev/null || true
                    
                    # Build image dengan tag untuk registry lokal
                    echo "🔨 Building image with tag: \${REGISTRY}/\${IMAGE_NAME}:\${env.BUILD_NUMBER}"
                    podman build -t \${REGISTRY}/\${IMAGE_NAME}:\${env.BUILD_NUMBER} -t \${REGISTRY}/\${IMAGE_NAME}:latest .
                    
                    echo "✅ Container images built:"
                    podman images | grep \${IMAGE_NAME} | head -5
                """
            }
        }

        stage('Push to Local Registry') {
            steps {
                echo "📤 Pushing to Local Registry..."
                sh """
                    # Verify registry is still running
                    if ! podman ps --format "{{.Names}}" | grep -q registry; then
                        echo "❌ Registry container not running!"
                        exit 1
                    fi
                    
                    # Push images dengan retry logic
                    echo "📤 Pushing \${REGISTRY}/\${IMAGE_NAME}:\${env.BUILD_NUMBER}"
                    for i in {1..3}; do
                        if podman push --tls-verify=false \${REGISTRY}/\${IMAGE_NAME}:\${env.BUILD_NUMBER}; then
                            echo "✅ Successfully pushed :\${env.BUILD_NUMBER}"
                            break
                        else
                            echo "⚠️ Push attempt \${i} failed, retrying in 5 seconds..."
                            sleep 5
                        fi
                    done
                    
                    echo "📤 Pushing \${REGISTRY}/\${IMAGE_NAME}:latest"
                    for i in {1..3}; do
                        if podman push --tls-verify=false \${REGISTRY}/\${IMAGE_NAME}:latest; then
                            echo "✅ Successfully pushed :latest"
                            break
                        else
                            echo "⚠️ Push attempt \${i} failed, retrying in 5 seconds..."
                            sleep 5
                        fi
                    done
                    
                    # Verify pushed images
                    echo "✅ Images pushed to local registry:"
                    echo "📋 Catalog:"
                    curl -s http://localhost:5000/v2/_catalog | jq . 2>/dev/null || curl -s http://localhost:5000/v2/_catalog
                    
                    echo "🏷️ Tags for \${IMAGE_NAME}:"
                    curl -s http://localhost:5000/v2/\${IMAGE_NAME}/tags/list | jq . 2>/dev/null || curl -s http://localhost:5000/v2/\${IMAGE_NAME}/tags/list
                """
            }
        }
        
        stage('Deploy to Kubernetes') {
            steps {
                echo "🚀 Deploying to Kubernetes..."
                sh """
                    # Configure K3s untuk bisa pull dari registry insecure
                    echo "🔧 Configuring K3s for insecure registry..."
                    sudo mkdir -p /etc/rancher/k3s/
                    cat > /tmp/registries.yaml << 'EOF'
mirrors:
  "localhost:5000":
    endpoint:
      - "http://localhost:5000"
EOF
                    sudo cp /tmp/registries.yaml /etc/rancher/k3s/registries.yaml
                    
                    # Restart k3s agar konfigurasi berlaku
                    echo "🔄 Restarting K3s to apply registry config..."
                    sudo systemctl restart k3s
                    sleep 15
                    
                    # Wait for k3s to be ready
                    echo "⏳ Waiting for K3s to be ready..."
                    until kubectl get nodes 2>/dev/null; do
                        echo "K3s not ready yet, waiting..."
                        sleep 5
                    done
                    
                    # Update deployment dengan image dari registry lokal
                    echo "📝 Updating deployment image to: \${REGISTRY}/\${IMAGE_NAME}:\${env.BUILD_NUMBER}"
                    sed -i 's|image:.*|image: \${REGISTRY}/\${IMAGE_NAME}:\${env.BUILD_NUMBER}|g' k8s/deployment.yaml
                    
                    # Apply Kubernetes manifests
                    kubectl apply -f k8s/deployment.yaml
                    
                    echo "✅ Deployment applied!"
                """
            }
        }
        
        stage('Wait for Deployment') {
            steps {
                echo "⏳ Waiting for deployment to be ready..."
                sh """
                    # Wait for rollout dengan timeout
                    echo "🔄 Checking rollout status..."
                    timeout 180s kubectl rollout status deployment/mywebapi-deployment || echo "⚠️ Rollout check timed out"
                    
                    echo "📊 Deployment status:"
                    kubectl get deployment mywebapi-deployment
                    
                    echo "🐳 Pods status:"
                    kubectl get pods -l app=\${IMAGE_NAME} -o wide
                    
                    echo "🔗 Service status:"
                    kubectl get svc mywebapi-service
                """
            }
        }
        
        stage('Smoke Test') {
            steps {
                echo "🔍 Running smoke tests..."
                sh """
                    # Wait for service to be ready
                    echo "⏳ Waiting for service to stabilize..."
                    sleep 20
                    
                    # Get service details
                    echo "🧪 Testing application endpoints..."
                    
                    # Test 1: Health endpoint
                    echo "🩺 Testing health endpoint..."
                    kubectl run smoke-test-health-\${env.BUILD_NUMBER} \
                        --image=curlimages/curl \
                        --restart=Never \
                        --rm -i \
                        --command -- \
                        curl -s -f -o /dev/null -w "Health: %{http_code}\\n" \
                        http://mywebapi-service/healthz \
                        && echo "✅ Health check passed" \
                        || echo "⚠️ Health check failed"
                    
                    # Test 2: Weather forecast endpoint
                    echo "🌤️ Testing weather forecast endpoint..."
                    kubectl run smoke-test-weather-\${env.BUILD_NUMBER} \
                        --image=curlimages/curl \
                        --restart=Never \
                        --rm -i \
                        --command -- \
                        curl -s -f -o /dev/null -w "Weather: %{http_code}\\n" \
                        http://mywebapi-service/weatherforecast \
                        && echo "✅ Weather endpoint passed" \
                        || echo "⚠️ Weather endpoint failed"
                    
                    echo "✅ Smoke test completed!"
                """
            }
        }
    }
    
    post {
        always {
            echo "📊 Pipeline execution completed"
            echo "🔗 Build: \${env.BUILD_NUMBER}"
            echo "🎯 Agent: \${env.NODE_NAME}"
            echo "📈 Result: \${currentBuild.currentResult}"
            
            sh '''
            echo "📋 Final Status Report:"
            echo "=== Deployment ==="
            kubectl get deployment mywebapi-deployment
            echo "=== Pods ==="
            kubectl get pods -l app=mywebapi -o wide
            echo "=== Service ==="
            kubectl get svc mywebapi-service
            echo "=== Registry ==="
            curl -s http://localhost:5000/v2/_catalog 2>/dev/null || echo "Registry not accessible"
            '''
        }
        success {
            echo "🎉 CI/CD SUCCESS! .NET WebAPI deployed to Kubernetes!"
            sh '''
            echo "🚀 Application deployed successfully!"
            echo ""
            echo "📌 To access the application:"
            echo "1. Port forward: kubectl port-forward svc/mywebapi-service 8080:80"
            echo "2. Then open: http://localhost:8080/weatherforecast"
            echo ""
            echo "📌 To check logs:"
            echo "kubectl logs -l app=mywebapi --tail=20"
            '''
        }
        failure {
            echo "❌ Pipeline failed!"
            sh '''
            echo "🔍 Debug information:"
            echo "=== Pods details ==="
            kubectl describe pods -l app=mywebapi
            echo "=== Deployment details ==="
            kubectl describe deployment mywebapi-deployment
            echo "=== Events ==="
            kubectl get events --field-selector involvedObject.name=mywebapi-deployment --sort-by=.lastTimestamp
            echo "=== Registry logs ==="
            podman logs registry 2>/dev/null | tail -20 || echo "Could not get registry logs"
            '''
        }
    }
}