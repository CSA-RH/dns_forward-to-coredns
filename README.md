# dns_forward-to-coredns

## Introduction
This repository was based on the work of the following link and removes the Template requirement to create the K8s objects.
https://github.com/openshift-cs/OpenShift-Troubleshooting-Templates/tree/master/efs-dns-config

## Procedure

1. **Deploy coredns**:

   - Export vars

     ```
     export DOMAIN=efs.us-west-2.amazonaws.com
     export NAME=coredns           #Name K8s
     export RESOURCE=fs-df3e4fda   #EFS FilesystemID
     export IP_ADDRESS=30.3.3.3    # EFS Mount IP
     ```

   - Create Namespace
     
     ```
     oc new-project mycoredns
     ```

   - Create ConfigMap
     
     ```
     cat <<EOF > configmap.yaml
     kind: ConfigMap
     apiVersion: v1
     metadata:
       name: "${NAME}-config"
       labels:
         app: "${NAME}"
     data:
       Corefile: |
         ${DOMAIN}:8053 {
             errors
             health
             file /etc/coredns/${DOMAIN}
             cache 30
             reload
         }
       ${DOMAIN}: |
         \$TTL    1800
         \$ORIGIN ${DOMAIN}.

         @ IN SOA dns domains (
             2020031101   ; serial
             300          ; refresh
             1800         ; retry
             14400        ; expire
             300 )        ; minimum

         ${RESOURCE}        IN  A  ${IP_ADDRESS}
     EOF

     oc create -f configmap.yaml    


   - Create Deployment

     ```
     cat <<EOF > deployment.yaml
     kind: Deployment
     apiVersion: apps/v1
     metadata:
       name: ${NAME}
       labels:
         app: ${NAME}
     spec:
       replicas: 1
       selector:
         matchLabels:
           app: ${NAME}
           deployment: ${NAME}
       template:
         metadata:
           labels:
             app: ${NAME}
             deployment: ${NAME}
         spec:
           containers:
           - name: ${NAME}
             image: quay.io/openshift/origin-coredns:4.15
             command:
               - "/usr/bin/coredns"
             args:
               - "-dns.port"
               - "8053"
               - "-conf"
               - "/etc/coredns/Corefile"
             volumeMounts:
               - mountPath: /etc/coredns
                 name: "${NAME}-config"
             ports:
               - containerPort: 8053
             readinessProbe:
               timeoutSeconds: 10
               initialDelaySeconds: 10
               httpGet:
                 path: "/health"
                 port: 8080
                 scheme: "HTTP"
             livenessProbe:
               timeoutSeconds: 10
               initialDelaySeconds: 10
               httpGet:
                 path: "/health"
                 port: 8080
                 scheme: "HTTP"
             resources:
               requests:
                 cpu: "100m"
                 memory: "70Mi"
               limits:
                 memory: "256Mi"
           volumes:
           - name: "${NAME}-config"
             configMap:
               defaultMode: 420
               name: "${NAME}-config"
     EOF

     oc create -f deployment.yaml
     ```

   - Create Service

     ```
     cat <<EOF > service.yaml
     apiVersion: v1
     kind: Service
     metadata:
       labels:
         app: "${NAME}"
       name: "${NAME}"
     spec:
       ports:
       - name: 53-tcp
         port: 53
         protocol: TCP
         targetPort: 8053
       - name: 53-udp
         port: 53
         protocol: UDP
         targetPort: 8053
       selector:
         app: "${NAME}"
         deployment: "${NAME}"
     EOF

     oc create -f service.yaml
     ``` 

2. **Configure OpenShift DNS Operator Forwarding**
    1. Get the ClusterIP of your new service and set the EFS Domain
        ```$bash
        export SERVICE_IP=$(oc get service coredns -o custom-columns=CLUSTER-IP:.spec.clusterIP --no-headers)
        export FORWARD_DOMAIN=efs.us-west-2.amazonaws.com
        ```

    2. Configure the DNS operator
        - If you don't have any DNS forwarding currently in use, you must run this first
            ```$bash
            oc patch dns.operator default --type=json --patch='[{"op": "add", "path": "/spec", "value": {"servers": []}}]'
            ```

        - Configure DNS Forwarding
            ```$bash
            oc patch dns.operator default --type=json --patch="[{\"op\": \"add\", \"path\": \"/spec/servers/-1\", \"value\": {\"name\": \"upstream-dns\", \"zones\": [\"$FORWARD_DOMAIN\"], \"forwardPlugin\": {\"upstreams\": [\"$SERVICE_IP\"]}}}]"
            ```

2. **Test**
    - Connect to a random Pod and rosolve the EFS FQDN

      ```$bash
      #Connect to a Pod in the SDN and resolve the DNS FQDN
      oc -n project-test rsh postgresql-1-fzlld

      sh-4.4$ 
      sh-4.4$ nslookup fs-df3e4fda.efs.us-west-2.amazonaws.com
      Server:		172.30.0.10
      Address:	172.30.0.10#53

      Name:	fs-df3e4fda.efs.us-west-2.amazonaws.com
      Address: 30.3.3.3
      ```

3. **Things to keep in mind about the solution described**
  - Recreating or creating new EFS mount points will change the EFS IP.
  - The customized DNS server must be deployed in each cluster.
