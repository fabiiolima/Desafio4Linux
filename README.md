# Plataforma de Observabilidade 4Linux

Automação Ansible do desafio técnico de observabilidade. A solução separa os componentes centrais na AWS das cargas monitoradas na GCP.

## Arquitetura

```text
AWS — K3s local
├── Prometheus
├── Grafana
├── Zabbix Server + Agent 2 + PostgreSQL
├── OpenSearch + OpenSearch Dashboards
└── Fluent Bit local

GCP — K3s com três nodes
├── master01: Coffee Shop + Fluent Bit + node-exporter + Zabbix Agent
├── worker01: Coffee Shop + Fluent Bit + node-exporter + Zabbix Agent
└── worker02: Coffee Shop + Fluent Bit + node-exporter + Zabbix Agent
```

O `kube-state-metrics` e o Prometheus coletor rodam no cluster GCP. O coletor consulta kubelet/cAdvisor e envia CPU, memória e filesystem de pods por `remote_write` (conexão de saída) ao Prometheus da AWS. O Prometheus da AWS consulta métricas da aplicação, dos nodes e dos objetos Kubernetes. Fluent Bit envia logs de containers e dos nodes para o OpenSearch da AWS.

## Estrutura Ansible

| Caminho | Responsabilidade |
|---|---|
| `roles/preparacao_linux` | Pacotes básicos das VMs GCP |
| `roles/cluster_k3s` | Control plane e workers K3s |
| `roles/kubeconfig_remoto` | Acesso da AWS ao K3s GCP |
| `roles/aplicacao_coffee_shop` | Manifests e validação da aplicação |
| `roles/agentes_gcp` | Zabbix Agent e node-exporter |
| `roles/coletores_gcp` | Fluent Bit e kube-state-metrics |
| `roles/observabilidade_aws` | Stack Helm central da AWS |

As variáveis ficam exclusivamente em `inventory/group_vars/all.yml`. Os playbooks de observabilidade validam o kubeconfig utilizado para impedir instalações no ambiente errado.

## Execução

Instale a collection necessária:

```bash
ansible-galaxy collection install -r requirements.yml
```

Implantação completa:

```bash
ansible-playbook playbooks/implantacao_completa.yml
```

Execução por etapa:

```bash
ansible-playbook site.yml
ansible-playbook playbooks/monitoring.yml
ansible-playbook playbooks/coffee-shop.yml
ansible-playbook playbooks/agents.yml
```

A Coffee Shop é aplicada pelo módulo `kubernetes.core.k8s`. O playbook valida três pods em execução e três valores distintos de `spec.nodeName`, garantindo uma réplica no master e uma em cada worker.

## Endpoints e credenciais

| Componente | Endereço | Credencial inicial |
|---|---|---|
| Coffee Shop | `http://34.133.198.40:30000` | Não aplicável |
| OpenSearch Dashboards | `http://54.173.233.120:30080` | Segurança desabilitada no laboratório |
| Zabbix | `http://54.173.233.120:30081` | Usuário e senha |
| Prometheus | `http://54.173.233.120:30082` | Não aplicável |
| Grafana | `http://54.173.233.120:30083` | Usuário e senha |
| OpenSearch API | `http://54.173.233.120:30084` | Segurança desabilitada no laboratório |

## Métricas

O Prometheus coleta:

- `/metrics` da Coffee Shop;
- node-exporter dos três servidores;
- kube-state-metrics do cluster GCP;
- kubelet/cAdvisor: CPU, memória e filesystem reais de pods;
- métricas do K3s local da AWS.

O Zabbix recebe checks ativos dos três agentes GCP e coleta passivamente as métricas do próprio Zabbix Server pelo Agent 2 em sidecar. Os hosts GCP usam o template Linux e possuem verificações de CPU, memória, disco, rede e disponibilidade.

## Logs

Há um Fluent Bit em cada node GCP. Eles leem `/var/log/containers`, adicionam metadados Kubernetes e enviam os eventos ao OpenSearch da AWS. O índice diário é `coffee-shop-*` e contém logs da aplicação e da infraestrutura Kubernetes. O OpenSearch Dashboards recebe automaticamente o index pattern `coffee-shop-*`, com `@timestamp` como campo temporal e os campos do índice atualizados.

## GitLab CI

O clone original está configurado como remoto `upstream`, evitando push acidental ao projeto de terceiros. Para associar seu projeto:

```bash
cd /home/ubuntu/4linux/coffee-shop
git remote add origin URL_DO_SEU_PROJETO.git
git push -u origin main
```

## Segurança e recomendações

Este é um ambiente de avaliação. Como evolução, mova senhas e tokens para Ansible Vault, habilite TLS, restrinja NodePorts aos IPs da AWS/GCP, aumente o disco da AWS e troque senhas iniciais após a entrega.
