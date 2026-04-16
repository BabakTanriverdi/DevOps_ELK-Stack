# Hands-on ELK Stack: Elasticsearch, Logstash, Kibana ve Metricbeat Kurulumu

> **Amaç:** Bu uygulamalı eğitim, ELK Stack bileşenlerinin (Elasticsearch, Logstash, Kibana, Metricbeat) Ubuntu 24.04 üzerinde kurulumunu, yapılandırılmasını ve kullanımını kapsar.

---

## Öğrenme Hedefleri

Bu eğitimin sonunda öğrenciler:

- ELK Stack bileşenlerini ve aralarındaki ilişkiyi açıklayabilecek
- Elasticsearch, Logstash, Kibana ve Metricbeat'i Ubuntu üzerine kurabilecek
- Yapılandırma dosyalarını düzenleyerek bileşenleri birbirine bağlayabilecek
- Kibana arayüzü üzerinden log verilerini görselleştirebilecek

---

## ELK Stack Nedir?

**ELK Stack**, üç açık kaynaklı araçtan oluşan güçlü bir log yönetimi ve analiz platformudur:

| Bileşen | Görev | Varsayılan Port |
|---|---|---|
| **Elasticsearch** | Verilerin depolandığı ve sorgulandığı arama/analitik motoru | 9200 |
| **Logstash** | Log verilerini toplayan, dönüştüren ve Elasticsearch'e ileten pipeline | 5044 |
| **Kibana** | Elasticsearch verilerini görselleştiren web arayüzü | 5601 |
| **Metricbeat** | Sistem metriklerini toplayıp Elasticsearch'e gönderen hafif ajan | — |

---

## Kurulum Adımları

1. EC2 Instance Oluşturma
2. Java Kurulumu
3. Elasticsearch Kurulumu ve Yapılandırması
4. Metricbeat Kurulumu
5. Logstash Kurulumu ve Yapılandırması
6. Kibana Kurulumu ve Yapılandırması

---

## Adım 1 – EC2 Instance Oluşturma

### Instance Özellikleri

| Parametre | Değer |
|---|---|
| **AMI** | Ubuntu 24.04 LTS (64-bit x86) |
| **Instance Type** | t3a.large (minimum) |
| **Storage** | 16 GB |

> **Not:** Üretim (production) ortamlarında her bileşen için ayrı instance kullanılması önerilir. Bu eğitimde tüm bileşenler tek instance üzerine kurulacaktır.

### Security Group Yapılandırması

Aşağıdaki portları Security Group'a ekleyin:

| Port | Protokol | Kullanım |
|---|---|---|
| `22` | TCP | SSH erişimi |
| `5601` | TCP | Kibana web arayüzü |
| `9200` | TCP | Elasticsearch HTTP API |
| `9100` | TCP | Node Exporter (opsiyonel) |

> **Güvenlik ipucu:** Statik bir genel IP'niz varsa kaynak IP'yi kendi adresinizle sınırlandırın. Tüm trafiğe (`0.0.0.0/0`) izin vermekten kaçının.

### EC2'ye SSH ile Bağlanma

```bash
# Windows kullanıcıları için Git Bash veya PowerShell önerilir
ssh -i <pem-dosyasi>.pem ubuntu@<EC2-PUBLIC-IP>
```

Bağlandıktan sonra paket listesini güncelleyin:

```bash
# apt paket veritabanını günceller; mevcut paketlerin en son sürüm bilgilerini çeker
# Herhangi bir yükleme yapmaz, yalnızca listeyi yeniler
sudo apt-get update
```

---

## Adım 2 – Java Kurulumu

Elasticsearch ve Logstash, JVM (Java Virtual Machine) üzerinde çalışır; bu nedenle Java kurulumu zorunludur.

```bash
# OpenJDK'nın dağıtım tarafından önerilen varsayılan sürümünü kurar
sudo apt-get install default-jre
```

```bash
# Kurulumun başarılı olduğunu doğrular; sürüm bilgisini ekrana yazdırır
java -version
```

Beklenen çıktı: `openjdk version "21.x.x"` gibi bir satır.

---

## Adım 3 – Elasticsearch Kurulumu ve Yapılandırması

### Elasticsearch Nedir?

**Elasticsearch**, Apache Lucene tabanlı dağıtık bir arama ve analitik motorudur. JSON belgeleri depolar, indeksler ve çok hızlı tam metin araması yapar. ELK Stack'in veri deposu görevini üstlenir.

### 3.1 – GPG Anahtarı Ekleme

