# n8n com Túnel Ngrok

leia este README em [English 🇺🇸](README.md)

Este repositório contém uma configuração do Docker Compose para rodar o **n8n** com o **Ngrok** como serviço de tunelamento. O n8n é uma ferramenta de automação de fluxo de trabalho que permite conectar diferentes serviços e APIs. O Ngrok expõe servidores locais atrás de NATs e firewalls para a internet pública através de túneis seguros.

> Baseado em [joffcom/n8n-ngrok](https://github.com/joffcom/n8n-ngrok) com configuração adicional para suporte a HTTPS e OAuth.

## Pré-requisitos

Antes de começar, certifique-se de ter o seguinte instalado:
- [Docker](https://docs.docker.com/get-docker/) ou [Podman](https://podman.io/getting-started/installation)
- [Docker Compose](https://docs.docker.com/compose/install/) (mesmo se estiver usando o Podman)

## Configuração

### 1. Clonar o Repositório

```bash
git clone https://github.com/nelsonsoares/n8n-ngrok-https.git
cd n8n-ngrok-https
```

### 2. Autenticação do Ngrok

Você precisa de uma conta no Ngrok. Se não tiver uma, crie em [ngrok.com](https://ngrok.com/). Após criar a conta, obtenha seu token de autenticação no painel do Ngrok.

### 3. Reservar um Domínio Permanente

Por padrão, o Ngrok pode fornecer um domínio efêmero. Para que esta configuração funcione de forma confiável, é altamente recomendável **reservar um domínio permanente** no seu Painel do Ngrok em **Cloud Edge > Domains**.

Se você já tiver um domínio reservado, copie-o; caso contrário, o Ngrok atribuirá um a você na primeira conexão, mas você deve reservá-lo para evitar ter que alterar seu `.env` toda vez que reiniciar o serviço.

### 4. Configurar Variáveis de Ambiente

1. Copie o arquivo de template para criar seu próprio arquivo `.env`:
   ```bash
   cp .env-template .env
   ```
2. Abra o arquivo `.env` e preencha seus detalhes:

```sh
TIMEZONE=America/Sao_Paulo
NGROK_TOKEN=seu_auth_token_do_ngrok_aqui
URL=https://seu-dominio.ngrok-free.dev
NGROK_DOMAIN=seu-dominio.ngrok-free.dev

# Banco de Dados (PostgreSQL com pgvector)
POSTGRES_USER=seu_usuario_do_banco
POSTGRES_PASSWORD=sua_senha_do_banco
POSTGRES_DB=nome_do_seu_banco

# Redis (para Modo Fila / Queue Mode)
REDIS_PASSWORD=sua_senha_do_redis

# Nome do Projeto Docker (Garante portabilidade se a pasta for renomeada)
COMPOSE_PROJECT_NAME=n8n-ngrok
```

> [!CAUTION]
> **AVISO DE SEGURANÇA:** Nunca faça commit do seu arquivo `.env` para o controle de versão. Este projeto inclui um arquivo `.gitignore` que exclui o `.env` por padrão. Se você vazar acidentalmente seu `NGROK_TOKEN` ou `NGROK_DOMAIN`, rotacione seus segredos imediatamente no painel do Ngrok.

> [!IMPORTANT]
> - A variável `URL` **deve** incluir o prefixo `https://` para o OAuth funcionar.
> - A variável `NGROK_DOMAIN` agora é passada **dinamicamente** para o container do Ngrok via `docker-compose.yaml`. Isso elimina a necessidade de um arquivo `ngrok.yml` externo, melhorando a compatibilidade com o Podman (evitando problemas de permissão de volume em modo rootless).

## Detalhes Principais de Configuração

### Escalabilidade Profissional e Suporte a IA

Esta configuração foi feita para simular um ambiente profissional, incluindo:

- **PostgreSQL (com pgvector)**: Substitui o SQLite para um gerenciamento de dados robusto. A extensão `pgvector` é **habilitada automaticamente** na inicialização via `init-db.sql`, permitindo que você use o Postgres como um Banco de Dados Vetorial para fluxos de IA/LLM (embeddings).
- **Redis e Modo Fila (Queue Mode)**: Separa a instância principal do n8n das execuções. O serviço `n8n-worker` lida com a carga de processamento, garantindo que o editor continue rápido mesmo sob carga pesada.
- **Ferramentas de Gestão (Navegador)**: 
    - **pgAdmin**: Gerencie o Postgres pelo navegador em [http://localhost:5050](http://localhost:5050).
    - **Redis Commander**: Gerencie o Redis pelo navegador em [http://localhost:8081](http://localhost:8081).

### 💡 Entendendo o Acesso e Segurança

Para quem está começando, é fundamental entender que existem duas "camadas" diferentes de acesso:

1.  **Interface de Gestão (O Software)**: Ao abrir o pgAdmin no navegador, você está entrando na *ferramenta de gestão*. Use as credenciais padrão `admin@admin.com` / `admin`.
2.  **Serviços de Banco (Os Dados)**: Uma vez dentro da ferramenta (ou ao configurar um nó no n8n), você se conecta ao *banco de dados real* usando as credenciais definidas no seu `.env` (ex: `POSTGRES_USER=seu_usuario_do_banco`).


> [!NOTE]
> **Conexão via Windows**: Se usar uma ferramenta desktop (como DBeaver), use `localhost` e Porta `5433`.
> **Conexão dentro do n8n**: Ao configurar um nó de Postgres no n8n, use Host `postgres` e Porta `5432`.

### URL de Redirecionamento OAuth (HTTPS)

O `docker-compose.yaml` inclui duas variáveis de ambiente críticas:

| Variável | Propósito |
|----------|-----------|
| `WEBHOOK_URL` | Define a URL base para webhooks |
| `N8N_EDITOR_BASE_URL` | Define a URL base para o editor do n8n, garantindo que as URLs de Redirecionamento OAuth sejam geradas com `https://` |

Ambas usam a variável `URL` do `.env`, que **deve** começar com `https://`. Isso garante que, ao configurar credenciais OAuth (por exemplo, Gmail), a URL de redirecionamento será:

```
https://seu-dominio.ngrok-free.dev/rest/oauth2-credential/callback
```

Esta URL é o que você precisa adicionar como um **URI de redirecionamento autorizado** no Google Cloud Console (ou em qualquer provedor OAuth).

## Rodando a Aplicação

```bash
docker-compose up -d
```

## Acessando o n8n

Após iniciar os serviços, acesse o n8n navegando para o seu domínio Ngrok no navegador:

```
https://seu-dominio.ngrok-free.dev
```

## Parando a Aplicação

```bash
docker-compose down
```

> **DICA DE ESTABILIDADE:** Se você encontrar problemas de conexão ou erros de configuração (como `ERR_NGROK_9034`), sempre use `docker-compose down` para parar e remover totalmente os containers e redes virtuais antes de iniciá-los novamente. Isso garante um estado limio para a configuração dinâmica.

### Renomeação de Pasta e Portabilidade

Por padrão, o Docker Compose usa o **nome da pasta** como o nome do projeto. Se você renomear a pasta (por exemplo, de `n8n-ngrok` para `n8n-ngrok-https`), o Docker pode perder o rastro dos volumes existentes ou procurar arquivos no caminho errado.

Para evitar isso, este projeto fixa o nome do projeto no arquivo `.env`:
- `COMPOSE_PROJECT_NAME=n8n-ngrok`

Isso garante que, mesmo que você renomeie a pasta no seu computador ou se outra pessoa clonar o repositório com um nome diferente, o Docker sempre o tratará como o mesmo projeto (`n8n-ngrok`), mantendo seu banco de dados e configurações seguros.

## ⚠️ Suporte a Python no Nó Code

A imagem Docker padrão do n8n é baseada no **Alpine Linux**, uma distribuição Linux leve projetada para ser o mais minimalista possível. Por causa disso, o **Python não está incluído** na imagem — o Alpine traz apenas o básico necessário para rodar o n8n (Node.js). O nó Code roda **JavaScript (Node.js)** nativamente, o que é suficiente para a maioria dos fluxos de trabalho de automação.

Se você precisar de suporte a Python no nó Code, deve construir uma **imagem Docker personalizada**:

```dockerfile
FROM docker.n8n.io/n8nio/n8n

USER root
RUN apk add --no-cache python3 py3-pip
USER node
```

Em seguida, atualize o `docker-compose.yaml` para construir a partir do Dockerfile em vez de baixar a imagem:

```yaml
n8n:
    container_name: n8n
    build: .
    # image: docker.n8n.io/n8nio/n8n  # comente esta linha
```

> [!NOTE]
> O JavaScript é a linguagem mais utilizada no nó Code do n8n e é recomendada para a maioria dos casos de uso.
