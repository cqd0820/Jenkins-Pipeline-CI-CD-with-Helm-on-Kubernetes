# Jenkins-Pipeline-CI-CD-with-Helm-on-Kubernetes

The following implementation is how to use jenkins CI/CD pipeline in the Kubernetes orchestration with helm.

We can apply the code in Jenkins pipeline job to rollout a Hello-world nginx web instance.

Please proactive the prerequisit before rollout the pipeline.


Prerequisit:
```
Github

Jenkins 

Docker runner

Dockerhub

Kubernetes cluster

Helm
```


Apart from that, I put on a few solution of what I have encountered from my deployment provisioning.

1. Add jenkins user to docker group to fix the issue that Jenkins user can not access to /var/run/docker.sock

```Bash
usermod -a -G docker jenkins
```

2. Fix permission bugs while trying to use helm on pipeline.
```
Go to Jenkins --> Manage Jenkins --> Configure Global Security

Select Project-based Matrix Authorization Strategy under Authorization

Set permission for Anonymous User to Read / Write Jenkins Jobs. Check for overall Read should work in your case. You can also try other options.
```

3. Set Jenkins user kubectl env variable to get Jenkins user has the authorizition to use helm CMD. 
```Bash
mkdir -p /home/jenkins/.kube

cp -i /etc/kubernetes/admin.conf /home/jenkins/.kube/config

chown jenkins:jenkins /home/jenkins/.kube/config
```
![Jenkins Pipeline](https://github.com/showerlee/Jenkins-Pipeline-CI-CD-with-Helm-on-Kubernetes/blob/master/Jenkins/kube-helm-pipeline.png?raw=true "Jenkins Pipeline with Helm on Kubernetes")

Any details please check author's original repo: https://github.com/judexzhu/Jenkins-Pipeline-CI-CD-with-Helm-on-Kubernetes
