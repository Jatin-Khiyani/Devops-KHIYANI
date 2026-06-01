## httpd

In this part, I set up a apache reverse proxy to access springboot application. 


### Terminal Comands

1. Get httpd.conf

```
docker run --rm httpd:2.4 cat /usr/local/apache2/conf/httpd.conf > httpd.conf
```

2. add this code to the end of the httpd.conf file 

'''
<VirtualHost *:80>
    ProxyPreserveHost On
    ProxyPass / http://${BACKEND_HOST}:8080/
    ProxyPassReverse / http://${BACKEND_HOST}:8080/
</VirtualHost>
'''

3. Build and run docker 

```
docker build -t apache-reverse-proxy .   
```

```
docker run --rm -p 80:80 \                                    
  -e BACKEND_HOST=host.docker.internal \
  apache-reverse-proxy
  ```

  ### Questions

  1. **Q 1.5 1-5 Why do we need a reverse proxy?**

    Reverse proxy is useful as it provides only one entrance to the application and all other ports that might have secure information are hidden. Backend and frontend can be securly seperated

