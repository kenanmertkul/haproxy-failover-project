
# HAProxy Failover and Load Balancing Project

Bu proje, **HAProxy** kullanarak iki birincil (primary) sunucu arasında **yük paylaşımı (load balancing)** ve bir yedek (backup) sunucu ile **failover mekanizmasını** test etmek için geliştirilmiştir. Ayrıca, Prometheus ve Grafana entegrasyonuyla sistemin izlenebilirliğini sağlar.

## Gereksinimler

Proje çalıştırılmadan önce aşağıdaki yazılımlar sisteminizde kurulu olmalıdır:

- **Docker** (v20.10+)
- **Docker Compose** (v1.29+)

---

## Kullanım Adımları

1. **Proje Yapısı**

   Proje aşağıdaki Docker servislerini içerir:

   - **HAProxy**: Trafik yönlendirme, yük paylaşımı ve failover mekanizması.
   - **ServerA** ve **ServerB**: Apache HTTP sunucuları, yükü paylaşır.
   - **ServerC**: Apache HTTP yedek sunucusu (backup).
   - **Prometheus**: HAProxy metriklerini toplar.
   - **Grafana**: Prometheus metriklerini görselleştirir.
   - **HAProxy Exporter**: Prometheus için HAProxy metriklerini toplar.

2. **Projenin Klonlanması ve Başlatılması**
   ```bash
   git clone https://github.com/kenanmertkul/haproxy-failover-project.git
   cd haproxy-failover-project
   ```

   ```bash
   docker-compose up -d
   ```

   Yukarıdaki komut projeyi klonlar, tüm konteynerleri başlatır ve sistemi çalıştırır.

3. **Testler ve Doğrulamalar**

   ### Failover ve Yük Paylaşımı Testi

   - **Başlangıç Durumu**:
     - `serverA` ve `serverB` konteynerleri çalışırken tarayıcıdan `http://localhost` adresine gidin ve **ServerA** veya **ServerB** çıktısını doğrulayın. 
     - Bu, HAProxy'nin **yük paylaşımı** yaptığını gösterir.

   - **Failover Senaryosu**:
     - Öncelikle `serverA` ve `serverB` konteynerlerini durdurun:
       ```bash
       docker stop fail-over-project-serverA-1
       docker stop fail-over-project-serverB-1
       ```
     - Tarayıcıdan `http://localhost` adresine gidin ve yedek sunucu (ServerC) çıktısını doğrulayın.
     - `serverA` ve `serverB` konteynerlerini yeniden başlatın:
       ```bash
       docker start fail-over-project-serverA-1
       docker start fail-over-project-serverB-1
       ```
     - Yük paylaşımı mekanizmasının yeniden devreye girdiğini kontrol edin.

   ### İzleme Testi

   - **HAProxy İstatistik** arayüzüne erişim:
     - Tarayıcıdan `http://localhost:8404/stats` adresine gidin.
     - Kullanıcı adı: `admin`
     - Şifre: `password`
   - **Prometheus** arayüzüne erişim:
     - Tarayıcıdan `http://localhost:9090` adresine giderek Prometheus veri kaynağını inceleyin.
   - **Grafana** arayüzüne erişim:
     - Tarayıcıdan `http://localhost:3000` adresine gidin.
     - Varsayılan kullanıcı adı: `admin`
     - Varsayılan şifre: `admin`
     - Prometheus bir veri kaynağı olarak eklenmiştir ve sistem metriklerini izlemek için bir HAProxy panosu kurulmuştur.

   ### Yük Testi (Apache Bench)

   Apache Bench kullanarak yük testi yapabilirsiniz. Aşağıdaki komut, 1000 istek ve 10 eşzamanlı bağlantıyla bir test gerçekleştirir:

   ```bash
   ab -n 1000 -c 10 http://localhost/
   ```

   Test sırasında, HAProxy'nin **ServerA** ve **ServerB** arasında yükü nasıl paylaştığını ve birincil sunucuların çökmesi durumunda **ServerC**'nin devreye girip girmediğini gözlemleyin.

---

## Teknik Detaylar ve Konfigürasyonlar

- **HAProxy Konfigürasyonu**:
   `haproxy.cfg` dosyası aşağıdaki gibi yapılandırılmıştır:
   ```haproxy
   backend servers
       balance roundrobin
       server serverA serverA:80 check inter 2s rise 3 fall 2
       server serverB serverB:80 check inter 2s rise 3 fall 2
       server serverC serverC:80 check backup inter 2s rise 3 fall 2
   ```
   - **`balance roundrobin`**: Yük paylaşımı mekanizmasını etkinleştirir.
   - **`backup`**: Sunucunun yedek olarak ayarlandığını belirtir.
   - **`check`**, **`inter`**, **`rise`**, **`fall`**: Sunucu sağlığını kontrol eder ve failover senaryosunu yönetir.

- **Prometheus ve Grafana**:
   - **Prometheus**: `prometheus.yml` dosyasında HAProxy Exporter hedefi tanımlıdır:
     ```yaml
     scrape_configs:
       - job_name: 'haproxy'
         static_configs:
           - targets: ['haproxy-exporter:9101']
     ```
