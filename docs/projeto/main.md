# Relatório do Projeto de API em Nuvem

## Objetivo

Este relatório documenta o desenvolvimento, containerização e implantação de uma API RESTful em FastAPI, integrada a um banco de dados MySQL, distribuída via Docker Hub e orquestrada tanto localmente (Docker Compose) quanto em ambiente OpenStack (gerenciado por Juju), com balanceamento de carga via Nginx.

## 1. Tarefa 1 - Desenvolvimento da API com FastAPI

### 1.1 Rotas de Usuário

1. **POST /registrar**

   * Recepção de JSON com `nome`, `email` e `senha` para criar usuário.
2. **POST /login**

   * Validação de credenciais e retorno de JWT.
3. **GET /consultar**

   * Rota protegida que exige token Bearer.

### 1.2 Autenticação e Segurança

* Uso de **bcrypt** (via Passlib) para hashear senhas.
* Emissão de tokens JWT com `python-jose`, configurados para expirar em 30 minutos.
* Substituição de OAuth2 por **HTTPBearer** para simplificar o fluxo de teste no Swagger UI.

### 1.3 Modelagem de Dados e Hash de Senha

* Funções `get_password_hash()` e `verify_password()` em `app/core/security.py`.
* Criação de `create_access_token()` e `decode_access_token()` para JWT.

## 2. Tarefa 1.2 - Modelagem e Persistência com SQLAlchemy + MySQL

### 2.1 Configuração do SQLAlchemy (`app/core/database.py`)

* `DATABASE_URL` lido de variável de ambiente.
* Definição de `engine`, `SessionLocal` e `Base` para MySQL via PyMySQL.

### 2.2 Definição do Modelo ORM (`app/models.py`)

```python
class User(Base):
    __tablename__ = "users"
    id = Column(Integer, primary_key=True, index=True)
    nome = Column(String(255), nullable=False)
    email = Column(String(255), unique=True, index=True, nullable=False)
    senha = Column(String(255), nullable=False)
```

### 2.3 Funções CRUD

* `create_user(db, user: Usuario)`: insere novo usuário com hash de senha.
* `get_user_by_email(db, email)`: retorna usuário ou `None`.
* `authenticate_user(db, email, senha)`: compara senha em texto com hash salvo.

### 2.4 Dependência de Sessão no FastAPI (`app/app.py`)

* `get_db()` retorna `SessionLocal` para cada requisição.
* Evento de startup com retry para aguardar o MySQL criado pelo Compose.

## 3. Tarefa 2 - Containerização Local via Docker e Docker Compose

### 3.1 Dockerfile

* **Base:** `python:3.10-slim`
* Instalação de dependências via `requirements.txt` (FastAPI, Uvicorn, SQLAlchemy, PyMySQL, Passlib, python-jose).
* Comando de inicialização:

  ```bash
  CMD ["uvicorn", "app.app:app", "--host", "0.0.0.0", "--port", "8080"]
  ```

### 3.2 docker-compose.yml

```yaml
version: "3.8"

services:
  db:
    image: mysql:8.0
    container_name: mysql_projeto
    restart: always
    environment:
      MYSQL_ROOT_PASSWORD: rootpassword
      MYSQL_DATABASE: p_cloud
      MYSQL_USER: cloud
      MYSQL_PASSWORD: cloud-n123
    healthcheck:
      test: ["CMD","mysqladmin","ping","-h","localhost","-ucloud","-pcloud-n123"]
      interval: 5s
      retries: 5
      timeout: 2s
    volumes:
      - mysql_data:/var/lib/mysql
    ports:
      - "3307:3306"

  app:
    build:
      context: .
      dockerfile: Dockerfile
    image: luizdurand/api-fastapi-nuvem:latest
    container_name: api_fastapi
    ports:
      - "8080:8080"
    environment:
      DATABASE_URL: mysql+pymysql://cloud:cloud-n123@db:3306/p_cloud
      SECRET_KEY: sua_chave_jwt_super_secreta
    depends_on:
      db:
        condition: service_healthy

volumes:
  mysql_data:
```

### 3.3 Testes Locais

1. Executar: `docker compose up -d --build`.
2. Acessar: `http://localhost:8080/docs` para validar CRUD e endpoint protegido.
3. No MySQL Workbench, configurar conexão em `127.0.0.1:3307` (usuário `cloud`, schema `p_cloud`).

## 4. Tarefa 3 - Integração com Ambiente OpenStack (via Juju)

### 4.1 Provisionamento de MySQL InnoDB Cluster

