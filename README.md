# Guia Completo para Rodar MySQL Localmente com Docker

Este é um guia completo, prático e testado para rodar MySQL localmente usando Docker.
O objetivo é criar um ambiente robusto, persistente e fácil de replicar, ideal para desenvolvimento.

 ## 1. Conceitos Iniciais
 ### Por que usar Docker?

- Evita instalar o MySQL diretamente no sistema operacional

- Elimina conflitos de versão

- Ambiente isolado, fácil de recriar

- Se algo quebrar, basta remover o container e subir outro

### O que é o Dockerfile?

É uma **receita** que define como a imagem será construída.
Aqui, será usado para configurar o timezone do container.

### O que é o docker-compose.yml?

É o **maestro** do ambiente.
Ele define:

- As portas expostas

- Volumes de persistência

- Variáveis de ambiente

- Comandos de inicialização

## 2. Preparação dos Arquivos

Crie uma pasta para o projeto e adicione os arquivos abaixo.

 ### Passo 1 — Arquivo `.env` (Segurança)

Importante: nunca suba este arquivo para o Git.


```bash
# .env
MYSQL_ROOT_PASSWORD=root_secret
MYSQL_DATABASE=meubanco
MYSQL_USER=user
MYSQL_PASSWORD=password
TZ=America/Sao_Paulo
```

### Passo 2 — Arquivo `init.sql` (Inicialização automática)

Qualquer arquivo dentro do diretório `/docker-entrypoint-initdb.d/` é executado na primeira criação do banco.

```sql
-- init.sql
CREATE TABLE IF NOT EXISTS usuarios (
    id INT AUTO_INCREMENT PRIMARY KEY,
    nome VARCHAR(100) NOT NULL,
    email VARCHAR(100) UNIQUE NOT NULL,
    criado_em TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

INSERT INTO usuarios (nome, email) VALUES 
('João Silva', 'joao@teste.com'),
('Maria Souza', 'maria@teste.com');
```

### Passo 3 — Dockerfile (Imagem Personalizada)

```dockerfile
# Dockerfile
FROM mysql:8.0

# Define variáveis de ambiente para timezone (leitura correta de datas)
ENV TZ=America/Sao_Paulo

# Configura link simbólico para o horário local
RUN ln -snf /usr/share/zoneinfo/$TZ /etc/localtime && echo $TZ > /etc/timezone
```

🧩 Passo 4 — `docker-compose.yml` (Orquestração)

```yml
# docker-compose.yml
version: '3.8'

services:
  db:
    # Builda a imagem a partir do Dockerfile na pasta atual
    build: . 
    container_name: mysql_local_dev
    restart: always
    
    # Carrega variáveis do arquivo .env
    env_file:
      - .env
      
    # Mapeamento de portas: Host:Container
    ports:
      - "3306:3306"
      
    # Volumes para persistência e inicialização
    volumes:
      - mysql_data:/var/lib/mysql
      - ./init.sql:/docker-entrypoint-initdb.d/init.sql
      
    # Healthcheck: Garante que o serviço só é considerado "up" quando o MySQL aceita conexões
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost"]
      timeout: 20s
      retries: 10

volumes:
  mysql_data:
```

## 3. Executando e Gerenciando

Abra o terminal na pasta onde estão os arquivos.

 ### 1. Subir o serviço (Build + Up)
 ```bash
docker-compose up -d --build
```

Resultado esperado:
O Docker baixa a imagem, aplica configurações e inicia o container `mysql_local_dev`.

### 2. Verificar Status

```bash
docker-compose ps
# ou
docker ps
```

Você deve ver algo como:

```makefile
STATUS: Up (healthy)
PORTS: 0.0.0.0:3306->3306/tcp
```

### 3. Ver Logs

```bash
docker-compose logs -f
```

Procure por:

```arduino
ready for connections
```

### 4. Acessar o MySQL de dentro do container

```bash
docker exec -it mysql_local_dev mysql -u devuser -p
```

Depois, teste:
```bash
USE meubanco;
SELECT * FROM usuarios;
```

### 5. Parar e Limpar

Parar:

```bash
docker-compose down
```

Parar e **apagar volumes** (⚠ apaga o banco):

```bash
docker-compose down -v
```

## 4. Conceitos Importantes
### Persistência

O volume:

```yml
mysql_data:/var/lib/mysql
```

Garante que os dados persistem mesmo que o container seja removido.

Sem isso, tudo seria apagado a cada reinício.

## Scripts de Inicialização (init.sql)

Regras:

Arquivos em `/docker-entrypoint-initdb.d/` só rodam quando o banco é criado pela primeira vez.

Para rodar novamente:

```bash
docker-compose down -v
docker-compose up -d
```

## 5. Acesso Externo ao Banco
### MySQL Client (terminal local)
```bash
mysql -h 127.0.0.1 -P 3306 -u devuser -p
```
## Ferramentas gráficas (DBeaver / MySQL Workbench)
| Campo  | Valor |
| ------------- | ------------- |
| Host  | localhost  |
| Porta  | 3306  |
| Banco  | meubanco  |
| Usuário  | devuser  |
| Senho  | devpass  |


## Exemplo de conexão em Node.js

```js
const connectionString = "mysql://devuser:devpass@localhost:3306/meubanco";
```

## 6. Diferenças por Sistema Operacional
### Linux

- Melhor performance nativa

- Quase sem problemas de permissões quando usa volumes nomeados

### Windows (Docker Desktop)

- Use backend WSL 2 para melhor desempenho

- Acesso via localhost funciona normalmente

### macOS

- Imagem mysql:8.0 funciona em ARM64

- Em raros casos, pode usar:

```yml
platform: linux/amd64
```
## 7. Troubleshooting & Checklist
## Erro: Porta 3306 ocupada

```nginx
Bind for 0.0.0.0:3306 failed: port is already allocated
```


### Solução:

Pare serviços MySQL nativos

Ou altere no compose: `"3307:3306"`

## Connection Refused

Possível causa: banco ainda não está pronto.

### Solução:
Aguarde status `healthy`.

## Access Denied

Normalmente ocorre ao mudar senha depois que o volume já existe.

### Solução:
```bash
docker-compose down -v
docker-compose up -d
```
## init.sql não roda

Motivo: já existe volume criado.

### Solução: apagar volume:
```bash
docker-compose down -v
```
