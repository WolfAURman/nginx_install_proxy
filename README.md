## Создание обратного прокси для ssl и нескольких доменов для всего

## Благодарности:
- https://antoshabrain.blogspot.com/2021/09/nginx-proxy-manager-docker-compose.html
- https://github.com/NginxProxyManager/nginx-proxy-manager/issues/2036#issuecomment-1486149750

## Доп. инфа:
- https://github.com/nextcloud/all-in-one/blob/main/reverse-proxy.md
  
### Ведение в курс дела
Nginx Proxy Manager позволяет создать промежуточное звено между трафиком пользователей и конечными точками на вашем хосте. Вы можете увидеть это на картинке ниже:
![1](https://raw.githubusercontent.com/WolfAURman/nginx_install_proxy/main/media/1.png)
В этом и заключается прелесть этого способа. На одной машине вы можете расположить несколько контейнеров которым будет необходим 80 порт, но вам не нужно будет что-то "костылить" или изменять.
За вас будет это делать используя переадресацию nginx при переходе на необходимый домен. У вас всегда будут красивые и адекватные домены, и самое главное ssl.

### что нам нужно:
- прямые руки
- podman
- nextcloud
- Nginx Proxy Manager
- duckdns

1. Идём на duckdns.org и создаём там нужные нам домены:
![image](https://github.com/WolfAURman/nginx_install_proxy/assets/93985232/7220c2cf-dbe2-43eb-9106-7556cf8772d6)

2. После создания доменов нам необходимо получить образы контейнеров для Nginx Proxy Manager и nextcloud. Пишем:

- Nginx Proxy Manager
```
podman pull jc21/nginx-proxy-manager
```

- NextCloud
```
podman pull nextcloud
```

3. Создаём в домашней папке пользователя папки:
```
mkdir nextcloud && cd nextcloud && mkdir config && mkdir data && cd ~/ && mkdir nginx-proxy-manager && cd nginx-proxy-manager && mkdir data && mkdir letsencrypt && cd ~/
```

Этой командой создаём папку nextcloud, в ней создаём две папки в виде config и data. После создаём папку nginx-proxy-manager и в ней папку data и letsencrypt.

4. Создаём сеть:

```
podman network create nginx
```

5. Создаём контейнер с обратным прокси:

```
podman run --detach \
  --publish 80:80/tcp \
  --publish 443:443/tcp \
  --publish 45344:81/tcp \
  --name nginx-proxy-manager \
  --cpus=1 \
  --memory=256M \
  --volume /home/ikrell/nginx-proxy-manager/data:/data:rw \
  --volume /home/ikrell/nginx-proxy-manager/letsencrypt:/etc/letsencrypt:rw \
  --network nginx \
  jc21/nginx-proxy-manager:latest
```

6. После чего переходим на http://ip_вашего_сервера:45344 и настраиваем панель в виде изменения логина и пароля. Советую вписывать валидную почту.

После этого вы увидите такого вида панель:
![image](https://raw.githubusercontent.com/WolfAURman/nginx_install_proxy/main/media/Screenshot%20from%202023-07-10%2011-58-58.png)

7. Нам необходимо нажать на Proxy Hosts после чего нажать на add proxy host.

8. После чего вводим такого содержания наполнение и сохраняем.
![image](https://raw.githubusercontent.com/WolfAURman/nginx_install_proxy/main/media/Screenshot%20from%202023-07-10%2011-57-31.png)

9. После этого если всё исправно и не появилось ошибок, нажимаем на три точки и добавляем ssl сертификат:
![image](https://raw.githubusercontent.com/WolfAURman/nginx_install_proxy/main/media/2.png)

Если не возникло проблем - продолжаем.

10. После чего выходим из панели и удаляем контейнер командой:
```
podman rm nginx-proxy-manager -f
```

11. После этого создаём снова, но с уже открытыми портами 443 и 80, ибо третий нам без надобности.

```
podman run --detach \
  --publish 80:80/tcp \
  --publish 443:443/tcp \
  --name nginx-proxy-manager \
  --cpus=1 \
  --memory=256M \
  --volume /home/ikrell/nginx-proxy-manager/data:/data:rw \
  --volume /home/ikrell/nginx-proxy-manager/letsencrypt:/etc/letsencrypt:rw \
  --network nginx \
  jc21/nginx-proxy-manager:latest
```

12. После чего переходим по нашему ip или домену уже в панель с ssl сертификатом по https.

13. Теперь нам необходимо создать nextcloud контейнер, без публичных портов и.т.д:

```
podman run --detach \
  --name=nextcloud \
  --cpus=1 \
  --memory=256M \
  --volume=/home/ikrell/nextcloud/data:/var/www/html/data:rw \
  --volume=/home/ikrell/nextcloud/config:/var/www/html/config:rw \
  --network podman2 \
  nextcloud:latest
```

14. После этого мы узнаём ip этого контейнера для добавления его в наш прокси:
```
podman container inspect nextcloud | grep IPAddress
```
На выходе получаем 10.89.1.11 или что-то похожего содержания.

15. Создаём новый домен на тот же ip вашей машины, и переходим снова в панель прокси, добавляем ваш ip и новый домен, а так же аналогично как и ранее генерируем ssl сертификат:
![image](https://raw.githubusercontent.com/WolfAURman/nginx_install_proxy/main/media/Screenshot%20from%202023-07-10%2011-58-15.png)
![image](https://raw.githubusercontent.com/WolfAURman/nginx_install_proxy/main/media/2.png)

16. После чего сохраняем, и подключаемся по новому домену к вашему серверу уже с ssl сертификатом и https.

## FAQ
В: Почему для панели мы прописали localhost, но не прописали для других?

О: Потому что ip может измениться, и вы потом рискуете не попасть в панель. Это можно исправить, расскажу об этом в последующих обновлениях мануала.

В: Так почему бы нам везде не прописать localhost?

О: Потому что потом будет цикличная переадресация из-за одинаковых портов и всё сломается, обязательно прописывать ip и порт.

В: Зачем мы создаём новую сеть?

О: Для внутренней связи контейнеров, что б не было обходов извне.

В: Зачем мы пересоздавали контейнер?

О: Для первичной настройки, дабы у нас не висели никакие лишние порты для большей безопасности извне.