```bash
# Elastic deposunun GPG imza anahtarını indirir ve sisteme ekler
# -qO -   : sessiz modda indir, stdout'a yaz
# gpg --dearmor : ASCII zırhını ikili formata dönüştürür
# -o ... : çıktıyı keyrings klasörüne kaydeder (apt güven zinciri için)
wget -qO - https://artifacts.elastic.co/GPG-KEY-elasticsearch | sudo gpg --dearmor -o /usr/share/keyrings/elasticsearch-keyring.gpg
```

### 3.2 – HTTPS Desteği Ekleme

```bash
# Ubuntu'nun varsayılan kurulumunda apt, HTTPS üzerinden paket indirmeyi desteklemez
# Bu paket apt'ye HTTPS desteği kazandırır
sudo apt-get install apt-transport-https
```

### 3.3 – Elastic Deposunu Sisteme Tanıtma

```bash
# Elastic'in resmi apt deposunu kaynak listesine ekler
# signed-by: indirilen paketlerin bu GPG anahtarıyla doğrulanacağını belirtir
# tee: komutu hem /etc/apt/... dosyasına yazar hem de ekrana basar
echo "deb [signed-by=/usr/share/keyrings/elasticsearch-keyring.gpg] https://artifacts.elastic.co/packages/8.x/apt stable main" | sudo tee /etc/apt/sources.list.d/elastic-8.x.list
```

### 3.4 – Elasticsearch Kurulumu

```bash
# Paket listesini yeni depo ile günceller ve Elasticsearch'ü kurar
sudo apt-get update && sudo apt-get install elasticsearch
```

---

### 3.5 – Elasticsearch Yapılandırması

```bash
# Süper kullanıcıya geçer (yapılandırma dosyaları root yetkisi gerektirir)
sudo su

# Ana yapılandırma dosyasını vim ile açar
vim /etc/elasticsearch/elasticsearch.yml
```

Aşağıdaki satırların aktif (yorum işaretsiz) olduğundan emin olun:

```yaml
# ------------------------------------ Node ------------------------------------
node.name: node-1                          # Bu node'un adı; cluster içinde benzersiz olmalı

# ----------------------------------- Paths ------------------------------------
path.data: /var/lib/elasticsearch          # Verilerin saklanacağı dizin
path.logs: /var/log/elasticsearch          # Log dosyalarının yazılacağı dizin

# ---------------------------------- Network -----------------------------------
network.host: "localhost"                  # Hangi arayüzden dinleyeceği; dışa açmak için 0.0.0.0
http.port: 9200                            # HTTP API portu

# --------------------------------- Discovery ----------------------------------
cluster.initial_master_nodes: ["node-1"]   # Tek node'lu kurulumda bu node master olur
```

### 3.6 – Güvenlik Özelliklerini Devre Dışı Bırakma

Elasticsearch 8.x ile birlikte güvenlik özellikleri varsayılan olarak etkindir. Bu eğitimde HTTP üzerinden erişim için güvenliği devre dışı bırakın.

Aynı dosyada (`elasticsearch.yml`) otomatik oluşturulan güvenlik bloğunu yoruma alın:

```yaml
# --------------------------------- Security AUTO CONFIGURATION ----------------
# Aşağıdaki satırları # ile yoruma alın:
# xpack.security.enabled: true
# xpack.security.enrollment.enabled: true
# xpack.security.http.ssl:
#   enabled: true
#   keystore.path: certs/http.p12
# xpack.security.transport.ssl:
#   enabled: true
#   verification_mode: certificate
#   keystore.path: certs/transport.p12
```

Dosyanın **sonuna** aşağıdaki satırları ekleyin:

```yaml
# Güvenlik özelliklerini bu eğitim için devre dışı bırakır
xpack.security.enabled: false
xpack.security.enrollment.enabled: false
xpack.security.http.ssl.enabled: false
xpack.security.transport.ssl.enabled: false
```

### 3.7 – JVM Heap Boyutu Ayarlama

Elasticsearch, belleğin yaklaşık yarısını JVM heap'e ayırmanızı önerir. t3a.large instance'ı 8 GB RAM'e sahiptir; 4 GB heap uygun bir değerdir.

```bash
# JVM ayar dosyasını düzenler
vim /etc/elasticsearch/jvm.options
```

Aşağıdaki satırları aktif hale getirin (başındaki `#` işaretini kaldırın):

