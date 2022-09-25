# The Certified Kubernetes Application Developer

https://training.linuxfoundation.org/certification/certified-kubernetes-application-developer-ckad/ is a great certification to train and proove your kubernetes skills. For approximately $600 USD you can get the course and exam. The exam is a 2 hours hands-on exercise where you can proove that you can deploy, scale and debug applications on kubernetes. I took the exam in May 2022 and passed first try. Here are a few tips that can help you.

### Essential setup

Speed is essential. It is very important that if you can't write the commands on top of your head, that you have a quick way of performing certain actions.

Take a minute or two to setup your environment properly before starting with the first question. Bookmark the kubernetes cheat sheet and open the page as soon as the exam starts: https://kubernetes.io/docs/reference/kubectl/cheatsheet/. Copy and paste the following commands. Those three commands allow you to use the k alias for kubectl and have bash autocompletion activated for this alias. You can then type for example `k get po a<tab>` and it will autocomplete the pods name starting with a.

```bash
source <(kubectl completion bash)
alias k=kubectl
complete -F __start_kubectl k
```
I was reading a lot of articles where they suggested to set the following alias to quickly switch beween different namespaces:

```bash
alias kn='kubectl config set-context --current --namespace'
```

I was always using this alias during the labs, but when the exam approached I realised that you sometimes need to save a command to a file. Since I was afraid the command can't be just ran as is, when I set the namespace permanently I decided to ditch the alias for the exam and always specify it explicitly.

Next setup your vim properly.

```bash
vim .vimrc
```
and add the following three lines:

```bash
set tabstop=2
set expandtab
set shiftwidth=2
```
With these set you will be able to modify and add to the manifests without getting syntax errors because your yaml files are malformed.

Then set two more environment variables which will make your life much easier.

```bash
export now='--grace-period 0 --force'
```
You can use it to quickly delete a resource without waiting for kubernetes for gracefully shut it down. For example: `k delete po test $now`. This will immediately terminate the pod and save your valuable time.

This next variable is maybe the most important one to set of all the tips above:

```bash
export do='--dry-run=client -o yaml'
```
You should use it anytime you want to create a manifest from the command line. So insted of doing

```bash
k run test-pod --image=nginx
k get po test-pod -o yaml > test-pod.yaml
k delete po test-pod $now
```
you can simple use the do variable to have a manifest:

```bash
k run test-pod --image=nginx $do
```

This will save you a huge amount of time.

### Always use the command line and only check the docs if really needed

Try to use the command line to create resources whenever possible. You will be much faster using the cli instead of searching the documentation. I think I pretty much completed the whole exam without using the documentation. Keep in mind that you can always use the -h flag to get useful examples.

### Practise exams

I scheduled two practise exams from killer shell (https://killer.sh/) and found it very useful to practise for the real exam. The setup of the practise exams is very close to the real one and you can learn how you perform under time pressure. The practise exams are a bit harder than the real one, so if you are able to get a good score in the practise exam you should be ready to rock the real one! Have fun.




