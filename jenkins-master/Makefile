service = jenkins-svc
namespace = jenkinsmaster

kubectl := kubectl -n $(namespace)
aws := aws --region $(region)


default: help

help:
	@make -qp | awk -F':' '/^[a-z0-9][^$$#\/\t=]*:([^=]|$$)/ {split($$1,A,/ /);for(i in A)print A[i]}' | sort


namespace:
	@cat k8s/namespace.yaml | sed "s|NAMESPACE|$(namespace)|g" | kubectl apply -f -


deploy: namespace
	@cat k8s/jenkins.yaml | sed "s|NAMESPACE|$(namespace)|g" | kubectl apply -f -


stop:
	@cat k8s/jenkins.yaml | sed "s|NAMESPACE|$(namespace)|g" | kubectl delete -f -
