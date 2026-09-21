# Desafio: Fundamentos de Kubernetes na Prática

API REST (PostgREST) integrada a um banco PostgreSQL, orquestrada num cluster
Kubernetes local (minikube), com armazenamento persistente, configuração
externalizada via ConfigMap/Secret, e descoberta de serviço via DNS interno
do cluster — tudo declarado em manifests YAML versionáveis.

> **Status deste README:** documenta os Níveis 1 a 5 (namespace, persistência,
> secrets/configmap, integração API↔banco e prova de persistência). Os Níveis
> 6 (health checks/escala) e 7 (HPA, bônus) estão em andamento e serão
> adicionados aqui assim que concluídos.

## Sumário

- [Sobre o projeto](#sobre-o-projeto)
- [Arquitetura](#arquitetura)
- [Ferramenta de cluster usada](#ferramenta-de-cluster-usada)
- [Como aplicar os manifests](#como-aplicar-os-manifests)
- [Como testar a API](#como-testar-a-api)
- [O que foi construído, nível por nível](#o-que-foi-construído-nível-por-nível)
- [Decisões técnicas e trade-offs](#decisões-técnicas-e-trade-offs)
- [Estrutura do repositório](#estrutura-do-repositório)
- [Reflexões do desafio](#reflexões-do-desafio)
- [Checklist de requisitos](#checklist-de-requisitos)

## Sobre o projeto

Este repositório implanta, do zero, uma aplicação integrada a um banco de
dados dentro de um cluster Kubernetes local, aplicando os conceitos
fundamentais de orquestração de contêineres: Namespace, Deployment, Service,
ConfigMap, Secret, PersistentVolumeClaim, e (nos próximos níveis) probes de
saúde e escalonamento.

A API (**PostgREST**) lê e grava dados num **PostgreSQL**, com os dados
persistidos em disco (sobrevivendo à recriação do Pod do banco), configuração
externalizada, e a aplicação acessível de fora do cluster via
`kubectl port-forward`.

## Arquitetura

```
Você (curl / navegador)
        │  HTTP (port-forward :3000)
        ▼
  Service "postgrest"  (ClusterIP :3000)
        │
        ▼
  Deployment "postgrest"  (imagem postgrest/postgrest)
        │  string de conexão via Secret (PGRST_DB_URI)
        │  resolve o host "postgres" via DNS interno do cluster
        ▼
  Service "postgres"  (ClusterIP :5432)
        │
        ▼
  Deployment "postgres"  (imagem postgres:16)
        │  monta o volume em /var/lib/postgresql/data
        ▼
  PersistentVolumeClaim "postgres-pvc"  (1Gi, ReadWriteOnce)
```

A peça central da integração: a API **nunca** usa o IP do Pod do Postgres —
ela se conecta pelo **nome do Service** (`postgres`), que o DNS interno do
cluster resolve para o Pod correto, mesmo que ele seja recriado e mude de IP.

## Ferramenta de cluster usada

**Minikube**, rodando localmente em Arch Linux, com o driver `docker`.

```bash
minikube start
kubectl get nodes   # confirma que o nó aparece como "Ready" antes de aplicar qualquer manifest
```

## Como aplicar os manifests

Os arquivos em `manifests/` são numerados na ordem exata em que devem ser
aplicados (o Namespace precisa existir antes de qualquer recurso que dependa
dele, o Secret antes do Deployment que o consome, e assim por diante):

```bash
kubectl apply -f manifests/
```

Isso aplica todos os arquivos da pasta de uma vez, na ordem alfabética dos
nomes — por isso a numeração (`00-`, `01-`, `02-`...) importa.

Para conferir que tudo subiu corretamente:

```bash
kubectl get all -n desafio-kubernetes
```

## Como testar a API

Com os Pods rodando, abra um túnel local até o Service da API:

```bash
kubectl port-forward -n desafio-kubernetes svc/postgrest 3000:3000
```

Em outro terminal:

```bash
# Listar itens
curl http://localhost:3000/items

# Inserir um item
curl -X POST http://localhost:3000/items \
  -H "Content-Type: application/json" \
  -d '{"name": "meu primeiro item"}'

# Confirmar que foi salvo
curl http://localhost:3000/items
```

## O que foi construído, nível por nível

### Nível 1 — Namespace e primeiro contato

Criado o namespace `desafio-kubernetes` (todos os recursos do projeto vivem
isolados nele) e, para explorar o comportamento básico de um Pod, foi criado
um Pod avulso de teste com `kubectl run`, inspecionado com `get`/`describe`/
`logs`, e depois deletado.

![Namespace criado](docs/screenshots/00-namespace.png)
![Inspeção do Pod de teste](docs/screenshots/01-pod-teste-inspecao.png)
![Pod de teste deletado](docs/screenshots/02-pod-teste-deletado.png)

**O que esse nível provou:** um Pod criado diretamente (sem um controlador
como Deployment) **não volta sozinho** quando deletado — depois do
`kubectl delete`, `kubectl get pods` retornou vazio. Não existe nenhum
processo de reconciliação cuidando dele. Isso justifica por que, na prática,
Pods quase nunca são criados diretamente — eles são gerenciados por
controladores (Deployments, no caso deste projeto) que garantem um número
desejado de réplicas rodando o tempo todo.

### Nível 2 — Banco de dados com persistência

Implantado o PostgreSQL (`postgres:16`) como Deployment, com um
`PersistentVolumeClaim` de 1Gi (`ReadWriteOnce`) montado em
`/var/lib/postgresql/data` — o caminho onde a imagem oficial grava seus
arquivos de dados. Um Service `ClusterIP` foi criado para que outros recursos
do cluster consigam localizar o banco pelo nome, não por IP.

![PVC vinculado (Bound)](docs/screenshots/06-pvc-bound.png)

![Deployment e Service do PostgreSQL criados](docs/screenshots/05-criacao-postgres-db.png)

### Nível 3 — Configuração e segredos

As credenciais do banco (`POSTGRES_USER`, `POSTGRES_PASSWORD`) foram movidas
para um `Secret` (tipo `Opaque`), injetado no container via `envFrom`. O nome
do banco (`POSTGRES_DB`), que não é uma informação sensível, foi colocado em
um `ConfigMap` separado.

![Secret criado e decodificação em Base64](docs/screenshots/03-secret-e-decodificacao.png)
![ConfigMap criado](docs/screenshots/04-criacao-configmap.png)

### Nível 4 — A API conectada ao banco (a integração)

Implantado o PostgREST como Deployment, configurado via variável de ambiente
`PGRST_DB_URI` — uma string de conexão que aponta para o **nome do Service**
do Postgres (`postgres`), não para um IP. Essa chave foi adicionada ao mesmo
Secret do Nível 3, reaproveitando usuário e senha já existentes. Uma tabela
`items` foi criada no banco, e confirmada como exposta automaticamente pela
API via `GET`/`POST`.

![API conectada ao banco, inserindo e lendo dados](docs/screenshots/08-api-conectada-ao-banco.png)

### Nível 5 — Expor a API e provar a persistência

Com a API já respondendo via `port-forward`, foi inserido um item de teste
e, em seguida, o **Pod do PostgreSQL foi deletado propositalmente**. O
Deployment recriou o Pod automaticamente (com um nome novo, confirmando que
era um Pod diferente do original). A API foi consultada novamente, e o item
inserido **antes** da deleção continuava lá.

![Pod do Postgres recriado + dado ainda acessível pela API](docs/screenshots/09-persistencia-comprovada.png)

**Isso comprova que os dados nunca estiveram "dentro" do Pod** — eles sempre
estiveram no volume persistente (PVC), que sobrevive independentemente de
qual Pod específico está montando ele no momento.

## Decisões técnicas e trade-offs

- **`stringData` em vez de `data` no Secret** — evita ter que converter
  manualmente cada valor para Base64; o Kubernetes faz essa codificação
  automaticamente ao aplicar.
- **A senha aparece duplicada dentro do Secret** (uma vez isolada em
  `POSTGRES_PASSWORD`, outra vez embutida em `PGRST_DB_URI`) — é uma
  limitação prática do Kubernetes puro, sem ferramentas de templating como
  Helm. Não é um problema de segurança adicional (o Secret já protege ambos
  os valores da mesma forma), só uma redundância de representação.
- **`PGRST_DB_ANON_ROLE` configurado como o próprio usuário `admin`
  (superusuário)** — simplificação consciente para manter o desafio direto.
  Em produção, o correto seria criar um *role* específico no PostgreSQL com
  permissões restritas apenas às tabelas que a API deve expor.
- **Sem `storageClassName` explícito no PVC** — o minikube já vem com uma
  `StorageClass` padrão habilitada (`default-storageclass`), então omitir o
  campo deixa o Kubernetes provisionar o volume automaticamente.

## Estrutura do repositório

```
.
├── manifests/
│   ├── 00-namespace.yaml          # Namespace isolando todos os recursos
│   ├── 01-secret.yaml             # Credenciais do Postgres + string de conexão da API
│   ├── 02-configmap.yaml          # Nome do banco (config não sensível)
│   ├── 03-postgres-pvc.yaml       # Armazenamento persistente (1Gi)
│   ├── 04-postgres-deployment.yaml
│   ├── 05-postgres-service.yaml
│   ├── 06-postgrest-deployment.yaml
│   └── 07-postgrest-service.yaml
├── docs/
│   └── screenshots/                # Evidências de cada nível, numeradas
└── README.md
```

## Reflexões do desafio

**Nível 1 — o Pod avulso volta sozinho ao ser deletado?**
Não. Sem um controlador (Deployment, StatefulSet, etc.) supervisionando,
não existe nenhum processo garantindo um estado desejado ao deletar, o
Pod simplesmente deixa de existir. É por isso que, na prática, Pods quase
nunca são criados diretamente.

**Nível 2 — PVC vs. `emptyDir`, qual a diferença?**
Um `emptyDir` também é um volume, mas seu ciclo de vida está atrelado ao
**Pod**, não ao cluster, se o Pod for removido, os dados do `emptyDir` vão
junto. Um PVC é um recurso independente: mesmo que o Pod que o utiliza seja
destruído e recriado, o volume por trás do PVC continua existindo, e o novo
Pod pode montar o mesmo volume, com os mesmos dados.

**Nível 3 — o valor do Secret em `-o yaml` é criptografia de verdade?**
Não,é apenas **codificação Base64**, reversível por qualquer pessoa que
tenha acesso ao valor codificado (`echo "valor" | base64 -d` decodifica
instantaneamente, sem nenhuma chave secreta envolvida). Um Secret do
Kubernetes evita que a credencial fique visível "a olho nu" dentro do YAML
do Deployment, mas não é, por si só, uma proteção equivalente a criptografia
real contra alguém com acesso de leitura ao cluster.

**Nível 4 — por que usar o nome do Service em vez do IP do Pod?**
Porque o IP de um Pod muda toda vez que ele é recriado. Se a string de
conexão da API apontasse para um IP fixo, bastaria o Pod do banco reiniciar
uma única vez para a conexão quebrar permanentemente. O Service oferece um
nome estável, resolvido via DNS interno do cluster, que sempre aponta para o
Pod correto independente de quantas vezes ele seja recriado ou qual IP
tenha no momento.

**Nível 5 — quantos componentes tiveram que funcionar juntos para o dado sobreviver?**
Cinco: o **PVC** (guardando os dados fisicamente), o **Deployment do
Postgres** (recriando o Pod automaticamente), o **Service do Postgres**
(mantendo a API capaz de encontrá-lo mesmo após a recriação), o **Secret**
(fornecendo as credenciais corretas ao novo Pod) e a **API** (consultando o
banco de novo, sem nenhuma reconfiguração manual). Isso mostra como o
Kubernetes coordena múltiplas peças independentes para entregar resiliência
sem intervenção humana.

## Checklist de requisitos

- [x] Namespace próprio criado e todos os recursos isolados nele
- [x] PostgreSQL rodando com PersistentVolumeClaim
- [x] Credenciais do banco em Secret (não hardcoded no YAML)
- [x] Configuração não sensível em ConfigMap
- [x] API (PostgREST) conectada ao banco pelo nome do Service
- [x] API acessível de fora do cluster e servindo dados do banco
- [x] Dado inserido pela API sobrevive à deleção do Pod do banco (persistência comprovada)
- [ ] Liveness e Readiness probes configuradas na API *(Nível 6, em andamento)*
- [ ] Requests e Limits de recursos definidos *(Nível 6, em andamento)*
- [x] Manifests versionados em arquivos YAML organizados
- [x] README documentando como aplicar e testar o projeto