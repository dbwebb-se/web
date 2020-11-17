---
author: moc 
category:
    - devops
    - monitoring
revision:
    "2020-11-17": (A, moc) Skapad inför HT2020.
...
Övervaka nginx med Prometheus och Grafana {#intro}
=====================================================

I denna artikel skall vi gå igenom hur man kan övervaka trafiken till våran nginx server med hjälp av Prometheus och Grafana.

<!--more-->

Förutsättningar {#forutsattningar}
-------------------------------------
Ni har jobbat igenom installerat [Complete Node Exporter Mastery with Prometheus](https://devconnected.com/complete-node-exporter-mastery-with-prometheus/) och installerat Grafana samt Prometheus.


Aktivera stub_status i nginx {#stub_status}
---------------------------------------------------------
Om ni har installerat nginx via `apt-get` kommer den redan med modulen [`ngx_http_stub_status_module`](http://nginx.org/en/docs/http/ngx_http_stub_status_module.html) som vi nu behöver aktivera för att den [prometheus exportör](https://github.com/nginxinc/nginx-prometheus-exporter) vi senare skall använda oss utav skall fungera.


Så i min konfigurationsfil för nginx, lägger jag till en ny `endpoint` och startar `stub_status`:

```
server {
    listen 443 ssl;
...
    location /metrics {
        stub_status on;
    }
...
```

Vi kan nu testa våran konfigurationsfil `nginx -c /etc/nginx/nginx.conf -t` för att försäkra oss att våra ändringar stämmer. Om allt går igenom testarna kan vi starta om webbservern `sudo service nginx reload`.


Går vi nu in på `https://<domännamn>/metrics` så kan vi se att det skrivs ut information liknande:

```
Active connections: 5 
server accepts handled requests
 5 5 55 
Reading: 0 Writing: 1 Waiting: 4 
```

Using the Prometheus exporter {#prom_exporter}
---------------------------------------------------------
Som vi såg innan kan vi redan se de mätvärden vi är ute efter men, för att göra det läsbart för Prometheus behöver vi skapa en ny endpoint. Ett av verktygen vi kan använda för att göra tillgängligt är `nginx-prometheus-exporter` som vi pratade om tidigare. Det enklaste sättet att starta exportören är via Docker, men om ni hellre vill installera den [som en binary](https://github.com/nginxinc/nginx-prometheus-exporter#running-the-exporter-binary) kan man också göra det.

Docker kommandot jag använder mig utav:

```bash
$ docker run \
    --restart always \
    -p 9113:9113 \
    nginx/nginx-prometheus-exporter:0.4.2 \
    -nginx.scrape-uri=https://<domännamn>/metrics \
    -nginx.retries=10 \
    -nginx.ssl-verify=false \
    -web.telemetry-path=/prometheus
```

Konfigurera Prometheus {#configure_prom}
---------------------------------------------------------
Nästa steg för att få igång detta är att konfigurera prometheus genom att lägga till ett jobb i [prometheus.yml](https://devconnected.com/complete-node-exporter-mastery-with-prometheus/#b_Installing_Prometheus):

```yml
scrape_configs: 
  - job_name: nginx
    metrics_path: /prometheus
    scrape_interval: 2m
    static_configs: 
    - targets: ["domännamn:9113"]
```

Starta om prometheus, för mig räckte det att skicka en POST `curl -X POST http://<monitor ip>:9090/-/reload` men fungerar inte det kan ni testa `sudo killall -HUP prometheus` som skickar en `SIGHUP` signal till Prometheus processen och laddar om sina konfigurationer.

När allt har körts bör du kunna se om prometheus är korrekt konfigurerad genom att besöka `/targets`. Detta ska visa status "UP" bredvid nginx jobbet.


Konfigurera Grafana {#configure_graf}
---------------------------------------------------------
När du har konfigurerat Grafana att ansluta till Prometheus-databasen kan du ställa in dina *dashboards*. I det här fallet är jag intresserad efter ett diagram som visar den totala mängden anslutningar som har gjorts under en viss tid. Detta kan vi göra genom att titta på `nginx_http_requests_total`.

Eftersom `nginx_http_requests_total` visar den totala mängden förfrågningar, måste vi använda funktionen `delta()` för att titta på skillnaden över en viss tid. Om vi till exempel vill beräkna det totala antalet förfrågningar över en tidsperiod, kan du använda följande för att se mängden requests per två minuter:

```
delta(nginx_http_requests_total{job="nginx"}[2m])
```

Nu skall vi ha en graf liknande:

[FIGURE src="https://dimitr.im/static/87d93f8565b6ee6b8e70660e49fdafe5/b1718/grafana-nginx-requests.png"]


Avslutningsvis {#avslutningsvis}
-------------------------------------
Och där var det klart, nu har vi en fin *dashboard* för nginx. Tyvärr behöver man ha Nginx Plus för att få ut mer intressant data som hur många 4xx/5xx request man får in. Nu har vi inte Plus versionen och får nöja oss med att kunna se att servern är igång och hur många requests servern har fått.

Glöm inte att också öppna portarna i era *security groups*.