* Charm: `mysql-innodb-cluster/8.0` com 3 unidades (R/W/R/O).
* Exposição do MySQL Router em `172.16.0.152:3306`.

### 4.2 Configuração das VMs de API

* Instâncias: Ubuntu 22.04 (`api-vm1`, `api-vm2`) criadas via Juju.
* Instalar Docker:

  ```bash
  sudo apt update && sudo apt install -y docker.io
  ```
* Pulando a imagem pública:

  ```bash
  docker pull luizdurand/api-fastapi-nuvem:latest
  ```
* Executar container apontando para MySQL Router:

  ```bash
  docker run -d \
    --name api_fastapi \
    -e DATABASE_URL="mysql+pymysql://cloud:cloud-n123@172.16.0.152:3306/p_cloud" \
    -e SECRET_KEY="sua_chave_jwt_super_secreta" \
    -p 8080:8080 \
    luizdurand/api-fastapi-nuvem:latest
  ```

### 4.3 Verificação do Status Juju

* Comando: `juju status mysql-innodb-cluster/1`.
* Resultado: `public-address: 172.16.0.152`.
* O `mysql-router/0` gerencia fail-over.

### 4.4 Configuração do Load Balancer (lb-vm)

* Instalar Nginx:

  ```bash
  sudo apt update && sudo apt install -y nginx
  ```
* Arquivo `/etc/nginx/sites-available/default`:

  ```nginx
  upstream apis {
      server <IP_API_VM1>:8080;
      server <IP_API_VM2>:8080;
  }
  server {
      listen 80;
      location / {
          proxy_pass http://apis;
          proxy_set_header Host $host;
          proxy_set_header X-Real-IP $remote_addr;
      }
  }
  ```
* Testar e reiniciar:

  ```bash
  sudo nginx -t && sudo systemctl restart nginx
  ```
* Associar Floating IP à VM de LB para acesso externo: `http://<FLOATING_IP>/docs`.

## 5. Tarefa 4 - Publicação da Imagem no Docker Hub

### 5.1 Build e Push

```bash
cd ~/Desktop/CLOUD/api_cloud/api
docker build -t luizdurand/api-fastapi-nuvem:latest .
docker login
docker push luizdurand/api-fastapi-nuvem:latest
```

### 5.2 Teste em Nova VM

1. Em uma VM qualquer:

   ```bash
   docker pull luizdurand/api-fastapi-nuvem:latest
   docker run -d \
     --name api_fastapi \
     -e DATABASE_URL="mysql+pymysql://cloud:cloud-n123@db-externo:3306/p_cloud" \
     -e SECRET_KEY="sua_chave_jwt_super_secreta" \
     -p 8080:8080 \
     luizdurand/api-fastapi-nuvem:latest
   ```
2. Acessar: `http://<IP_VM>:8080/docs` e verificar conexão com banco remoto.

## 6. Testes Finais e Conclusão

* **Persistência:** após reiniciar containers ou VMs, dados em `users` permanecem.
* **Autenticação:** fluxo de registro, login e consulta protegido por JWT funcionando.
* **Balanceamento:** Nginx distribui tráfego entre `api-vm1` e `api-vm2`; falha de uma instância não interrompe serviço.
* **Alta Disponibilidade:** InnoDB Cluster + mysql-router garantem continuidade em caso de falha de até uma unidade.

O projeto demonstra a integração completa entre código, containers, repositório Docker e infra em nuvem, garantindo escalabilidade, segurança e alta disponibilidade.

## 7. Tarefa 5 - Deploy na AWS Lightsail

Além das etapas anteriores, realizamos o deploy da API e do MySQL na AWS Lightsail. As principais configurações foram:

### 7.1 Instância MySQL (fastapi-db)

* **Provisionamento:** Criamos uma instância Lightsail do tipo MySQL (8.0.42) com 1 GB RAM, 2 vCPUs e 40 GB SSD.
* **Credenciais:** usuário `admin` com senha visível nas configurações (e.g. `SenhaSegura123`).
* **Endpoint Público:** Exemplo de endpoint: `ls-6f2e65f181730bb8f397f6fa2692ccbcd64f9e99.c90ug4gek4kp4.us-west-2.rds.amazonaws.com:3306`.
* **Configurações de Rede:** Modo público habilitado para permitir conexões externas.

### 7.2 Container Service (fastapi-service)

