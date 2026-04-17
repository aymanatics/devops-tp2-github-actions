FROM httpd:2.4-alpine

COPY index.html /usr/local/apache2/htdocs/index.html
COPY reverse-proxy.conf /usr/local/apache2/conf/extra/reverse-proxy.conf

RUN echo "LoadModule proxy_module modules/mod_proxy.so" >> /usr/local/apache2/conf/httpd.conf \
 && echo "LoadModule proxy_http_module modules/mod_proxy_http.so" >> /usr/local/apache2/conf/httpd.conf \
 && echo "Include conf/extra/reverse-proxy.conf" >> /usr/local/apache2/conf/httpd.conf
