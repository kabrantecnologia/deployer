# Repositório de Deploy (Deployer)

Este repositório centraliza e automatiza o processo de deploy para as aplicações frontend da Kabra. Ele utiliza GitHub Actions para construir imagens Docker, publicá-las no GitHub Container Registry (GHCR) e, finalmente, implantá-las em um servidor de produção/staging via SSH.

## Visão Geral do Processo

O fluxo de trabalho é o seguinte:

1.  **Acionamento Manual**: Um usuário aciona o workflow através da aba "Actions" no GitHub, escolhendo qual projeto e branch deseja implantar.
2.  **Build da Imagem**: O GitHub Actions faz o checkout do código-fonte da aplicação, utiliza o `Dockerfile` específico do projeto (localizado neste repositório) para construir uma imagem Docker.
3.  **Publicação da Imagem**: A imagem recém-construída é tagueada e enviada para o GitHub Container Registry (GHCR).
4.  **Deploy no Servidor**: O workflow se conecta ao servidor via SSH e executa um comando para baixar a nova imagem e reiniciar o serviço usando o `docker-compose.yml` correspondente.

---

## Pré-requisitos

Antes de usar este sistema, garanta que o seguinte está configurado:

### 1. Segredos do Repositório GitHub

Vá para `Settings > Secrets and variables > Actions` no seu repositório `deployer` e configure os seguintes segredos:

-   `GH_PAT`: Um **GitHub Personal Access Token (classic)** com escopo de permissão `repo`. Isso é necessário para que o Actions possa clonar seus repositórios de aplicação privados.
-   `SERVER_HOST`: O endereço IP ou domínio do servidor onde o deploy será feito.
-   `SERVER_USER`: O nome de usuário para a conexão SSH com o servidor (ex: `joaohenrique`).
-   `SERVER_SSH_KEY`: A chave SSH **privada** (conteúdo do arquivo `~/.ssh/id_rsa`) que tem acesso ao servidor. Isso permite que o Actions se conecte sem senha.

### 2. Configuração do Servidor

O servidor de destino deve ter:

-   Docker e Docker Compose instalados.
-   Uma instância do Traefik (ou outro proxy reverso) rodando para gerenciar o tráfego e os certificados SSL.
-   A chave SSH **pública** correspondente à `SERVER_SSH_KEY` deve estar no arquivo `~/.ssh/authorized_keys` do servidor.

---

## Como Adicionar um Novo Projeto para Deploy

Siga estes passos para adicionar uma nova aplicação (ex: `meu-novo-app-frontend`):

**1. Crie o Diretório de Configuração**

Dentro deste repositório `deployer`, crie um diretório com o nome exato do repositório do seu projeto:

```bash
mkdir meu-novo-app-frontend
```

**2. Adicione o `Dockerfile`**

Dentro do diretório recém-criado (`meu-novo-app-frontend/`), adicione um arquivo `Dockerfile`. Para a maioria dos projetos Vue/React/Angular, você pode usar este template:

```Dockerfile
# Estágio 1: Build da Aplicação
FROM node:18-alpine AS build
WORKDIR /app
COPY package.json package-lock.json* ./
RUN npm install
COPY . .
RUN npm run build

# Estágio 2: Servidor de Produção
FROM nginx:stable-alpine
COPY --from=build /app/dist /usr/share/nginx/html
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

**3. Adicione o `docker-compose.yml`**

No mesmo diretório, adicione um arquivo `docker-compose.yml`. Este arquivo será usado no servidor para rodar o contêiner.

**Importante:** Altere o `container_name` e a regra de `Host` para corresponder ao seu novo projeto.

```yaml
name: meu-novo-app

services:
  frontend:
    image: ${DOCKER_IMAGE} # Esta variável é preenchida pelo GitHub Actions
    container_name: meu-novo-app-frontend # Altere aqui
    restart: unless-stopped
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.meu-novo-app.rule=Host(`app-staging-meu-novo-app.kabran.com.br`)" # Altere o subdomínio aqui
      - "traefik.http.routers.meu-novo-app.entrypoints=web,websecure"
      - "traefik.http.routers.meu-novo-app.tls.certresolver=cloudflare"
      - "traefik.http.services.meu-novo-app.loadbalancer.server.port=80"
    networks:
      - traefik-proxy

networks:
  traefik-proxy:
    external: true
```

**4. Atualize o Workflow**

Abra o arquivo `.github/workflows/main-deploy.yml` e adicione o nome do seu novo projeto à lista de `options`:

```yaml
on:
  workflow_dispatch:
    inputs:
      project_name:
        # ...
        options:
          - integra-frontend
          - compliance-frontend
          - tricket-frontend
          - meu-novo-app-frontend # Adicione a nova opção aqui
```

**5. Envie as Alterações**

Faça o commit e push de todos os novos arquivos para o repositório `deployer`:

```bash
git add .
git commit -m "feat: add configuration for meu-novo-app-frontend"
git push origin main
```

---

## Como Fazer o Deploy de um Projeto

Com tudo configurado, o deploy é o passo mais simples:

1.  **Prepare o Servidor (Apenas na primeira vez)**: Crie o diretório do projeto no servidor e copie o `docker-compose.yml` para lá.

    ```bash
    # No seu ambiente local/servidor, onde o repo deployer está clonado
    cd /home/joaohenrique/workspaces/projects
    
    # Crie o diretório de destino no servidor
    mkdir meu-novo-app-frontend
    
    # Atualize o repo deployer e copie o compose
    cd deployer && git pull && cd ..
    cp deployer/meu-novo-app-frontend/docker-compose.yml meu-novo-app-frontend/
    ```

2.  **Execute o Workflow no GitHub**:
    *   Vá para a aba **"Actions"** do repositório `deployer`.
    *   Selecione o workflow **"Manual Deployer"** na barra lateral.
    *   Clique no botão **"Run workflow"**.
    *   Selecione o projeto que deseja implantar (ex: `meu-novo-app-frontend`).
    *   (Opcional) Especifique o branch do código-fonte da aplicação. O padrão é `main`.
    *   Clique no botão verde **"Run workflow"** para iniciar o deploy.

Você poderá acompanhar o progresso em tempo real na própria página do Actions.
