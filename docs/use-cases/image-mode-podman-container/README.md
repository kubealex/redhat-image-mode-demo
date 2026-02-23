# Use Case - Embedding a Containerized application into Image Mode

In this example, we want to show how to easily embed a containerized application into Image Mode, and let **bootc** handle its lifecycle.

To achieve this, we will link the application image as a [Logically Bound Image (LBI)](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/10/html/using_image_mode_for_rhel_to_build_deploy_and_manage_operating_systems/building-and-managing-logically-bound-images), that allows to link one or more specific container images to a bootc image.

We will have two Containerfiles, one for the application and one for the bootc image.

The application Containerfile will only add a custom index.html page to the Nginx base image:

<details>
  <summary>Review Containerfile.nginx</summary>
  ```dockerfile
  --8<-- "use-cases/image-mode-podman-container/Containerfile.nginx"
  ```
</details>


The bootc containerfile in this example will:

- Create the systemd unit for the container
- Link the container to bootc

<details>
  <summary>Review Containerfile</summary>
  ```dockerfile
  --8<-- "use-cases/image-mode-podman-container/Containerfile"
  ```
</details>


## Review the container unit

Together with the Containerfile, we can see a special file, *nginx.container*, that will take care of defining the Quadlet for our container.

The definition includes the ports to expose, the image to run and the restart policy.

<details>
  <summary>Review Containerfile.upgrade</summary>
  ```bash
  --8<-- "use-cases/image-mode-podman-container/nginx.container"
  ```
</details>

## Login to the container registry and fix the username


We can start by fixing the QUAY_USER variable with your [quay.io](quay.io) username:

```bash
export QUAY_USER=YOURQUAYUSERNAME
```

Log-in to Quay.io:

```bash
podman login -u $QUAY_USER quay.io
```

## Building and pushing the application image

From the root folder of the repository, switch to the use case directory:

```bash
cd use-cases/image-mode-podman-container
```

You can build the image right from the Containerfile using Podman:

```bash
podman build -f Containerfile.nginx -t quay.io/$QUAY_USER/nginx-lbi:v1.0 .
```

Push the image:

```bash
podman push quay.io/$QUAY_USER/nginx-lbi:v1.0
```

## Building the bootc image

From the root folder of the repository, switch to the use case directory:

```bash
cd use-cases/image-mode-podman-container
```

Before proceeding with the build, replace the QUAY_USER variable in the **nginx.container** file:

```bash
sed -i "s/QUAY_USER/$QUAY_USER/g' nginx.container
```

You can build the image right from the Containerfile using Podman:

```bash
podman build -f Containerfile -t quay.io/$QUAY_USER/rhel-image-mode:containerized-nginx .
```

And let's push this image too:

```bash
podman quay.io/$QUAY_USER/rhel-image-mode:containerized-nginx
```

## Convert to QCOW2 and create a Virtual Machine in KVM

Let's proceed with the QCOW image creation. During the conversion, the image will be made available to the target system, so that at boot it will be available and ready to be executed:

```bash
sudo podman run \
    --rm \
    -it \
    --privileged \
    --pull=newer \
    --security-opt label=type:unconfined_t \
    -v $(pwd)/output:/output \
    -v /var/lib/containers/storage:/var/lib/containers/storage \
    registry.redhat.io/rhel10/bootc-image-builder:latest \
    build \
    --type qcow2 \
    quay.io/$QUAY_USER/rhel-image-mode:containerized-nginx
```

We can then proceed with the creation of the VM

```bash
sudo virt-install \
    --name rhel-bootc-vm \
    --vcpus 4 \
    --memory 4096 \
    --import --disk ./output/qcow2/disk.qcow2,format=qcow2 \
    --os-variant rhel10.0 \
    --network network=default
```

## Retrieve the address and test the application

Now that the VM is in place, we only need the IP address to test it out.
As we exposed the application container *port 80* to the host *port 8080*, we will be able to just jump in a browser and verify that the page is correctly showing!

To retrieve the IP:

```bash
virsh domifaddr rhel-bootc-vm | awk '/ipv4/ {print $4}' | cut -d/ -f1 | head -n1
```

In a browser, go to http://{VM_IP}:8080 to verify that everything is working as expected!

![](./assets/browser.png)

## Updating the application container image

To update the application container image, it will be enough to:

- Create a new version of our application image - i.e. quay.io/$QUAY_USER/nginx-lbi:v2.0
- Reference the new version in the nginx.container file
- Rebuild the Image Mode container image
- Run the **bootc upgrade** command on the target host
