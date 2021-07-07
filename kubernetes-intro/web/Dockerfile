FROM nginx:stable-alpine
RUN apk add --no-cache shadow \
    && usermod -u 1001 nginx \
    && groupmod -g 1001 nginx \
    && ln -s /usr/share/nginx/html /app \ 
    && chown -R nginx:nginx /var/cache/nginx \ 
    && sed -i 's/\/var\/run\//\/tmp\//' /etc/nginx/nginx.conf \
    && sed -i 's/\(listen[[:space:]]*\)80;/\18000;/' /etc/nginx/conf.d/default.conf \
    && sed -i 's/\(listen[[:space:]]*\[::\]:\)80;/\18000;/' /etc/nginx/conf.d/default.conf \
    && sed -i '/^user[[:space:]]*nginx;$/d' /etc/nginx/nginx.conf
EXPOSE 8000
USER 1001:1001
