
<!-- ABOUT THE PROJECT -->

# Feito Com

Abaixo segue o que foi utilizado na criação deste projeto:

- [Minikube](https://minikube.sigs.k8s.io/docs/start/) - Ferramenta de código aberto que permite criar um ambiente de teste do Kubernetes em sua máquina local. Com o Minikube, é possível criar e implantar aplicativos em um cluster Kubernetes em sua máquina local.
- [Helm](https://helm.sh/) - Ferramenta de gerenciamento de pacotes de código aberto para o Kubernetes. O Helm permite empacotar aplicativos Kubernetes em um formato padrão chamado de gráfico, que inclui todos os recursos necessários para implantar o aplicativo, incluindo configurações e dependências.
- [ArgoCD](https://argo-cd.readthedocs.io/en/stable/) - Ferramenta declarativa que usa a abordagem GitOps para implantar aplicações no Kubernetes. O Argo CD é gratuito, tem código aberto, é um projeto incubado pela CNCF, e possui uma interface web de visualização e gerenciamento dos recursos, mas também pode ser configurado via linha de comando.
- [Spark](https://spark.apache.org/) - O Spark é um framework de processamento de dados distribuído e de código aberto, que permite executar processamento de dados em larga escala, incluindo processamento em batch, streaming, SQL, machine learning e processamento de gráficos. Ele foi projetado para ser executado em clusters de computadores e fornece uma interface de programação fácil de usar para desenvolvedores;
- [Airflow](https://airflow.apache.org/) - O Airflow é uma plataforma de orquestração de fluxo de trabalho de dados de código aberto que permite criar, agendar e monitorar fluxos de trabalho complexos de processamento de dados. Ele usa uma linguagem de definição de fluxo de trabalho baseada em Python e possui uma ampla gama de conectores pré-construídos para trabalhar com diferentes sistemas de armazenamento de dados, bancos de dados e ferramentas de processamento de dados;
- [Reflector](https://github.com/emberstack/kubernetes-reflector) - O Reflector é uma ferramenta de sincronização de estado de código aberto que permite sincronizar recursos Kubernetes em diferentes clusters ou namespaces. Ele usa a abordagem de controlador de reconciliação para monitorar e atualizar automaticamente o estado dos recursos Kubernetes com base em um estado desejado especificado;
- [Minio](https://min.io/) - O Minio é um sistema de armazenamento de objetos de código aberto e de alta performance, compatível com a API Amazon S3. Ele é projetado para ser executado em clusters distribuídos e escaláveis e fornece recursos avançados de segurança e gerenciamento de dados;
- [Postgres](https://www.postgresql.org/) - O Postgres é um sistema de gerenciamento de banco de dados relacional de código aberto, conhecido por sua confiabilidade, escalabilidade e recursos avançados de segurança. Ele é compatível com SQL e é usado em uma ampla gama de aplicativos, desde pequenos sites até grandes empresas e organizações governamentais.

<!-- GETTING STARTED -->

# Começando

Antes de executar o pipeline de dados, é necessário instalar algumas aplicações que serão responsáveis por manter e gerenciar o processo. Este guia apresentará uma série de comandos para instalar as ferramentas necessárias.

## Pré-requisitos

Antes de prosseguir com a configuração e uso da solução, você precisará fazer um **`fork`** deste projeto e configurar um ambiente de desenvolvimento local para criar, testar e executar o projeto e ter uma chave ssh configurada em seu computador. Para isso, siga o guia abaixo:

### instalação do cluster

O primeiro passo é montar um ambiente com um cluster Kubernetes local para executar a aplicação e o pipeline de dados. Para esta POC, usaremos o cluster de Kubernetes **[minikube](https://minikube.sigs.k8s.io/docs/)**. [Siga este guia de instalação para instalar o Minikube](https://minikube.sigs.k8s.io/docs/start/).

Também usaremos o **[helm](https://helm.sh/)** para nos ajudar a instalar algumas aplicações. [Siga este guia de instalação para instalar o Helm](https://helm.sh/docs/intro/install/).

Após instalar esses pré-requisitos, é hora de iniciar o nosso cluster Minikube. Para que tudo ocorra bem, é aconselhável usar um cluster de no mínimo 8GB de memória e 2 CPUs. Execute o seguinte comando no terminal:

```
minikube start --memory=8000 --cpus=2
```

Para acessar alguns serviços via loadbalancer no Minikube, é necessário utilizar o [tunelamento do minikube](https://minikube.sigs.k8s.io/docs/handbook/accessing/#example-of-loadbalancer). Para isso, abra uma nova aba no seu terminal e execute o seguinte comando:
```sh
minikube tunnel
```

### Link entre cluster k8 minikube (WSl2) e windows lens. 

minikube update-context -> update minikube .kube/config file

cd ~ -> cd .kube -> cat config

get server: {n°}

```sh
netsh advfirewall firewall add rule name="Allow Minikube {n°}" dir=in action=allow protocol=TCP localport={n°}
```

```sh
sudo apt-get update
sudo apt-get install ufw

sudo ufw allow 32700:32799/tcp
sudo ufw enable
```

now trade the paths of your config: home\username\ -> \\wsl$\Ubuntu\home\....

Then add the config file copy to lens.

Com isso, o seu ambiente estará pronto para receber acessos via loadbalancer.


## Instalação das ferramentas

Depois do ambiente inicializado será necessario instalar algumas aplicações que serão responsaveis por manter e gerenciar nosso pipeline de dados.

Estando conectado em um cluster Kubernetes, execute os seguintes comandos para criar todos os namespaces necessarios:

```sh
kubectl create namespace orchestrator
kubectl create namespace database
kubectl create namespace ingestion
kubectl create namespace processing
kubectl create namespace datastore
kubectl create namespace deepstorage
kubectl create namespace cicd
kubectl create namespace app
kubectl create namespace management
kubectl create namespace misc
```

Instale o argocd que será responsavel por manter nossas aplicações:
```sh
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update
helm install argocd argo/argo-cd --namespace cicd --version 5.27.1
```

Altere o service do argo para loadbalancer:
```sh
# create a load balancer
kubectl patch svc argocd-server -n cicd -p '{"spec": {"type": "LoadBalancer"}}'
```

Em seguida instale o argo cli para fazer a configuração do repositorio:
```sh
sudo curl -sSL -o /usr/local/bin/argocd https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
sudo chmod +x /usr/local/bin/argocd
```
Em seguida armazene o ip atribiudo para acessar o argo e faça o login no argo, com os seguintes comandos:
```sh
ARGOCD_LB=$(kubectl get services -n cicd -l app.kubernetes.io/name=argocd-server,app.kubernetes.io/instance=argocd -o jsonpath="{.items[0].status.loadBalancer.ingress[0].ip}")

# get password to log into argocd portal
# argocd login 192.168.0.200 --username admin --password UbV0FdJ2ZNCD8kxU --insecure
kubectl get secret argocd-initial-admin-secret -n cicd -o jsonpath="{.data.password}" | base64 -d | xargs -t -I {} argocd login $ARGOCD_LB --username admin --password {} --insecure
```
> [!WARNING] 
Here, the tunnel kernel will probably ask for sudo permission.



Uma vez feita a autenticação não é necessario adicionar um cluster, pois o argo esta configurado para usar o cluster em que ele esta instalado, ou seja, o cluster local ja esta adicionado como **`--in-cluster`**, bastando apenas adicionar o seu repositorio com o seguinte comando:

```sh

argocd repo add git@github.com:Gabriel-Philot/{repo-path}.git --ssh-private-key-path ~/.ssh/{path-private-ssh-key-on computer} --insecure-skip-server-verification

```

> caso queira ver o password do argo para acessar a interface web execute este comando: `kubectl get secret argocd-initial-admin-secret -n cicd -o jsonpath="{.data.password}" | base64 -d`


> Lembrando que para este comando funcionar é necessario que você tenha uma `chave ssh` configurada para se conectar com o github no seu computador.

Para acessar o argocd pelo IP gerado no Loadbalancer execute o comando:

```sh
echo http://$ARGOCD_LB
```

Uma vez que tenha acessado a pagina de autenticação do argocd use o `username` admin e o password gerado na instalação do argocd, executando o comando:

```sh
kubectl get secret argocd-initial-admin-secret -n cicd -o jsonpath="{.data.password}" | base64 -d
```

## Aqui o reflector armazena as secrets basicamente e distribui entre diferentes namespaces

```sh
kubectl apply -f manifests/management/reflector.yaml
```

Antes de executar os comandos, você pode alterar os secrets dos arquivos localizados na pasta `secrets/` se quiser mudar as senhas de acesso aos bancos de dados e ao storage.

Após o Reflector estar funcionando, execute o comando que cria os secrets nos namespaces necessários:

### alterar github -> manifests/misc/secrets.yaml
```sh
# secrets
kubectl apply -f manifests/misc/secrets.yaml
```

> Caso não queira instalar o Reflactor para automatizar o processo de criar o secret em vários namespaces diferentes, você pode replicar manualmente o secret para outro namespace executando este comando:
`kubectl get secret minio-secrets -n deepstorage -o yaml | sed s/"namespace: deepstorage"/"namespace: processing"/| kubectl apply -n processing -f -`

Uma vez que os secrets estejam configurados, é possível instalar os bancos de dados e o storage do pipeline de dados com o seguinte comando:

```sh
# databases
kubectl apply -f manifests/database/postgres.yaml
# deep storage
kubectl apply -f manifests/deepstorage/minio.yaml
```

Por fim, instale o Spark e o Airflow, juntamente com suas permissões para executar os processos do Spark, executando os seguintes comandos:

```sh
# add & update helm list repos
helm repo add spark-operator https://kubeflow.github.io/spark-operator
helm repo add apache-airflow https://airflow.apache.org/
helm repo update
```

```sh
# processing
kubectl apply -f manifests/processing/spark.yaml
```
<!-- Para criar um imagem do airflow com algumas libs inclusas, para isto execute o seguinte comando:
```sh 
eval $(minikube docker-env)
docker build -f images/airflow/dockerfile images/airflow/ -t airflow:0.1
``` -->

Antes de instalar o Airflow, é preciso atender a um requisito: criar um secret contendo sua `chave ssh`, para que o Airflow possa baixar as `DAGs` necessárias por meio do `gitSync`. É possível criar esse secret com o seguinte comando:

> Lembrando que você deve ter a `chave ssh` configurada em sua máquina.

> .ssh/{path-private-ssh-key-on-computer}


### change github -> orchestrator/airflow.yaml repo
```sh
kubectl create secret generic airflow-ssh-secret --from-file=gitSshKey=$HOME/.ssh/{path-private-ssh-key-on-computer} -n orchestrator
```

```sh
# orchestrator
kubectl apply -f manifests/orchestrator/airflow.yaml
```

Em seguida, instale as configurações de acesso:
### change github -> anifests/misc/access-control.yaml repo

```sh
kubectl apply -f manifests/misc/access-control.yaml
```

Para que seja possivel o Ariflow executar de maneira independente os processos spark é preciso que ele tenha uma conexão com o cluster, e para isto é necessario passar essa informação ao Airflow. Para adicionar a conexão com o cluster ao Airflow execute:

>[!NOTE] 
Here we can change the values of the config.json (images/airflow/connections.json) but also need to change it in the secrets (minio-secrets.yaml / postgress-secrets.yaml). Carefull with the basecode 64 encryption.

```sh
kubectl get pods --no-headers -o custom-columns=":metadata.name" -n orchestrator | grep scheduler | xargs -i sh -c 'kubectl cp images/airflow/connections.json orchestrator/{}:./ -c scheduler | kubectl -n orchestrator exec {} -- airflow connections import connections.json'
```
<!-- export SCHEDULER_POD_NAME="$(kubectl get pods --no-headers -o custom-columns=":metadata.name" -n orchestrator | grep scheduler)"
kubectl cp images/airflow/connections.json orchestrator/$SCHEDULER_POD_NAME:./ -c scheduler
kubectl -n orchestrator exec $SCHEDULER_POD_NAME -- airflow connections import connections.json -->


Ótimo, agora que você configurou as ferramentas necessárias, temos o ambiente de desenvolvimento e de execução instalado e pronto para uso.

### Executando o projeto


Antes de tudo, é necessário possuir uma `imagem do Spark` que contenha todos os JARs necessários para a execução do nosso pipeline. Para criar uma imagem do Spark com essas especificações, execute:


try later to push from docker repo.
```sh
eval $(minikube docker-env)
docker build --no-cache -f images/spark/dockerfile images/spark/ -t sparkminikube:0.1
```


## Acess airflow and check the admin/connections
```sh
kubectl get services -n orchestrator -l component=webserver,argocd.argoproj.io/instance=airflow -o jsonpath="{.items[0].status.loadBalancer.ingress[0].ip}"
```

Uma vez na interface do Airflow, ative o pipeline de dados `pipeline-delta-lake-deep-dive-complete` e veja a mágica acontecer. Ou, se preferir, você também pode executar cada etapa separadamente, seguindo a sequência:
  * `ingestion-from-local-data-file-to-bronze-tables`
  * `transformation-and-enrichment-from-bronze-to-silver`
  * `delivery-data-from-silver-to-gold`

Caso não deseje executar o pipeline pelo Airflow, você pode executar o pipeline de dados executando os seguintes comandos em sequência:
```sh
kubectl apply -f dags/spark_jobs/ingestion_from_local_data_file_to_bronze_tables.yaml -n processing
kubectl apply -f dags/spark_jobs/transform_and_enrichment_from_bronze_to_silver.yaml -n processing
kubectl apply -f dags/spark_jobs/delivery_data_from_silver_to_gold.yaml -n processing
```
Para verificar os arquivos no `data lakehouse`, acesse a interface web do `MinIO` e use as credenciais de acesso encontradas no arquivo *[minio-secrets.yaml](/secrets/minio-secrets.yaml)* na pasta *[secrets](/secrets/)*. Caso não saiba o IP atribuído ao MinIO, execute:

## geting miniO port

```sh
kubectl get services -n deepstorage -l app.kubernetes.io/name=minio -o jsonpath="{.items[0].status.loadBalancer.ingress[0].ip}"
```

Caso queira obter as credenciais de acesso do `MinIO`, execute:
```sh
echo "user: $(kubectl get secret minio-secrets -n deepstorage -o jsonpath="{.data.root-user}" | base64 -d)"
echo "password: $(kubectl get secret minio-secrets -n deepstorage -o jsonpath="{.data.root-password}" | base64 -d)"
```

check out the dag in airflow UI + logs, and the files at MiniO.