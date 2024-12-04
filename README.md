## Sanal Makinelerin Kurulumu
Makine kurulumları için gerekli VM'ler şunlardır:

Kurulum öncesinde private network'de çalışan aygıtlar var mı istenilen aralıktaki adreslerde kontrol edilmesi gerekmektedir.

```bash
ping 192.168.1.X
# Expected Output : Reply from 192.168.1.X: Destination host unreachable.
```

Biz yapacağımız bu kurulumda Ubuntu 24.04 kullanıyor olacağız kurulumda önemli olarak OpenSSH'ı dahil etmemiz ve Network Arayüzü üzerinden statik olarak ip adreslerini vermeliyiz.

__Her bir makineye 2 CPU ve 4GB RAM verilmesi önerilir.__
__RKE2 sistem kaynak gereksinimine [buradan](https://docs.rke2.io/install/requirements#vm-sizing-guide) bakabilirsiniz__




| VM Names      | VM IP    |
| ------------- |:-------------:|
| workplane     | 192.168.1.100/24 |
| controlplane (master node)      | 192.168.1.101/24      |
| worker01 (agent) |     192.168.1.102/24  |
| worker02 (agent) | 192.168.1.103/24 |

---
## Lablabs RKE2 GitHub Repo'sunun İndirilmesi
Verilen bu [link](https://github.com/lablabs/ansible-role-rke2 "Lablabs Ansible RKE2 Role") üzerinden lablabs/ansible-role-rke2 klonlayabilirsiniz. Bu çalışmayı workplane üzerinden yaparak cluster kurulumu yapacağız.

```bash
cd ~

git clone https://github.com/lablabs/ansible-role-rke2.git 

cd ansible-role-rke2
```

## Workplane üzerine Ansible Kurulumu
Ansible kurulumunun kullandığınız Ubuntu 24.04 ortamında tercih ettiğimiz 2 farklı şekilde kurulumu vardır bunlar:

* Python PIP ile kurulum - [link](https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html#installing-and-upgrading-ansible-with-pip)

* OS bazlı kurulum - [link](https://docs.ansible.com/ansible/latest/installation_guide/index.html)

Biz burada apt package manager ile kuruluma devam edeceğiz.

```bash
sudo apt update

sudo apt install ansible -y
```

Ansible'nin yönetilecek node'lar için ssh bağlantısını ise şu şekilde sağlayabiliriz:

* İlk olarak SSH Key oluşturulmalıdır.
```bash
ssh-keygen -t rsa -b 2048
```
Bu bize home dizini altında ~/.ssh public ve private key eşlerini oluşturacaktır.
* SSH anahtarımızı node'lara taşıyalım
```bash
for ip in 192.168.1.{101..103}; do
  ssh-copy-id -i ~/.ssh/id-rsa.pub <user>@$ip
done
```
Bu adımdan sonra ssh ile bağlanarak bağlantıyı test etmeyi unutmayın.

## Ansible Inventory ve Config Testi

Workplane'de rolü klonladığımız dizin altında inventory dosyamızı oluşturabiliriz. Inventory dosyasını ise sahip olduğumuz konfigürasyona göre şu şekilde tanımlayabiliriz:

```toml
[masters]
master-01 ansible_host=192.168.1.101

[workers]
worker-01 ansible_host=192.168.1.102
worker-02 ansible_host=192.168.1.103

[k8s_cluster:children]
masters
workers
```

Şimdi ad-hoc komut ile sağlıklı bir şekilde tanımladığımızı kontrol edebiliriz

```bash
ansible all -i inventory.ini -m ping
```

Sürekli olarak inventory dosya yolunu tanımlamak yerine bunları __ansible.cfg__ dosyasını oluşturarak default olarak kullanılan parametreleri burada tanımlayabiliriz.

```toml
[defaults]

inventory = ./inventory.ini
private_key_file = ~/.ssh/id_rsa
host_key_checking = False
```

Şimdi config dosyamızı test edebiliriz

```bash
ansible all -m ping
```

## Playbook Tanımlanması ve Çalıştırılması

İlk olarak yine klonladığımız dizinde bir install_rke2.yaml playbook oluşturarak ve bunları __lablabs/ansible-role-rke2__ ' de verilen parametreler ile birlikte playbook değişkenlerini tanımlayabiliriz ve oluşturabiliriz.

Burada one server(master) and several agent(worker) nodes şeklinde çalışacağımız için [bu şekilde](https://github.com/lablabs/ansible-role-rke2?tab=readme-ov-file#playbook-example) yazabilir ve değişkenleri ekleyebiliriz. Bizim kullandığımız örnek dosya ise şu şekildedir.

```yaml
---
- name: Deploy RKE2 Kubernetes Cluster
  hosts: all
  become: yes
  vars:
    rke2_ha_mode: false
    rke2_airgap_mode: false
    rke2_servers_group_name: masters
    rke2_agents_group_name: workers
    rke2_api_ip: "{{ hostvars[groups[rke2_servers_group_name].0]['ansible_default_ipv4']['address'] }}"
    rke2_download_kubeconf: true

  roles:
    - name: lablabs-rke2
      #/path/to/cloned/rke2
	  role_path: ./ 
```

Playbook çalıştırılarak cluster'ın ayağa kalkması sağlanır.

```bash
ansible-playbook install_rke2.yaml --ask-become-pass
```

> Bağlantı durumunda göre birkaç dakika sürebilir

## Master Node'da Kubectl Kurulumu ve Kubeconfig ayarlanması
Kurulan cluster'imizde kubectl api'si ve kubeconfig ayarları direkt olarak tanımlanmıyor bu yüzden master node'a bağlanarak bunların ayarlanmasını şu şekilde sağlayabiliriz:

```bash
ssh <user>@192.168.1.101

sudo snap install kubectl --classic

curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"

chmod +x kubectl

sudo mv kubectl /usr/local/bin/

kubectl version
```

## Kubeconfig Ayarlarının Yapılması

Kubeconfig file ayarlamaları için direkt kubectl çalıştırırsak hata alacağız. Bundan dolayı aşağıda Kubeconfig yapılandırması ve izinleri ile cluster'a sağlıklı bir şekilde erişebiliriz.

```bash
export KUBECONFIG=/etc/rancher/rke2/rke2.yaml

mkdir -p ~/.kube

cp /etc/rancher/rke2/rke2.yaml ~/.kube/config

#İzinler

sudo chown <user>:<user> ~/.kube/config

chmod 600 ~/.kube/config

kubectl get nodes
```

> __BAŞARILI !__ Yaptığımız adımları düzgün bir şekilde izledikten sonra cluster'imize erişebiliriz.





