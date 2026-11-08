###################################
# Target: docker
###################################
FROM base AS docker

COPY --from=build --chown=nginx:nginx /src/dist/browser /usr/share/nginx/html
COPY --from=build /src/nginx-custom.conf /etc/nginx/nginx.conf

RUN mkdir -p /var/cache/nginx /var/run \
    && chown -R nginx:nginx /var/cache/nginx /var/run/

USER nginx

CMD ["/bin/sh",  "-c",  "envsubst '${ENVIRONMENT}' < /etc/nginx/nginx.conf > /tmp/nginx.conf && envsubst < /usr/share/nginx/html/assets/environment-settings.template.json > /usr/share/nginx/html/assets/environment-settings.json && exec nginx -c /tmp/nginx.conf -g 'daemon off;'"]
