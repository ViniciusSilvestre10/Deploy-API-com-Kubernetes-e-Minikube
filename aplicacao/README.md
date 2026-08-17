# Vollmed API - Kubernetes Deployment

Projeto de implantação da API **Vollmed** (Node.js) e banco de dados **MySQL** em um cluster Kubernetes (Minikube), incluindo monitoramento via Dashboard, backups automáticos e criptografia de Secrets.

## 📋 Sobre o projeto

A aplicação é composta por:
- **API Vollmed**: aplicação Node.js que se conecta a um banco MySQL.
- **Banco de dados MySQL**: rodando como StatefulSet, com armazenamento persistente.
- **CronJob de backup**: gera dumps automáticos do banco de dados diariamente.
- **ConfigMap e Secrets**: separam configurações não-sensíveis (host, nome do banco) de dados sensíveis (usuário, senha).
- **Services**: expõem a aplicação via `LoadBalancer`/`NodePort` e o banco via Service headless.

## 🗂️ Estrutura de arquivos

```
.
├── Dockerfile                  
├── configmap.yaml               # Variáveis de ambiente não sensíveis
├── secrets.yaml                 # Variáveis de ambiente sensíveis
├── mysql.yaml                   # Service (headless) + StatefulSet + PVC do MySQL
├── mysql-dump-cronjob.yaml      # CronJob de backup automático
├── vollmed-deployment.yaml      # Deployment da API Vollmed
├── vollmed-service.yaml         # Service LoadBalancer da API
├── encryption-config.yaml       # Configuração de criptografia de Secrets/ConfigMaps
└── README.md
```


## ✅ Pré-requisitos

