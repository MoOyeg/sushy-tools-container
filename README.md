# sushy-tools-container


This repo provides an example of how to run [Sushy Tools](https://opendev.org/openstack/sushy-tools) to emulate Redfish for KVM. Specifically for OpenShift BMH hosting.

- Create the Sushy Tools Image with Libvirt dependencies
```bash
sushy_image=$(buildah from registry.access.redhat.com/ubi8:latest)
buildah run $sushy_image -- /bin/bash -c 'dnf install -y git python3.12 python3.12-pip openssh-clients python3-pyOpenSSL pkgconfig python3.12-devel libvirt-devel libvirt-daemon-kvm libvirt-client virt-install && \
dnf groupinstall -y "Development Tools" && \
dnf clean all && \
rm -rf /var/cache/yum /var/cache/dnf'
buildah run $sushy_image -- git clone https://github.com/openstack/sushy-tools.git
buildah run $sushy_image -- /bin/bash -c 'mkdir /sushy-tools/.ssh && touch /sushy-tools/.ssh/config && \
echo "IdentityFile /sushy-tools/.ssh/id_rsa" >> /sushy-tools/.ssh/config && \
echo "PubKeyAuthentication yes" >> /sushy-tools/.ssh/config && \
echo "StrictHostKeyChecking no" >> /sushy-tools/.ssh/config'
buildah run $sushy_image -- pip3.12 install -r ./sushy-tools/requirements.txt
buildah run $sushy_image -- pip3.12 install gunicorn libvirt-python
buildah config --env SUSHY_EMULATOR_CONFIG=/sushy-tools/sushy-emulator.conf $sushy_image
buildah config --env GUNICORN_HOST="0.0.0.0" $sushy_image
buildah config --env GUNICORN_PORT="8000" $sushy_image
buildah config --env GUNICORN_WORKER_COUNT="3" $sushy_image
buildah config --env SERVER_CERT="/sushy-tools/server.crt" $sushy_image
buildah config --env SERVER_KEY="/sushy-tools/server.key" $sushy_image
buildah config --workingdir=/sushy-tools --entrypoint 'gunicorn -b ${GUNICORN_HOST}:${GUNICORN_PORT} --certfile=${SERVER_CERT} --keyfile=${SERVER_KEY} --workers=${GUNICORN_WORKER_COUNT} --log-level debug "sushy_tools.emulator.main:app"' $sushy_image
buildah commit $sushy_image sushy-image
```
    - Build command above will use gunicorn as webserver and take variables at runtime to control execution
    - $GUNICORN_HOST is the IP address application will listen on
    - $GUNICORN_PORT is the Port application will listen on
    - $GUNICORN_WORKER_COUNT amount of worker processes GUNICORN should listen with
    - $SERVER_CERT is the SSL server cert gunicorn will use
    - $SERVER_KEY is the SSL server cert gunicorn will use

- Container was built around the idea that the node running the emulation container might not be the same as the KVM Host.So we need to have SSH credentials that provide access to the KVM node.To this ssh copy an id to give the user that will run the emulator container access on the KVM host. User must have access to list,stop,start,delete and edit KVM domains on the KVM host.Example:

    ```bash
    ssh-copy-id -i ~/.ssh/mykeykvm user@kvm_host
    ```

- Create configuration file for sushy tools. An example is provided in [example-sushy.conf](./example-sushy.conf).
The important things to set
- SUSHY_EMULATOR_LIBVIRT_URI = qemu+ssh://root@kvm_host/system  #KVM host this emulator will connect to
- SUSHY_EMULATOR_AUTH_FILE = /sushy-conf/htpasswd #If auth is required create a htpasswd file
  e.g 
  ```bash
  htpasswd -nbB emulator emulator > /data/test-sushy/htpasswd
  ```
- SUSHY_EMULATOR_ALLOWED_INSTANCES = ['uuid_1','uuid_2'] #If you need to filter out the list of KVM domains this emulator will have access to.
  e.g To get Domain UUID's
  ```bash
  virsh dumpxml $domain | grep uuid
  ```

- For all other options please review [documentation](https://docs.openstack.org/sushy-tools/latest/user/dynamic-emulator.html)
    
- Create Emulator Container and Pass in SSH_KEY, Certs and Configuration
ssh_key="/home/user/.ssh/my_key"
conf_folder="/home/user/myfolder"
cert_folder="/home/user/mycerts"

podman run -d --replace \
--name emulator \
--userns=keep-id \
-v ${ssh_key}:/sushy-tools/.ssh/id_rsa:z \
-v ${conf_folder}:/sushy-conf:Z \
-v ${cert_folder}:/sushy-certs:Z \
-e SUSHY_EMULATOR_CONFIG="/sushy-conf/sushy-emulator.conf" \
-e SERVER_CERT="/sushy-certs/server.crt" \
-e SERVER_KEY="/sushy-certs/server.key" \
-e GUNICORN_HOST="0.0.0.0" \
-e GUNICORN_WORKER="1" \
-e GUNICORN_PORT=8002 \
-p 8002:8002 sushy-image:latest

- Confirm Sushy is running from Logs

- Check domain status
   
    - If Required Create a domain to test
    ```bash
    virt-install \
    --name vm_name \
    --ram 24000 \
    --disk size=100 \
    --vcpus 8 \
    --os-type linux \
    --os-variant rhel9.0 \
    --graphics vnc \
    --print-xml > tmpfile
    virsh define --file tmpfile
    rm tmpfile
    ```
    
    - Get UUID for domain in question
    ```bash
    virsh dumpxml $domain | grep uuid
    ```

    - Curl container Endpoint
    ```bash
    curl -k -X GET http://localhost:8002/redfish/v1/Systems
    ```

    - Curl container Endpoint with Authentication
    ```bash
    curl -k -u username:password -X GET http://localhost:8002/redfish/v1/Systems
    ```

    - Try PowerOn and PowerOff on VM
    ```bash
    curl -k -u -u username:password -d '{"ResetType":"On"}' \
    -H "Content-Type: application/json" -X POST \
     https://localhost:8002/redfish/v1/Systems/$uuid/Actions/ComputerSystem.Reset


    curl -k -u -u username:password -d '{"ResetType":"ForceOff"}' \
    -H "Content-Type: application/json" -X POST \
     https://localhost:8002/redfish/v1/Systems/$uuid/Actions/ComputerSystem.Reset
    ```

- An Example OpenShift BaremetalHost is provided in 
