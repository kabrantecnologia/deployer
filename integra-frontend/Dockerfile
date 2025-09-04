# Estágio 1: Build da Aplicação
FROM node:18-alpine AS build

# Define o diretório de trabalho
WORKDIR /app

# Copia os arquivos de dependência
COPY package.json package-lock.json* ./

# Instala as dependências
RUN npm install

# Copia o restante do código da aplicação
COPY . .

# Executa o build da aplicação
RUN npm run build

# Estágio 2: Servidor de Produção
FROM nginx:stable-alpine

# Copia os arquivos estáticos gerados no estágio de build
COPY --from=build /app/dist /usr/share/nginx/html

# Copia uma configuração customizada do Nginx (opcional, mas recomendado)
# COPY nginx.conf /etc/nginx/conf.d/default.conf

# Expõe a porta 80
EXPOSE 80

# Comando para iniciar o Nginx
CMD ["nginx", "-g", "daemon off;"]
