# Choose an appropriate base image
FROM httpd:2.4

# Copy your index.html to the appropriate directory in your image
COPY ./index.html /usr/local/apache2/htdocs/
# Copy conf file
COPY ./httpd.conf /usr/local/apache2/conf/httpd.conf