```
-Xms4g    # Başlangıç heap boyutu: 4 GB
-Xmx4g    # Maksimum heap boyutu: 4 GB (Xms ile aynı olması önerilir)
```

> **Neden eşit ayarlanır?** `Xms` ve `Xmx`'i eşit tutmak, JVM'nin heap'i genişletip daraltmasını önler ve daha öngörülü bir performans sağlar.

### 3.8 – Elasticsearch'ü Başlatma ve Doğrulama

```bash
# Elasticsearch servisini başlatır
sudo systemctl start elasticsearch

# Sistem yeniden başladığında otomatik çalışması için etkinleştirir
sudo systemctl enable elasticsearch

# Servis durumunu kontrol eder
sudo systemctl status elasticsearch
```

```bash
# HTTP isteği göndererek Elasticsearch'ün çalışıp çalışmadığını test eder
# Başarılı yanıtta cluster_name, version gibi bilgiler JSON olarak döner
curl http://localhost:9200
```

Beklenen yanıt:

```json
{
  "name" : "node-1",
  "cluster_name" : "elasticsearch",
  "version" : { ... },
  "tagline" : "You Know, for Search"
}
```

---

## Adım 4 – Metricbeat Kurulumu

### Metricbeat Nedir?

**Metricbeat**, sistem ve servis metriklerini (CPU, bellek, disk, ağ, Docker vb.) toplayarak doğrudan Elasticsearch'e veya Logstash'e gönderen hafif bir ajandır.

```bash
# Metricbeat'i kurar (Elastic deposu zaten eklendiği için doğrudan yüklenebilir)
sudo apt-get install metricbeat

# Metricbeat servisini başlatır
sudo systemctl start metricbeat

# Sistem yeniden başladığında otomatik çalışması için etkinleştirir
sudo systemctl enable metricbeat
```

---

## Adım 5 – Logstash Kurulumu ve Yapılandırması

### Logstash Nedir?

**Logstash**, çeşitli kaynaklardan veri toplayan (input), dönüştüren/filtreleyen (filter) ve hedefe ileten (output) bir veri işleme pipeline'ıdır. ELK Stack'te log verilerini Elasticsearch'e aktarma görevi üstlenir.

### 5.1 – Logstash Kurulumu

```bash
# Logstash'i kurar
sudo apt-get install logstash
```

### 5.2 – Test Log Dosyası Oluşturma

Logstash'in çalışıp çalışmadığını test etmek için örnek Apache access log içeriğiyle bir dosya oluşturun:

```bash
# /home dizininde test_log.log dosyasını oluşturur ve açar
sudo vi /home/test_log.log
```

Aşağıdaki örnek log satırlarını dosyaya yapıştırın:

```
72.132.86.148 - - [30/Mar/2020:00:00:05 +0000] "GET /item/books/2957 HTTP/1.1" 200 85 "/category/toys" "Mozilla/5.0 (compatible; MSIE 9.0; Windows NT 6.1; Trident/5.0)"
204.39.102.110 - - [30/Mar/2020:00:00:10 +0000] "GET /category/electronics HTTP/1.1" 200 67 "-" "Mozilla/5.0 (Macintosh; Intel Mac OS X 10.6; rv:9.0.1) Gecko/20100101 Firefox/9.0.1"
180.171.26.145 - - [30/Mar/2020:00:00:15 +0000] "GET /item/books/2957 HTTP/1.1" 200 85 "/category/books?from=0" "Mozilla/4.0"
140.45.44.120 - - [30/Mar/2020:00:01:05 +0000] "GET /item/software/2055 HTTP/1.1" 500 46 "-" "Mozilla/4.0"
24.228.172.209 - - [30/Mar/2020:00:01:10 +0000] "GET /item/electronics/1670 HTTP/1.1" 500 93 "-" "Mozilla/5.0"
```

### 5.3 – Logstash Pipeline Yapılandırması

```bash
# Logstash için yeni bir pipeline yapılandırma dosyası oluşturur
sudo vi /etc/logstash/conf.d/apache-01.conf
```

Aşağıdaki içeriği dosyaya ekleyin:

```ruby
# INPUT: Logstash'in veriyi nereden okuyacağını tanımlar
input {
  file {
    path => "/home/test_log.log"   # İzlenecek log dosyasının yolu
    start_position => "beginning"  # Dosyayı baştan okur (varsayılan: end — sadece yeni satırları okur)
  }
}

# OUTPUT: İşlenen verinin nereye gönderileceğini tanımlar
output {
  elasticsearch {
    hosts => ["localhost:9200"]    # Elasticsearch'ün adresi; veri buraya iletilir
  }
}
```