* **Configuração do Serviço de Container:** Criamos um serviço Lightsail do tipo Nano (512 MB RAM, 0.25 vCPUs).
* **Imagem Docker:** `luizdurand/api-fastapi-nuvem:latest` carregada diretamente no Console do Lightsail.
* **Variáveis de Ambiente:** Definimos:

  * `DATABASE_URL`: `mysql+pymysql://admin:SenhaSegura123@ls-6f2e65f181730bb8f397f6fa2692ccbcd64f9e99.c90ug4gek4kp4.us-west-2.rds.amazonaws.com:3306/dbmaster`
  * `SECRET_KEY`: `troque_isto_por_uma_chave_secreta`
* **Portas Abertas:** Porta 8080 exposta publicamente.

Em seguida, habilitamos o serviço e aguardamos o deploy concluir. A API passou a responder em um domínio público fornecido pelo Lightsail, permitindo acesso externo via HTTP.

Para visualizar a demonstração da aplicação FastAPI em funcionamento após o deploy na AWS Lightsail, assista ao vídeo a seguir: https://youtu.be/yFd4dmGgxxA

---

* **Configuração do Serviço de Container:** Criamos um serviço Lightsail do tipo Nano (512 MB RAM, 0.25 vCPUs).
* **Imagem Docker:** `luizdurand/api-fastapi-nuvem:latest` carregada diretamente no Console do Lightsail.
* **Variáveis de Ambiente:** Definimos:

  * `DATABASE_URL`: `mysql+pymysql://admin:SenhaSegura123@ls-6f2e65f181730bb8f397f6fa2692ccbcd64f9e99.c90ug4gek4kp4.us-west-2.rds.amazonaws.com:3306/dbmaster`
  * `SECRET_KEY`: `troque_isto_por_uma_chave_secreta`
* **Portas Abertas:** Porta 8080 exposta publicamente.

Em seguida, habilitamos o serviço e aguardamos o deploy concluir. A API passou a responder em um domínio público fornecido pelo Lightsail, permitindo acesso externo via HTTP.

## 8. Estimativa de Custo Mensal

Para estimar o custo mensal da infraestrutura na AWS Lightsail, utilizamos dois componentes principais:

1. **Instância de Banco de Dados (fastapi-db)**

   * Escolhemos o **plano Standard de \$15 USD/mês**, que inclui:

     * **1 GB RAM**
     * **2 vCPUs**
     * **40 GB SSD**
   * Este plano atende à necessidade de armazenamento do banco MySQL e garante desempenho suficiente para operações CRUD básicas.

   ![Plano de Banco Lightsail](db.png)

2. **Serviço de Container (fastapi-service)**

   * Selecionamos o **plano Nano**, que inclui:

     * **512 MB RAM**
     * **0.25 vCPUs**

   * A variação de nós (scale) impacta diretamente o custo mensal do serviço:

     | Número de Nós | Custo por mês (USD) |
     | ------------- | ------------------- |
     | 1             | 7                   |
     | 5             | 35                  |
     | 10            | 70                  |

   * Essas opções permitem ajustar a capacidade de processamento conforme demanda, mantendo-se dentro do orçamento.

   ![Plano de Container Lightsail - 1 nó](1no.png)
   ![Plano de Container Lightsail - 1 nó](5nos.png)
   ![Plano de Container Lightsail - 1 nó](10nos.png)

### Cálculo Total

* **Banco de Dados (fixo):** \$15 USD/mês

* **Serviço de Container (variável):**

  * **1 nó:** \$7 USD/mês
  * **5 nós:** \$35 USD/mês
  * **10 nós:** \$70 USD/mês

**Total estimado (exemplo para 1 nó):** \$22 USD/mês
**Total estimado (exemplo para 5 nós):** \$50 USD/mês
**Total estimado (exemplo para 10 nós):** \$85 USD/mês

Todos os valores permanecem dentro do limite orçado (≤ USD 50) para configurações até 5 nós. Para 10 nós, ultrapassa o limite, então deve-se avaliar a necessidade de escala.

Este valor inclui apenas os recursos principais (banco e container). A AWS Lightsail não cobra por transferência de dados dentro da cota mensal de 500 GB; excedentes podem gerar custos adicionais baixos (ex.: \$0.09 USD/GB extra). Assim, permanecemos confortavelmente dentro do orçamento para configurações apropriadas.

---

## 9. Diagrama da Arquitetura

A seguir, apresentamos o diagrama da arquitetura final do projeto. Ele ilustra como o **Client / User** interage com o **Lightsail Container Service (API Container)** via HTTPS e como a API se comunica com o **Lightsail Managed MySQL (Database)** via SQL. Esse diagrama está localizado imediatamente abaixo deste texto.

![Arquitetura Final do Projeto](diag.png)

---
