# standalonekubelet
Two Options to get up and running 1. Use a pre-built image or 2. use vanilla FCOS image and apply configs to it. Both options are described below. Using the pre-built image will get you up and running the fastest.


# Use pre-built image


Pull ISO from: https://github.com/petebowden/fedora-coreos-config/releases/latest/ or use instructions below to build.


Modify config.fcc to include your pub key. config.fcc is used for the installation of fcos.

`podman run -i --rm quay.io/coreos/fcct:release --pretty --strict < config.fcc > config.ign`

Copy the contents of config.ign inline into iso-install.fcc in the storage-> files-> path: /home/core/config.ign" section. iso-install.fcc is used for the boot of the installer ISO. Optionally you can add your pub key to this fcc if you need to log in to troubleshoot the install.

Convert `iso-install.fcc` to `iso-install.ign`:

`podman run -i --rm quay.io/coreos/fcct:release --pretty --strict < iso-install.fcc > iso-install.ign`


Embed `iso-install.ign` into installer:

`podman run --privileged --pull=always --rm -v .:/data -w /data quay.io/coreos/coreos-installer:release iso ignition embed -i /data/iso-install.ign fedora-coreos-33.20210302.dev.4-live.x86_64.iso`

Start VM with configured ISO:

`sudo virt-install --name fcos --ram 4500 --vcpus 2 --disk size=20,bus=virtio,format=qcow2 --accelerate --cdrom /path/to/ffedora-coreos-33.20210302.dev.4-live.x86_64.iso --network default`

Once install is finished the VM will reboot, at which point you can add a podspec to `/etc/kubernetes/manifest`. There is a sample (echoserver.yaml) podspec attached.
## Build your own image process
Pre-built image was created using coreos-assembler with modified configs. Configs can be found at: https://github.com/petebowden/fedora-coreos-config To build this image yourself, follow the instructions for [building an image with coreos-assembler](https://coreos.github.io/coreos-assembler/building-fcos/), but pull the custom configs instead of the defaults at the "Initializing Step". After the build step, to generate a live ISO execute: `cosa buildextend-metal4k` and then `cosa buildextend-live`. This will result in a bootable ISO.

# Manual configuration

Tested on F32 - mostly copied from FCOS docs
download fcos image

`podman pull quay.io/coreos/fcct:release`

Modify fcc to include your pub key

convert fcc to ign

`podman run -i --rm quay.io/coreos/fcct:release --pretty --strict < iso-install.fcc > iso-install.ign`

pull latest image

`podman run --privileged --pull=always --rm -v .:/data -w /data quay.io/coreos/coreos-installer:release download -f iso`

embed ign into installer

`podman run --privileged --pull=always --rm -v .:/data -w /data quay.io/coreos/coreos-installer:release iso ignition embed -i /data/iso-install.ign fedora-coreos-33.20210201.3.0-live.x86_64.iso`

`sudo virt-install --name cdrom --ram 4500 --vcpus 2 --disk size=20,bus=virtio,format=qcow2 --accelerate --cdrom /path/to/fedora-coreos-33.20210201.3.0-live.x86_64.iso --network default`

Container should boot, installer should run and then reboot.
Manual steps (for now - goal is kubelet should run in container).

ssh to node core@\<ip\>

```
sed -i -z s/enabled=0/enabled=1/ /etc/yum.repos.d/fedora-modular.repo
sed -i -z s/enabled=0/enabled=1/ /etc/yum.repos.d/fedora-updates-modular.repo`

rpm-ostree install cri-o kubernetes-node cri-tools

```
Need to reboot to have access to newly installed binaries

```
systemctl reboot
```
log back in

```
systemctl mask docker

cat <<EOF > /etc/kubernetes/kubelet.conf
kind: KubeletConfiguration
apiVersion: kubelet.config.k8s.io/v1beta1
cgroupDriver: systemd
cgroupRoot: /
staticPodPath: /etc/kubernetes/manifests
systemCgroups: /system.slice
authentication:
  anonymous:
    enabled: true
  webhook:
    enabled: false
authorization:
  mode: AlwaysAllow
EOF

cat <<EOF > /etc/systemd/system/kubelet.service
[Unit]
Description=Kubernetes Kubelet Server
Documentation=https://kubernetes.io/docs/concepts/overview/components/#kubelet https://kubernetes.io/docs/reference/generated/kubelet/
After=crio.service
Requires=crio.service

[Service]
WorkingDirectory=/var/lib/kubelet
EnvironmentFile=-/etc/kubernetes/config
EnvironmentFile=-/etc/kubernetes/kubelet
ExecStart=/usr/bin/kubelet \
           --config=/etc/kubernetes/kubelet.conf \
           --cgroup-driver=systemd \
           --fail-swap-on=false \
           --container-runtime=remote \
           --container-runtime-endpoint=unix:///var/run/crio/crio.sock \
           --runtime-cgroups=/system.slice/crio.service \
           --runtime-request-timeout=10m \
           --address=127.0.0.1 \
           --hostname-override=127.0.0.1 \
           --logtostderr=true 

Restart=on-failure
KillMode=process

[Install]
WantedBy=multi-user.target
EOF

rm /etc/cni/net.d/*

cat <<EOF > /etc/cni/net.d/100-crio-bridge.conflist
{
  "cniVersion": "0.4.0",
  "name": "bridge-firewalld",
  "plugins": [
    {
      "type": "bridge",
      "bridge": "cni0",
      "isDefaultGateway": true,
      "isGateway": true,
      "ipMasq": true,
      "ipam": {
        "type": "host-local",
        "subnet": "10.88.0.0/16",
        "routes": [
          {
            "dst": "0.0.0.0/0"
          }
        ]
      }
    },
    { 
      "type": "portmap",
      "capabilities": {
	"portMappings": true
     }
    },
    {
      "type": "firewall",
      "backend":"iptables"
    }
  ]
}
EOF

systemctl enable kubelet --now
```
Once this is up you can add a podspec to /etc/kubernetes/manifest There is a sample podspec attached.


Thank you to dmesser and his tutorial at: https://gist.github.com/dmesser/ffa556788660a7d23999427be4797d38 