**Pipeline bölüm açıklamaları:**

| Bölüm | Görev |
|---|---|
| `input` | Veri kaynağını tanımlar (dosya, syslog, kafka, beats vb.) |
| `filter` | Veriyi ayrıştırır ve dönüştürür (bu örnekte kullanılmadı) |
| `output` | İşlenen veriyi hedefe gönderir (Elasticsearch, stdout vb.) |

### 5.4 – Logstash'i Başlatma ve Doğrulama

```bash
# Logstash servisini başlatır
sudo systemctl start logstash

# Sistem yeniden başladığında otomatik çalışması için etkinleştirir
sudo systemctl enable logstash
```

```bash
# Elasticsearch'teki indeksleri listeler; logstash-* indeksinin oluşup oluşmadığını kontrol eder
# -XGET : HTTP GET isteği gönderir
# _cat/indices : tüm indeksleri tablo formatında listeleyen API endpoint'i
# ?v : sütun başlıklarını gösterir
# &pretty : çıktıyı okunabilir formatta basar
sudo curl -XGET 'localhost:9200/_cat/indices?v&pretty'
```

Beklenen çıktıda `logstash-YYYY.MM.DD` formatında bir indeks satırı görünmesi gerekir.

---

## Adım 6 – Kibana Kurulumu ve Yapılandırması

### Kibana Nedir?

**Kibana**, Elasticsearch'teki verileri görselleştiren, arama ve analiz imkânı sunan web tabanlı bir arayüzdür. Log analizi, metrik izleme ve dashboard oluşturma için kullanılır.

### 6.1 – Kibana Kurulumu

```bash
# Kibana'yı kurar
sudo apt-get install kibana
```

### 6.2 – Kibana Yapılandırması

```bash
# Kibana yapılandırma dosyasını açar
sudo vi /etc/kibana/kibana.yml
```

Aşağıdaki satırların aktif ve doğru değerlere sahip olduğundan emin olun:

```yaml
# =================== Kibana Sunucu Ayarları ===================

server.port: 5601
# Kibana'nın dinleyeceği port; Security Group'ta açık olmalı

server.host: "0.0.0.0"
# "0.0.0.0" = tüm ağ arayüzlerinden bağlantı kabul et
# "localhost" bırakılırsa EC2 dışından erişilemez

# =================== Elasticsearch Bağlantısı ===================

elasticsearch.hosts: ["http://localhost:9200"]
# Kibana'nın veri için bağlanacağı Elasticsearch adresi
```

### 6.3 – Kibana'yı Başlatma

```bash
# Kibana servisini başlatır
sudo systemctl start kibana

# Sistem yeniden başladığında otomatik çalışması için etkinleştirir
sudo systemctl enable kibana
```

> Kibana'nın tam olarak başlaması **30–45 saniye** sürebilir.

Tarayıcıdan erişin:

```
http://<EC2-PUBLIC-IP>:5601
```

Kibana dashboard ana sayfasına yönlendirileceksiniz.

---

## Tüm Servislerin Durumunu Kontrol Etme

```bash
# Tüm ELK bileşenlerinin çalışıp çalışmadığını toplu kontrol eder
sudo systemctl status elasticsearch
sudo systemctl status logstash
sudo systemctl status kibana
sudo systemctl status metricbeat
```

---

## ELK Stack Mimarisinin Özeti

```
Log Kaynakları
  ├── /home/test_log.log (Apache log)
  └── Sistem metrikleri (Metricbeat)
          │
          │ veri gönderir
          ▼
      Logstash (pipeline: input → filter → output)
      Metricbeat (hafif ajan)
          │
          │ index oluşturur / veri yazar
          ▼
      Elasticsearch        (port 9200)
      └── JSON belgelerini depolar, indeksler, arama yapar
          │
          │ veri sorgular
          ▼
      Kibana               (port 5601)
      └── Dashboard, görselleştirme, log arama arayüzü
```

---

## Kaynaklar

- [Elasticsearch Resmi Dokümantasyonu](https://www.elastic.co/guide/en/elasticsearch/reference/index.html)
- [Logstash Yapılandırma Rehberi](https://www.elastic.co/guide/en/logstash/current/configuration.html)
- [Kibana Kullanım Kılavuzu](https://www.elastic.co/guide/en/kibana/current/index.html)
- [Metricbeat Dokümantasyonu](https://www.elastic.co/guide/en/beats/metricbeat/current/index.html)
