# cronjob-to-delete-the-dangling-pvc-inside-the-kubernets
cronjob-to-delete-the-dangling-pvc-inside-the-kubernets
------craete cjdemo 
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: all-permissions
rules:
- apiGroups: ["*"]
  resources: ["*"]
  verbs: ["*"]

---

apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: all-permissions-binding
subjects:
- kind: ServiceAccount
  name: all-permissions-sa
  namespace: default
roleRef:
  kind: ClusterRole
  name: all-permissions
  apiGroup: rbac.authorization.k8s.io

---

apiVersion: v1
kind: ServiceAccount
metadata:
  name: all-permissions-sa
  namespace: default

---

apiVersion: batch/v1
kind: CronJob
metadata:
  name: hello
spec:
  schedule: "*/1 * * * *"
  jobTemplate:
    spec:
      template:
        spec:
          initContainers:
          - name: change-permissions
            image: busybox
            command: ["/bin/sh", "-c"]
            args:
            - "cp /scripts/delete-dangling-pvcs.sh /etc/pre-install/ && cd /etc/pre-install && chmod +x delete-dangling-pvcs.sh && ls -l && ./delete-dangling-pvcs.sh"
            volumeMounts:
            - name: script-volume
              mountPath: /scripts
            - name: pre-install
              mountPath: /etc/pre-install
          securityContext:
            runAsUser: 0
          serviceAccountName: all-permissions-sa
          containers:
          - name: hello
            image: bitnami/kubectl:latest
            command: ["/bin/sh", "-c"]
            args:
            - "/scripts/delete-dangling-pvcs.sh"
            volumeMounts:
            - name: script-volume
              mountPath: /scripts
              readOnly: false
          restartPolicy: OnFailure
          volumes:
          - name: script-volume
            configMap:
              name: my-script
              defaultMode: 0777
          - name: pre-install
            emptyDir: {}

2) Now craete configmap.yaml
   ........apiVersion: v1
kind: ConfigMap
metadata:
  name: my-script
data:
  delete-dangling-pvcs.sh: |-
    #!/bin/sh

    # Get a list of all PVCs in the cluster
      #pvc_list=$(kubectl get pvc --all-namespaces -o jsonpath='{range .items[?(@.status.phase=="Bound")]}{.metadata.namespace}/{.metadata.name}{"\n"}{end}')
      pvc_list=$(kubectl get pvc --all-namespaces -o jsonpath='{range .items[?(@.metadata.labels.user=="scalegrid")]}{.metadata.namespace}/{.metadata.name}{";"}{.metadata.labels.app}{"\n"}{end}' | grep "postgresql-db$" | awk -F ';' '{print $1}')

      # Iterate over the PVCs
      for pvc in $pvc_list; do
          namespace=$(echo "$pvc" | cut -d'/' -f1)
          pvc_name=$(echo "$pvc" | cut -d'/' -f2)

          # Check if the PVC has any associated PVs
          pv_list=$(kubectl describe pvc "$pvc_name" -n "$namespace" | grep -i "Used By:")

          # If no associated PV found, delete the PVC
          if echo "$pv_list" | grep -qi "none"; then
              echo "Deleting PVC: $pvc"
              kubectl delete pvc "$pvc_name" -n "$namespace"
          fi
      done
.......
3)this is an script for bin/sh
#!/bin/sh

# Get a list of all PVCs in the cluster
#pvc_list=$(kubectl get pvc --all-namespaces -o jsonpath='{range .items[?(@.status.phase=="Bound")]}{.metadata.namespace}/{.metadata.name}{"\n"}{end}')
pvc_list=$(kubectl get pvc --all-namespaces -o jsonpath='{range .items[?(@.metadata.labels.user=="scalegrid")]}{.metadata.namespace}/{.metadata.name}{";"}{.metadata.labels.app}{"\n"}{end}' | grep "postgresql-db$" | awk -F ';' '{print $1}')

# Iterate over the PVCs
for pvc in $pvc_list; do
    namespace=$(echo "$pvc" | cut -d'/' -f1)
    pvc_name=$(echo "$pvc" | cut -d'/' -f2)

    # Check if the PVC has any associated PVs
    pv_list=$(kubectl describe pvc "$pvc_name" -n "$namespace" | grep -i "Used By:")

    # If no associated PV found, delete the PVC
    if echo "$pv_list" | grep -qi "none"; then
        echo "Deleting PVC: $pvc"
        #kubectl delete pvc "$pvc_name" -n "$namespace"
    fi
done
..
4)and the statefulset for the refrence 
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgresql-db
spec:     
  replicas: 2
  selector:
    matchLabels:
      app: postgresql-db
  template:
    metadata:
      labels:
        app: postgresql-db
    spec:
      containers:
        - name: postgresql-db
          image: postgres:latest
          volumeMounts:
            - name: postgresql-db-disk
              mountPath: /data
              readOnly: false
          env:
            - name: POSTGRES_PASSWORD
              value: testpassword
            - name: PGDATA
              value: /data/pgdata
          resources:
            requests:
              cpu: "100m"
              memory: "50Mi"    
      volumes:
        - name: postgresql-db-disk
          emptyDir: {}
  volumeClaimTemplates:
    - metadata:
        name: postgresql-db-disk
        labels:
          user: scalegrid     
      spec:
        accessModes:
          - ReadWriteMany
        resources:
          requests:
            storage: 1Gi

            
......
5)thius run an cronjob and delete dangling pvc after every 1 min 
6)command required 
kubectl apply -f statefulset.yaml
kubetl apply -f cronjob.yaml
kubectl apply -f configmap.yaml
kubectl scale statefulset postgresdb-sql --replicas=3   and then 2
kubectl get pod 
kubectl get statefulset 
kubectl get cj 
kubectl get pvc --show-labels
kubectl logs podname 
./script.sh
kubectl delete -f statefulset ,cronjob,configmap  etc
this is for k8s below 1.27 version 
the images use have kubernet is must so we use bitnami/kubectl
also alpine/k8s:1.2222 version have both awscli and k8s 
init container is used to cahnege the permissions also cronjob have default 0600 access 
also we need to change access mode into read write many if multiple containers at same time reqires the access 
service account required to allow the permissions and also for that we required role and role binding to select permissions 
