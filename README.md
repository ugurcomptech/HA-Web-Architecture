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

```mermaid
graph TD

%% Stil Tanımları
classDef waf fill:#ff6666,stroke:#333,stroke-width:2px;
classDef haproxy fill:#ffcc99,stroke:#333,stroke-width:2px;
classDef nginx fill:#99ccff,stroke:#333,stroke-width:2px;
classDef galera fill:#ccffcc,stroke:#333,stroke-width:2px;
classDef gluster fill:#ff9999,stroke:#333,stroke-width:2px;

%% WAF Katmanı
subgraph WAF_Katmanı["WAF Katmanı"]
    WAF["SafeLine WAF\n10.0.0.1"]:::waf
end

%% HAProxy Load Balancer Katmanı
subgraph Yük_Dengeleyici_Katmanı["Yük Dengeleyici"]
    HAProxy["HAProxy\n10.0.0.2\nPort: 80/443"]:::haproxy
end

%% Web Katmanı
subgraph Web_Katmanı["Web Katmanı"]
    NGINX1["NGINX1\n10.0.0.3"]:::nginx
    NGINX2["NGINX2\n10.0.0.4"]:::nginx
end

%% Veritabanı Katmanı
subgraph Veritabanı_Katmanı["Veritabanı Katmanı"]
    Galera1["Galera1\n10.0.0.5"]:::galera
    Galera2["Galera2\n10.0.0.6"]:::galera
    Galera3["Galera3\n10.0.0.7"]:::galera
end

%% Dosya Sistemi Katmanı
subgraph Dosya_Sistemi_Katmanı["Dosya Sistemi Katmanı"]
    Gluster1["Gluster1\n10.0.0.3"]:::gluster
    Gluster2["Gluster2\n10.0.0.4"]:::gluster
end

%% Bağlantılar
WAF -->|HTTP/HTTPS| HAProxy
HAProxy -->|HTTP/HTTPS| NGINX1
HAProxy -->|HTTP/HTTPS| NGINX2

NGINX1 -->|MySQL| Galera1
NGINX1 -->|MySQL| Galera3
NGINX2 -->|MySQL| Galera2
NGINX2 -->|MySQL| Galera3

NGINX1 -->|GlusterFS| Gluster1
NGINX2 -->|GlusterFS| Gluster2

Galera1 <-->|Replikasyon| Galera2
Galera2 <-->|Replikasyon| Galera3
Galera3 <-->|Replikasyon| Galera1

Gluster1 <-->|Replikasyon| Gluster2

%% Açıklamalar
note1["📝 Notlar:
- İstekler ilk olarak SafeLine WAF (10.0.0.1) tarafından filtrelenir.
- HAProxy (10.0.0.2) istekleri Round-Robin ile NGINX sunucularına yönlendirir.
- NGINX sunucuları (10.0.0.3 / 10.0.0.4) statik içerik sunar ve hem veritabanına (Galera) hem dosya sistemine (GlusterFS) erişir.
- Galera Cluster multi-master replikasyon destekler.
- GlusterFS düğümleri dosya sistemini yedekli paylaşır."]


```
