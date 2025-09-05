# Usar uma imagem base do Nginx para servir arquivos estáticos
FROM nginx:stable-alpine
# Copiar todo o conteúdo do repositório para o diretório web do Nginx
COPY . /usr/share/nginx/html
# Expor a porta 80
EXPOSE 80
# Iniciar o Nginx
CMD [\"nginx\", \"-g\", \"daemon off;\"]
