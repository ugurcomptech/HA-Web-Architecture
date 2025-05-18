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

%% WAF Katmanı
subgraph WAF["WAF Katmanı"]
    WAF1["SafeLine\n10.0.0.1"]
end

%% HAProxy Katmanı
subgraph LB["Yük Dengeleyici"]
    HAProxy["HAProxy\n10.0.0.2"]
end

%% Web Katmanı
subgraph Web["Web Katmanı"]
    NGINX1["NGINX-1\n10.0.0.3"]
    NGINX2["NGINX-2\n10.0.0.4"]
end

%% Veritabanı Katmanı
subgraph DB["Veritabanı Katmanı"]
    Galera1["Galera-1\n10.0.0.5"]
    Galera2["Galera-2\n10.0.0.6"]
    Galera3["Galera-3\n10.0.0.7"]
end

%% Dosya Sistemi Katmanı
subgraph FS["Dosya Sistemi"]
    Gluster1["Gluster-1\n10.0.0.3"]
    Gluster2["Gluster-2\n10.0.0.4"]
end

%% Bağlantılar
WAF1 --> HAProxy
HAProxy --> NGINX1
HAProxy --> NGINX2

NGINX1 --> Galera1
NGINX1 --> Galera3
NGINX2 --> Galera2
NGINX2 --> Galera3

NGINX1 --> Gluster1
NGINX2 --> Gluster2

Galera1 <-->|Replikasyon| Galera2
Galera2 <-->|Replikasyon| Galera3
Galera3 <-->|Replikasyon| Galera1

Gluster1 <-->|Replikasyon| Gluster2


```
