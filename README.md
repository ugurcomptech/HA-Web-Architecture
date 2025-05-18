# HA Web Architecture (Yüksek Erişilebilirlikli Web Mimarisi)
Bu döküman, yüksek erişilebilirliğe sahip (High Availability - HA) bir web mimarisinin nasıl inşa edileceğini adım adım açıklamaktadır. Sistemin amacı, kesintisiz hizmet sağlayarak olası donanım veya yazılım hatalarında bile web uygulamalarının çalışmaya devam etmesini sağlamaktır.

Dökümanda, bu yapının temelini oluşturan bileşenlerin kurulumu ve yapılandırması detaylı bir şekilde ele alınmaktadır. Kullanılan ana servisler şunlardır:

## 1. Galera Cluster (MariaDB için)
Galera, veritabanı yüksek erişilebilirlik ve senkron replikasyon için kullanılan bir çözümdür. MariaDB ile birlikte kullanıldığında, birden fazla veritabanı sunucusu üzerinde veri eşzamanlı olarak tutulur. Bu sayede herhangi bir veritabanı düğümünün çökmesi durumunda, diğer düğümler kesintisiz olarak hizmet vermeye devam edebilir.

## 2. GlusterFS (Dağıtık Dosya Sistemi)
GlusterFS, birden fazla sunucu üzerinde veri paylaşımı ve senkronizasyon sağlamak için kullanılan açık kaynaklı bir dağıtık dosya sistemidir. Web sunucuları arasında dosya bütünlüğünü ve eşitliği koruyarak, tüm sunucuların aynı içerikle hizmet verebilmesini sağlar.

## 3. SafeLine (Firewall / Güvenlik Katmanı)
SafeLine, gelen ve giden trafiği kontrol ederek sistemin güvenliğini artırmak için kullanılan bir güvenlik çözümüdür. Web mimarisinde saldırılara karşı ilk savunma hattını oluşturur. Ayrıca DDoS, brute-force gibi saldırılara karşı koruma sağlar.

## 4. HAProxy (Yük Dengeleyici)
HAProxy, istemcilerden gelen trafiği arka planda çalışan birden fazla web sunucusu arasında dağıtarak hem yük dengelemesi hem de yüksek erişilebilirlik sağlar. Aktif-pasif veya aktif-aktif yapılandırmalarla çalışarak sistemin ayakta kalmasını garanti altına alır.

## Yapıya Ait Genel Diyagram

```
graph TD

%% Stil Tanımları
classDef waf fill:#ff6666,stroke:#333,stroke-width:2px;
classDef haproxy fill:#ffcc99,stroke:#333,stroke-width:2px;
classDef nginx fill:#99ccff,stroke:#333,stroke-width:2px;
classDef galera fill:#ccffcc,stroke:#333,stroke-width:2px;
classDef gluster fill:#ff9999,stroke:#333,stroke-width:2px;

%% WAF Katmanı
subgraph WAF_Katmanı["WAF Katmanı (SafeLine)"]
    WAF["WAF | IP: 10.0.0.1 | Güvenlik Filtreleme"]:::waf
end

%% HAProxy Load Balancer Katmanı
subgraph Yük_Dengeleyici_Katmanı["Yük Dengeleyici (HAProxy)"]
    HAProxy["HAProxy | IP: 10.0.0.2 | Port: 80/443 | Yük Dengeleme: Round-Robin | SSL Termination"]:::haproxy
end

%% Web Katmanı
subgraph Web_Katmanı["Web Katmanı (NGINX)"]
    NGINX1["NGINX1 | IP: 10.0.0.3 | Port: 80/443 | Statik İçerik Sunumu | Reverse Proxy"]:::nginx
    NGINX2["NGINX2 | IP: 10.0.0.4 | Port: 80/443 | Statik İçerik Sunumu | Reverse Proxy"]:::nginx
end

%% Veritabanı Katmanı
subgraph Veritabanı_Katmanı["Veritabanı Katmanı (MariaDB Galera Cluster)"]
    Galera1["Galera Node1 | IP: 10.0.0.5 | Port: 3306 | Multi-Master Replikasyon"]:::galera
    Galera2["Galera Node2 | IP: 10.0.0.6 | Port: 3306 | Multi-Master Replikasyon"]:::galera
    Galera3["Galera Node3 | IP: 10.0.0.7 | Port: 3306 | Multi-Master Replikasyon"]:::galera
end

%% Dosya Sistemi Katmanı
subgraph Dosya_Sistemi_Katmanı["Dağıtık Dosya Sistemi (GlusterFS)"]
    Gluster1["GlusterFS Node1 | IP: 10.0.0.3 | Replicated Volume | Brick: /data/gluster"]:::gluster
    Gluster2["GlusterFS Node2 | IP: 10.0.0.4 | Replicated Volume | Brick: /data/gluster"]:::gluster
end

%% Bağlantılar
%% WAF -> HAProxy
WAF -->|HTTP/HTTPS| HAProxy

%% HAProxy ile NGINX Bağlantıları
HAProxy -->|HTTP/HTTPS| NGINX1
HAProxy -->|HTTP/HTTPS| NGINX2

%% NGINX ile Galera Bağlantıları
NGINX1 -->|MySQL:3306| Galera1
NGINX1 -->|MySQL:3306| Galera3
NGINX2 -->|MySQL:3306| Galera2
NGINX2 -->|MySQL:3306| Galera3

%% NGINX ile GlusterFS Bağlantıları
NGINX1 -->|NFS/Gluster| Gluster1
NGINX2 -->|NFS/Gluster| Gluster2

%% Galera Cluster Replikasyon Bağlantıları
Galera1 <-->|WSREP Replikasyon| Galera2
Galera2 <-->|WSREP Replikasyon| Galera3
Galera3 <-->|WSREP Replikasyon| Galera1

%% GlusterFS Replikasyon Bağlantıları
Gluster1 <-->|Gluster Replikasyon| Gluster2

%% Açıklamalar
note1["Notlar: İstekler önce WAF (SafeLine) tarafından filtrelenir, sonra HAProxy yük dengeleme yapar. NGINX sunucuları statik içerik sunar ve veritabanı ile dosya sistemi katmanlarına erişir. Galera Cluster multi-master replikasyon, GlusterFS paylaşılan dosya sistemi sağlar."]

```
