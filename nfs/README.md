## NFS 설치 및 StorageClass 구성
- nfs-ganesha-server-and-external-provisioner(NFS 서버 기능 제공으로 보인다. k8s 클러스터 내부에 nfs을 제공하며 데이터도 노드에 저장하는 듯)
- nfs-subdir-external-provisioner
- csi-driver-nfs

## NFS 서버
```sh
sudo dnf install -y nfs-utils

# 방화벽 허용 및 부팅 시 NFS 서버 자동 시작
sudo systemctl enable --now nfs-server rpcbind
sudo firewall-cmd --add-service={nfs,nfs3,mountd,rpc-bind} --permanent 
sudo firewall-cmd --reload

# nfs 설정
vi /etc/exportfs
/share_name 192.168.1.0/24(rw,sync,no_subtree_check,no_root_squash)

# nfs 설정 적용
exportfs -ra
systemctl restart nfs-server
```

## NFS 클라이언트
```sh
dnf install -y nfs-utils
```

## NFS SC 설치
```sh
helm repo add nfs-subdir-external-provisioner https://kubernetes-sigs.github.io/nfs-subdir-external-provisioner/
helm repo update
helm install nfs-subdir-external-provisioner nfs-subdir-external-provisioner/nfs-subdir-external-provisioner \
  --set nfs.server=192.168.1.1 \
  --set nfs.path=/share_name/ \
  --set storageClass.defaultClass=true 
```

[공식 문서]  
https://docs.rockylinux.org/guides/file_sharing/nfsserver/  
https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/7/html/storage_administration_guide/nfs-serverconfig#nfs4-only

[NFS]  
https://github.com/kubernetes-csi/csi-driver-nfs?tab=readme-ov-file   
https://github.com/kubernetes-sigs/nfs-subdir-external-provisioner?tab=readme-ov-file  
https://github.com/kubernetes-sigs/nfs-ganesha-server-and-external-provisioner?tab=readme-ov-file  
