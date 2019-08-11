# Kubernentes

Material utilizado para gerar um cluster simples para utilização do kubernentes.

## Máquinas utilizadas

Segue abaixo o descritivo das máquinas utilizadas no cluster:

| Sistema operacional | usuário |        Sudo        |   IP Primário   | IP Secundário |
|:-------------------:|:-------:|:------------------:|:---------------:|:-------------:|
|      Centos 7       |   rke   | :white_check_mark: | 192.168.254.101 |   10.0.2.101  |
|      Centos 7       |   rke   | :white_check_mark: | 192.168.254.102 |   10.0.2.102  |
|      Centos 7       |   rke   | :white_check_mark: | 192.168.254.103 |   10.0.2.103  |

### Obsercações

- As máquinas devem trabalhar com comunicação irrestrita entre si.
- As máquinas conversam entre si usando o IP sencudário.
- Os IPs primários serão a porta de entrada dos usuários (ingress).
- O cliente de instalação deve ter chave SSH compartilhada com as máquinas acima.
- Na necessidade de configuração de firewall, utilizar controle de borda.


## Estrutura final

Finalizada a instalação, iremos ter a seguinte estrutura para nosso kubernetes:

|       Host      |    Control Plane   |        ETCd        |       Worker       |
|:---------------:|:------------------:|:------------------:|:------------------:|
| 192.168.254.101 | :white_check_mark: | :white_check_mark: | :white_check_mark: |
| 192.168.254.102 |         :x:        |         :x:        | :white_check_mark: |
| 192.168.254.103 |         :x:        |         :x:        | :white_check_mark: |

## Instalação de dependências

As dependências foram instaladas utilizando ansible. Após clocar o repositório, você pode fazer o mesmo processo usando o comando abaixo:

```bash
$ ansible all -m ping                       # Para validar a conexão com os hosts
$ ansible-galaxy install geerlingguy.ntp    # Para permitir a configuração de NTP nos servidores
$ ansible-playbook playbook.yml             # Para realizar a instalação das dependências
$ rke up                                    # Para usar o RKE para realizar as configurações
```

Com isso já temos nosso cluster configurado.

## Persistência de dados

Será utilizado Ceph para persistência de dados no cluster. Para configuração do mesmo, você deve usar os comandos abaixo:

```bash
$ kubectl --kubeconfig kube_config_cluster.yml apply -f rook/common.yml
$ kubectl --kubeconfig kube_config_cluster.yml apply -f rook/operator.yml
$ kubectl --kubeconfig kube_config_cluster.yml apply -f rook/cluster.yml
$ kubectl --kubeconfig kube_config_cluster.yml apply -f rook/storage.yml
```

### Dashboard

O Ceph possui um dashboard web que nos permite monitorar o status da operação. Vamos instalar o mesmo usando os comandos abaixo:

```bash
$ openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout tls.key -out tls.crt
$ kubectl --kubeconfig kube_config_cluster.yml \
    -n rook-ceph create secret tls tls-rookceph-ingress \
        --cert=tls.crt \
        --key=tls.key
$ kubectl --kubeconfig kube_config_cluster.yml create -f rook/dashboard.yml
```

#### Observações

- O ingress é criado com o domínio `ceph.kubernetes.local`. Edite o arquivo para mudar o mesmo.
- Para fazer testes locais aponte `ceph.kubernetes.local` para um dos nós no seu `/etc/hosts`.
- Para acessar o painel use o usuário `admin`. A senha é gerada de forma aleatória. Você pode descobrir a mesma usando o comando abaixo:

```bash
$ kubectl --kubeconfig kube_config_cluster.yml -n rook-ceph get secret rook-ceph-dashboard-password -o jsonpath="{['data']['password']}" | base64 --decode && echo
```

- Em caso de `500 internal server error` você deve executar os comandos abaixo:

```bash
$ kubectl --kubeconfig kube_config_cluster.yml create -f rook/tools.yml

# Usar o comando abaixo para verificar se o ceph-tools já está criado
$ kubectl --kubeconfig kube_config_cluster.yml -n rook-ceph get pod -l "app=rook-ceph-tools"

$ kubectl --kubeconfig kube_config_cluster.yml -n rook-ceph exec -it $(kubectl -n rook-ceph get pod -l "app=rook-ceph-tools" -o jsonpath='{.items[0].metadata.name}') bash
$ ceph dashboard ac-role-create admin-no-iscsi

$ for scope in dashboard-settings log rgw prometheus grafana nfs-ganesha manager hosts rbd-image config-opt rbd-mirroring cephfs user osd pool monitor; do 
    ceph dashboard ac-role-add-scope-perms admin-no-iscsi ${scope} create delete read update;
done

$ ceph dashboard ac-user-set-roles admin admin-no-iscsi
$ exit
$ kubectl --kubeconfig kube_config_cluster.yml -n rook-ceph delete deployment rook-ceph-tools
```

Para entender o porque dessa solução, acesse essa [issue](https://github.com/rook/rook/issues/3106).

## Interface WEB

Por fim, para gerênciamento iremos utilizadar rancher como interface web. Para instalar o mesmo, use os comandos a baixo:

```bash
$ kubectl --kubeconfig kube_config_cluster.yml -n kube-system create serviceaccount tiller
$ kubectl --kubeconfig kube_config_cluster.yml create clusterrolebinding tiller \
    --clusterrole=cluster-admin \
    --serviceaccount=kube-system:tiller
$ helm --kubeconfig kube_config_cluster.yml init --service-account tiller --upgrade
# O comando abaixo serve para aguardar o deploy do tiller
$ kubectl --kubeconfig kube_config_cluster.yml -n kube-system  rollout status deploy/tiller-deploy
$ helm --kubeconfig kube_config_cluster.yml repo add rancher-latest https://releases.rancher.com/server-charts/latest
$ helm --kubeconfig kube_config_cluster.yml install rancher-latest/rancher \
  --name rancher \
  --namespace cattle-system \
  --set hostname=rancher.kubernetes.local \
  --set ingress.tls.source=tls-rancher-ingress
```

### Observações

- A instalação foi feita utilizando `helm`. Você deve ter o mesmo instalado para fazer a instalação.
- Assim como o caso anterior, este ingress é criado com o domínio `rancher.kubernetes.local`. Edite o arquivo para mudar o mesmo.
- Para realizar testes locais, aponte o domínio `rancher.kubernetes.local` para um dos nós no seu `/etc/hosts`.

## Ansible

O playbook ansible utilizado faz as seguintes tarefas:

- Adiciona o epel ao centos.
- Instala o PIP nos servidores.
- Desabilita o uso de swap nos servidores.
- Instala o docker e suas dependências.
- Instala e configura o serviço de NTP.
- Habilita alguns módulos necessários no kernel.

## RKE

Dado o arquivo `cluster.yml`, nosso kubernetes é configurado nos nodes acima citados.