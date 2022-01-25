Taken from: https://akomljen.com/kubernetes-nginx-ingress-controller/


- Set up your app and expose as service.

kubectl create -f app-deployment.yaml -f app-service.yaml

- Create an nginx ingress controller.

kubectl create namespace ingress
kubectl create -f default-backend-deployment.yaml -f default-backend-service.yaml -n=ingress 
kubectl create -f nginx-ingress-controller-config-map.yaml -n=ingress
kubectl create -f nginx-ingress-controller-roles.yaml -n=ingress
kubectl create -f nginx-ingress-controller-deployment.yaml -n=ingress


- Set up proxy rules

kubectl create -f nginx-ingress.yaml -n=ingress
kubectl create -f app-ingress.yaml

- Expose your nginx controller to outside world as NodePort.

kubectl create -f nginx-ingress-controller-service.yaml -n=ingress


- Try accessing the sites.

curl -H "Host: test.akomljen.com" http://10.0.0.10:30000/app1
curl -H "Host: test.akomljen.com" http://10.0.0.10:30000/app2
curl -H "Host: test.akomljen.com" http://10.0.0.10:32000/nginx_status
