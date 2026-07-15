# Relatório Técnico — Plataforma de Observabilidade

## Estrutura

- A estrutura foi automatizada com Ansible e dividida entre AWS e GCP
- Ambiente na AWS é a máquina que gerencia e orquestra os servidores na GCP
- A GCP contém um cluster k3s e dois workers
- A AWS contém um k3s local dedicado aos serviços centrais de observabilidade (Zabbix, Grafana e OpenSearch)

## Ansible

- As roles for organizados por responsabilidade
- As variáveis estão em `inventory/group_vars/all.yml`
- Utilizamos templantes .yml.j2 para permitir leitura das variáveis
- O provisionamento da stack de observabilidade também é realizada pelo Ansible utilizando Helm para realizar a instalação no Kubernetes na AWS

## CI/CD

- Houve dificuldade ao realizar implantação da aplicação via GitLab CI ou GitHub Actions
- A fim manter a aplicação no ar foi efetuado provisionamento utilizando o próprio Ansible
	Neste caso em específico é necessário realizar o Git Clone do repositório para máquina local.
	Mesmo efetuando processo manual, ainda é possível automatiza-lo com python por exemplo.

Fluxo principal:
 implantacao_completa.yml
  └── coffee-shop.yml
      └── role aplicacao_coffee_shop
          ├── tasks/main.yml
          └── templates/coffee-shop.yml.j2

## Observabilidade

- No GCP é instalado um Prometheus reduzido que descobre e coleta automaticamente métricas do kubelet
- As métricas são: `container, kubelet, kube e node_namespace_pod_container`
- Na AWS o Prometheus recebe a armazena as séries de métricas

- Os containers do Zabbix foi dividido entre `Server, Banco, Web e Agent`
- Zabbix Server possui Agent 3 para coletar as próprias métricas
- É utilizado Zabbix Agent Ativo para monitorar as 3 VMs GCP

- Fluent Bit coleta logs dos containers Kubernetes
- Os logs são centralizados no OpenSearch AWS

- Grafana possui datasources Prometheus, Zabbix e OpenSearch provisionadas automaticamente.


## Recomendações Futuras

- Recomenda-se como evolução habilitar TLS.
- Usar Ansible Vault e ampliar o armazenamento da AWS.
- Usuário de senha de leitura para visualização no Zabbix e Grafana.
- Adicionar Alta Disponibilidade e rotinas de Backup.
- Realizar Tunning no banco de dados Zabbix a fim receber maior carga de dados.
- Implantar Zabbix Proxy para processar maior carga de dados.
- Centralizar envio de alertas pelo próprio Zabbix.
