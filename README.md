name: CI/CD — build, test, push, deploy
uses: docker/setup-buildx-action@v3


- name: Log in to GHCR
uses: docker/login-action@v3
with:
registry: ghcr.io
username: ${{ github.actor }}
password: ${{ secrets.GHCR_PAT }}


- name: Build and push image
uses: docker/build-push-action@v4
with:
context: .
push: true
tags: |
${{ env.IMAGE_NAME }}:latest
${{ env.IMAGE_NAME }}:${{ github.sha }}


deploy:
needs: build-and-push
runs-on: ubuntu-latest
if: github.ref == 'refs/heads/main'
steps:
- name: Checkout
uses: actions/checkout@v4


- name: Install kubectl
uses: azure/setup-kubectl@v3
with:
version: '1.30.0'


- name: Configure Kubeconfig
run: |
echo "${{ secrets.KUBE_CONFIG }}" > kubeconfig
export KUBECONFIG=$PWD/kubeconfig
kubectl config view


- name: Update image in Deployment and apply manifests
env:
IMAGE: ${{ env.IMAGE_NAME }}:${{ github.sha }}
run: |
export KUBECONFIG=$PWD/kubeconfig
# set the image for the deployment and apply k8s manifests
kubectl -n clients-api set image deployment/clients-api clients-api=$IMAGE --record || true
kubectl apply -f k8s/namespace.yaml
kubectl apply -f k8s/ingress-controller/nginx-ingress-controller.yaml
kubectl apply -f k8s/cert-manager/cluster-issuer-staging.yaml
kubectl apply -f k8s/cert-manager/certificate.yaml
kubectl apply -f k8s/mongodb/service.yaml
kubectl apply -f k8s/mongodb/statefulset.yaml
kubectl apply -f k8s/clients-api/configmap.yaml
kubectl apply -f k8s/clients-api/secret-example.yaml
kubectl apply -f k8s/clients-api/service.yaml
kubectl apply -f k8s/clients-api/deployment.yaml
kubectl apply -f k8s/clients-api/ingress.yaml