- [Docker](https://docs.docker.com/get-docker/)
- [Minikube](https://minikube.sigs.k8s.io/docs/start/)
- [kubectl](https://kubernetes.io/docs/tasks/tools/)
- Node.js (para testar a aplicação localmente antes do container)

## 🚀 Passo a passo

### 1. Testar a aplicação localmente (antes do Kubernetes)

Antes de subir qualquer coisa no cluster, valide se a aplicação funciona isoladamente:

```bash
npm install
npm start
```

Depois, teste também dentro de um container Docker isolado:

```bash
docker build -t vollmed-api .
docker run -p 3000:3000 vollmed-api
```

Isso garante que qualquer erro futuro seja de configuração do Kubernetes, e não da aplicação em si.

### 2. Iniciar o Minikube

```bash
minikube start
```

### 3. Habilitar o Dashboard do Kubernetes

O Dashboard ajuda a visualizar e confirmar que os recursos do cluster estão funcionando corretamente.

```bash
minikube addons enable dashboard
minikube dashboard
```

### 4. Instalar o metrics-server

Necessário para o Dashboard exibir uso de CPU/memória dos pods e nodes.

```bash
minikube addons enable metrics-server
```

Verifique se está funcionando:
```bash
kubectl top nodes
kubectl top pods
```

### 5. Aplicar o ConfigMap

Guarda variáveis não sensíveis, como o nome do banco e o host de conexão.

```bash
kubectl apply -f configmap.yaml
```

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: dados
data:
  DB_DATABASE: "testemed"
  DB_HOST: "mysql"
```

### 6. Aplicar os Secrets

Guarda variáveis sensíveis, como usuário e senha do banco.

```bash
kubectl apply -f secrets.yaml
```

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: senhas
stringData:
  DB_USER: "root"
  DB_PASSWORD: "12345"
```

### 7. Aplicar o banco de dados MySQL (Service + StatefulSet + PVC)

```bash
kubectl apply -f mysql.yaml
```

Esse arquivo contém três recursos:
- **Service headless** (`clusterIP: None`): fornece um DNS estável (`mysql`) para o pod do banco.
- **StatefulSet**: garante identidade estável do pod e volume persistente dedicado.
- **PersistentVolumeClaim**: reserva 5Gi de armazenamento para os dados do MySQL.

Verifique se o pod subiu corretamente:
```bash
kubectl get pods -l app=mysql
kubectl logs -l app=mysql
```

### 8. Aplicar o Deployment da aplicação Vollmed

```bash
kubectl apply -f vollmed-deployment.yaml
```

Esse Deployment sobe **3 réplicas** da API, injetando as variáveis do ConfigMap e Secret, e monitora a saúde da aplicação através de uma **liveness probe HTTP**:

```yaml
livenessProbe:
  httpGet:
    path: /paciente
    port: 3000
  initialDelaySeconds: 15
  periodSeconds: 3
```

A probe faz requisições periódicas (a cada 3s, após 15s de espera inicial) ao endpoint `/paciente`. Se a aplicação parar de responder corretamente, o Kubernetes reinicia o Pod automaticamente.

Verifique o status:
```bash
kubectl get deployments
kubectl get pods -l app=vollmed
```

### 9. Aplicar o Service da aplicação (LoadBalancer)

```bash
kubectl apply -f vollmed-service.yaml
```

```yaml
apiVersion: v1
kind: Service
metadata:
  name: loadbalancer-vollmed
spec:
  type: LoadBalancer
  selector:
    app: vollmed
  ports:
  - port: 3000
    targetPort: 3000
```

Como o Minikube não tem um LoadBalancer real de nuvem, é preciso expor esse serviço manualmente:

```bash
minikube service loadbalancer-vollmed
```

Esse comando abre o navegador (ou fornece a URL) apontando para a aplicação, usando um túnel criado pelo próprio Minikube.

> Alternativa via NodePort, caso prefira acesso por porta fixa em vez de LoadBalancer:
> ```yaml
> spec:
>   type: NodePort
>   ports:
>   - port: 3000
>     targetPort: 3000
>     nodePort: 30000
> ```

### 10. Agendar backups automáticos do banco (CronJob)

```bash
kubectl apply -f mysql-dump-cronjob.yaml
```

O CronJob executa diariamente às 3h da manhã (`schedule: "0 3 * * *"`), rodando uma imagem que faz o dump do banco MySQL e salva no host, usando as credenciais do ConfigMap/Secret.

```yaml
spec:
  schedule: "0 3 * * *"
```

Verifique execuções agendadas:
```bash
kubectl get cronjobs
kubectl get jobs
```

Veja os logs de uma execução específica:
```bash
kubectl logs job/<nome-do-job-gerado>
```

> ⚠️ O CronJob usa `hostPath` para salvar o backup (`/home/vinicius-ubuntu/volume_kubernetes`), o que funciona no Minikube (single-node) mas **não é recomendado em clusters multi-node de produção**, já que o backup fica preso a um nó específico.

### 11. Ativar criptografia dos Secrets/ConfigMaps em disco

Por padrão, o `etcd` (banco interno do Kubernetes) armazena Secrets **sem criptografia**. Para proteger essas credenciais em disco, é usado um `EncryptionConfiguration`:

```yaml
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
  - resources:
      - secrets
      - configmaps
    providers:
      - aescbc:
          keys:
            - name: key1
              secret: <chave-base64-gerada>
      - identity: {}
```

Passos para ativar:

1. Gere uma chave aleatória de 32 bytes em base64:
   ```bash
   head -c 32 /dev/urandom | base64
   ```
2. Substitua o valor de `secret` no arquivo `encryption-config.yaml` pela chave gerada.
3. Coloque o arquivo em um caminho acessível pelo `kube-apiserver` (ex: `/etc/kubernetes/enc/encryption-config.yaml`).
4. Configure o `kube-apiserver` para usar esse arquivo, adicionando a flag:
   ```
   --encryption-provider-config=/etc/kubernetes/enc/encryption-config.yaml
   ```
5. Reinicie o `kube-apiserver`.
6. (Opcional, mas recomendado) Re-criptografe os Secrets já existentes:
   ```bash
   kubectl get secrets -A -o json | kubectl replace -f -
   ```

> O provider `identity: {}` no final permite que Secrets **antigos, não criptografados**, ainda sejam lidos normalmente durante a migração — por isso ele fica como fallback, depois do `aescbc`.

## 🔍 Comandos úteis

**Listar todos os recursos principais:**
```bash
kubectl get pods,deployments,secrets,configmaps,services
```

**Ver detalhes e eventos de um pod (útil para depurar probes):**
```bash
kubectl describe pod <nome-do-pod>
```

**Ver logs de um pod (por label, sem precisar do nome exato):**
```bash
kubectl logs -l app=vollmed
kubectl logs -l app=mysql
```

**Deletar todos os recursos aplicados:**
```bash
kubectl delete -f .
```

