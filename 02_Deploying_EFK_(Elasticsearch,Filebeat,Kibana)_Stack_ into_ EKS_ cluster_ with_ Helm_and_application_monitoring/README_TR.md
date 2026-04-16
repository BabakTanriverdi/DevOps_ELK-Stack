# Hands-on EFK Stack: EKS Üzerinde Helm ile Kurulum ve Microservice Log İzleme

> **Amaç:** Bu uygulamalı eğitim, EFK Stack (Elasticsearch, Filebeat, Kibana) bileşenlerinin Amazon EKS cluster'ına Helm ile kurulumunu ve Kubernetes microservice uygulama loglarının izlenmesini kapsar.

---

## Öğrenme Hedefleri

Bu eğitimin sonunda öğrenciler:

- EKS cluster'ı oluşturup yönetebilecek
- `kubectl` ve `eksctl` araçlarını kurabilecek
- EBS CSI sürücüsünü EKS'e ekleyebilecek
- Helm ile EFK Stack bileşenlerini EKS'e deploy edebilecek
- Kibana üzerinden Kubernetes uygulama loglarını izleyebilecek

---

## EFK Stack Nedir?

EFK Stack, Kubernetes ortamlarında log yönetimi için tercih edilen üç bileşenli bir platformdur:

| Bileşen | Görev | Varsayılan Port |
|---|---|---|
| **Elasticsearch** | Logların depolandığı ve sorgulandığı arama/analitik motoru | 9200 |
| **Filebeat** | Pod'lardan log toplayan ve Elasticsearch'e ileten hafif ajan | — |
| **Kibana** | Logları görselleştiren ve analiz eden web arayüzü | 5601 |

> **ELK ve EFK farkı:** ELK Stack'te log işleme aracı olarak **Logstash** kullanılırken, EFK Stack Kubernetes ortamları için daha hafif olan **Filebeat** tercih eder.

---

## İçerik

- [Bölüm 1 – kubectl ve eksctl Kurulumu](#bölüm-1--kubectl-ve-eksctl-kurulumu)
- [Bölüm 2 – EKS Cluster Oluşturma](#bölüm-2--eks-cluster-oluşturma)
- [Bölüm 3 – EBS CSI Sürücüsü Kurulumu](#bölüm-3--ebs-csi-sürücüsü-kurulumu)
- [Bölüm 4 – Helm Kurulumu](#bölüm-4--helm-kurulumu)
- [Bölüm 5 – EFK Stack Kurulumu](#bölüm-5--efk-stack-kurulumu)
- [Bölüm 6 – Kibana Dashboard ve Uygulama Logları](#bölüm-6--kibana-dashboard-ve-uygulama-logları)
- [Kubernetes Log Komutları](#kubernetes-log-komutları)
- [Cluster Silme](#cluster-silme)

---

## Bölüm 1 – kubectl ve eksctl Kurulumu

### EC2 Instance Hazırlama

Aşağıdaki ayarlarla bir Amazon EC2 instance'ı başlatın:

| Parametre | Değer |
|---|---|
| **AMI** | Amazon Linux 2023 |
| **Instance Type** | t2.micro (yönetim sunucusu için yeterli) |
| **Security Group** | Port 22 (SSH) açık |

SSH ile bağlandıktan sonra sistemi güncelleyin:

```bash
# dnf: Amazon Linux 2023'ün paket yöneticisi (yum'un yerini almıştır)
# -y: tüm onay sorularına otomatik "evet" yanıtı verir
sudo dnf update -y
```

---

### kubectl Kurulumu

**kubectl**, Kubernetes cluster'ını komut satırından yönetmek için kullanılan resmi istemci aracıdır.

```bash
# Amazon EKS tarafından doğrulanan kubectl binary'sini indirir
# -O: indirilen dosyayı uzak sunucudaki adıyla kaydeder
curl -O https://s3.us-west-2.amazonaws.com/amazon-eks/1.30.2/2024-07-12/bin/linux/amd64/kubectl
```

```bash
# İndirilen binary'ye çalıştırma izni verir (varsayılan olarak izin yoktur)
# +x: execute (çalıştırma) iznini ekler
chmod +x ./kubectl
```

```bash
# $HOME/bin dizini yoksa oluşturur (-p: üst dizinleri de oluşturur)
# kubectl'i bu dizine kopyalar
# PATH ortam değişkenine $HOME/bin'i ekler (bu oturum için geçici)
mkdir -p $HOME/bin && cp ./kubectl $HOME/bin/kubectl && export PATH=$PATH:$HOME/bin
```

```bash
# PATH değişkenini kalıcı hale getirir
# ~/.bashrc: yeni terminal oturumlarında otomatik çalışan kabuk yapılandırma dosyası
echo 'export PATH=$PATH:$HOME/bin' >> ~/.bashrc
```

```bash
# kubectl'in başarıyla kurulduğunu ve sürümünü doğrular
kubectl version --client
```

---

### eksctl Kurulumu

**eksctl**, Amazon EKS cluster'larını komut satırından kolayca oluşturmak, güncellemek ve silmek için Weaveworks tarafından geliştirilen resmi araçtır.

```bash
# İşletim sistemini otomatik algılar (uname -s) ve uygun eksctl sürümünü indirir
# -s: sessiz mod, -L: yönlendirmeleri takip et, -O: dosyayı kaydet
curl -sLO "https://github.com/eksctl-io/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz"
```

```bash
# Arşivi /tmp dizinine çıkarır
# -x: extract, -z: gzip, -f: dosya adı, -C: hedef dizin
# rm: indirilen arşiv dosyasını siler (temizlik)
tar -xzf eksctl_$(uname -s)_amd64.tar.gz -C /tmp && rm eksctl_$(uname -s)_amd64.tar.gz
```

```bash
# eksctl binary'sini sistem genelinde erişilebilir /usr/local/bin dizinine taşır
sudo mv /tmp/eksctl /usr/local/bin
```

```bash
# Kurulumun başarılı olduğunu ve sürümü doğrular
eksctl version
```

---

## Bölüm 2 – EKS Cluster Oluşturma

### AWS Kimlik Bilgilerini Yapılandırma

```bash
# SSH key çifti henüz yoksa oluşturur
# -f: çıktı dosyasının yolu
ssh-keygen -f ~/.ssh/id_rsa

# AWS erişim anahtarlarını ve bölgeyi yapılandırır
# Alternatif: EC2 instance'ına IAM Role attach etmek (daha güvenli yöntem)
aws configure
```

### EKS Cluster Oluşturma

```bash
# eksctl ile yönetilen (managed) bir EKS cluster'ı oluşturur
eksctl create cluster \
  --name mycluster \                              # Cluster adı
  --region us-east-1 \                            # AWS bölgesi
  --zones us-east-1a,us-east-1b,us-east-1c \      # Kullanılacak Availability Zone'lar
  --nodegroup-name my-nodes \                     # Worker node grubu adı
  --node-type t2.medium \                         # Node instance türü (EKS için minimum t2.medium önerilir)
  --nodes 2 \                                     # Başlangıç node sayısı
  --nodes-min 1 \                                 # Otomatik ölçeklendirme minimum node sayısı
  --nodes-max 2 \                                 # Otomatik ölçeklendirme maksimum node sayısı
  --version 1.30 \                                # Kubernetes sürümü
  --ssh-access \                                  # Node'lara SSH erişimini etkinleştirir
  --ssh-public-key ~/.ssh/id_rsa.pub \            # SSH için kullanılacak public key
  --managed                                       # AWS tarafından yönetilen node grubu oluşturur
```

> **Not:** Cluster oluşturma işlemi **10–15 dakika** sürebilir. Arka planda CloudFormation stack'leri oluşturulur.

Alternatif kısa komut (varsayılan değerlerle):

```bash
eksctl create cluster \
  --region us-east-1 \
  --zones us-east-1a,us-east-1b,us-east-1c \
  --node-type t2.medium \
  --nodes 2 \
  --nodes-min 1 \
  --nodes-max 2 \
  --version 1.30 \
  --name mycluster
```

```bash
# Tüm eksctl parametrelerini ve varsayılan değerlerini listeler
eksctl create cluster --help
```

> AWS Console'da **EKS servisini** ve **CloudFormation servisini** açarak `eksctl-mycluster-cluster` stack'ini inceleyebilirsiniz.

---

## Bölüm 3 – EBS CSI Sürücüsü Kurulumu

### Amazon EBS CSI Sürücüsü Nedir?

**Amazon EBS Container Storage Interface (CSI) Driver**, EKS cluster'ının Amazon EBS (Elastic Block Store) birimlerini dinamik olarak oluşturmasına, bağlamasına ve yönetmesine olanak tanır. Elasticsearch gibi durum bilgisi (stateful) tutan uygulamalar için kalıcı depolama sağlar.

> EKS cluster'ı ilk oluşturulduğunda CSI sürücüsü kurulu gelmez; ayrıca eklenmesi gerekir.

---

### 3.1 – IAM OIDC Provider Oluşturma

EBS CSI sürücüsünün AWS API'lerine erişebilmesi için cluster'a bir **IAM OIDC (OpenID Connect) kimlik sağlayıcısı** bağlanmalıdır.

```bash
# Cluster'ın OIDC issuer URL'sinden provider ID'sini çeker ve değişkene atar
# --query: JSON çıktısından belirli alanı seçer
# cut -d '/' -f 5: URL'yi '/' ile böler ve 5. parçayı (ID) alır
oidc_id=$(aws eks describe-cluster --name mycluster --query "cluster.identity.oidc.issuer" --output text | cut -d '/' -f 5)
```

```bash
# Bu cluster ID ile eşleşen OIDC provider'ın hesapta zaten olup olmadığını kontrol eder
# Çıktı döndürülürse provider mevcuttur; sonraki adım atlanabilir
aws iam list-open-id-connect-providers | grep $oidc_id
```

```bash
# OIDC provider yoksa oluşturur
# --approve: onay sormadan işlemi gerçekleştirir
eksctl utils associate-iam-oidc-provider --region=us-east-1 --cluster=mycluster --approve
```

---

### 3.2 – EBS CSI Sürücüsü için IAM Rolü Oluşturma

```bash
# EBS CSI plugin'i için IAM servis hesabı ve rolü oluşturur
# --role-only: yalnızca IAM rolünü oluşturur, ServiceAccount'u annotate etmez
# --attach-policy-arn: AWS'nin hazır EBS CSI politikasını role ekler
eksctl create iamserviceaccount \
    --region us-east-1 \
    --name ebs-csi-controller-sa \                        # Kubernetes ServiceAccount adı
    --namespace kube-system \                             # ServiceAccount'ın bulunduğu namespace
    --cluster mycluster \
    --role-name AmazonEKS_EBS_CSI_DriverRole \            # Oluşturulacak IAM rol adı
    --role-only \
    --attach-policy-arn arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy \
    --approve
```

---

### 3.3 – EBS CSI Add-on Ekleme

```bash
# EBS CSI sürücüsünü EKS add-on olarak cluster'a ekler
# --service-account-role-arn: önceki adımda oluşturulan IAM rolünün ARN'si
# 111122223333 → kendi AWS hesap ID'nizle değiştirin
eksctl create addon \
  --region us-east-1 \
  --name aws-ebs-csi-driver \
  --cluster mycluster \
  --service-account-role-arn arn:aws:iam::111122223333:role/AmazonEKS_EBS_CSI_DriverRole \
  --force
```

> Daha fazla bilgi: [Amazon EBS CSI Driver Dokümantasyonu](https://docs.aws.amazon.com/eks/latest/userguide/managing-ebs-csi.html)

---

## Bölüm 4 – Helm Kurulumu

```bash
# Helm'in resmi kurulum betiğini indirir ve çalıştırır
curl https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3 | bash

# Kurulumun başarılı olduğunu ve sürümü doğrular
helm version
```

---

## Bölüm 5 – EFK Stack Kurulumu

### 5.1 – Namespace ve StorageClass Hazırlığı

```bash
# EFK bileşenlerini izole etmek için "efk" adında ayrı bir namespace oluşturur
# Namespace: Kubernetes'te kaynakları mantıksal gruplar halinde ayırma mekanizması
kubectl create namespace efk

# Oluşturulan namespace'i doğrular
kubectl get namespaces
```

```bash
# gp2 StorageClass'ını varsayılan (default) olarak işaretler
# Elasticsearch'in PersistentVolume talepleri bu sınıfı otomatik kullanır
# annotate: mevcut bir kaynağa metadata etiketi ekler
kubectl annotate storageclass gp2 storageclass.kubernetes.io/is-default-class="true"
```

### 5.2 – Elastic Helm Deposu Ekleme

```bash
# Elasticsearch, Kibana, Filebeat gibi Elastic ürünlerinin Helm chart'larını barındıran depoyu ekler
helm repo add elastic https://helm.elastic.co

# Eklenen deponun en güncel chart listesini çeker
helm repo update

# Eklenmiş tüm Helm depolarını listeler
helm repo list

# Depodaki tüm mevcut chart'ları listeler
helm search repo elastic
```

---

### 5.3 – Elasticsearch Kurulumu

```bash
# elastic/elasticsearch chart'ının tüm yapılandırma değerlerini dosyaya yazar
# Bu dosya üzerinde değişiklik yapılarak özelleştirilmiş kurulum gerçekleştirilir
helm show values elastic/elasticsearch >> elasticsearch.values
```

`elasticsearch.values` dosyasında aşağıdaki değerleri bulup düzenleyin:

```yaml
replicas: 1             # Tek node'lu test ortamı için 1 replica yeterlidir (varsayılan: 3)
minimumMasterNodes: 1   # Master seçimi için minimum node sayısı; replica sayısıyla uyumlu olmalı
```

```bash
# Özelleştirilmiş values dosyasıyla Elasticsearch'ü efk namespace'ine kurar
# -f: özel values dosyasını belirtir; varsayılan değerlerin üzerine yazar
# -n: hedef namespace
helm install elasticsearch elastic/elasticsearch -f elasticsearch.values -n efk
```

```bash
# efk namespace'indeki Helm release'lerini listeler; kurulum durumunu gösterir
helm list -n efk

# efk namespace'indeki tüm Kubernetes nesnelerini listeler (Pod, Service, PVC vb.)
kubectl get all -n efk
```

---

### 5.4 – Kibana Kurulumu

```bash
# Kibana chart'ının yapılandırma değerlerini dosyaya yazar
helm show values elastic/kibana >> kibana.values
```

`kibana.values` dosyasında `service` bölümünü aşağıdaki gibi güncelleyin:

```yaml
service:
  type: LoadBalancer    # "ClusterIP" yerine; AWS üzerinde External Load Balancer oluşturur
  loadBalancerIP: ""    # Boş bırakılırsa AWS otomatik IP atar
  port: 5601            # Kibana'nın dinleyeceği port
  nodePort: ""
  labels: {}
```

```bash
# Kibana'yı efk namespace'ine kurar
helm install kibana elastic/kibana -f kibana.values -n efk
```

```bash
# Kibana Service'ini izler; EXTERNAL-IP sütununda LoadBalancer adresi atanana kadar bekleyin
# Birkaç dakika sürebilir
kubectl get service kibana-kibana -n efk
```

Beklenen çıktı:

```
NAME            TYPE           CLUSTER-IP      EXTERNAL-IP                                                              PORT(S)
kibana-kibana   LoadBalancer   10.100.48.233   a8aea5ee60d594c4a9ca516672c21a52-407102138.us-east-1.elb.amazonaws.com   5601:32358/TCP
```

Tarayıcıdan erişin (EXTERNAL-IP + port):

```
http://a8aea5ee60d594c4a9ca516672c21a52-407102138.us-east-1.elb.amazonaws.com:5601
```

#### Kibana Giriş Bilgileri

```bash
# Kullanıcı adı: elastic (sabit)
# Şifreyi Kubernetes Secret'ından çeker ve base64 decode eder
kubectl -n efk get secret elasticsearch-master-credentials \
  -ojsonpath='{.data.password}' | base64 --decode
```

---

### 5.5 – Filebeat Kurulumu

**Filebeat**, Kubernetes ortamında her node'da çalışan (DaemonSet) hafif bir log toplama ajanıdır. Pod'ların stdout/stderr çıktılarını ve dosya loglarını otomatik olarak toplayıp Elasticsearch'e iletir.

```bash
# Filebeat'i efk namespace'ine varsayılan yapılandırmayla kurar
helm install filebeat elastic/filebeat -n efk
```

```bash
# Tüm EFK bileşenlerinin çalışıp çalışmadığını kontrol eder
kubectl get all -n efk

# Elasticsearch için oluşturulan PersistentVolumeClaim'leri listeler
# EBS CSI sürücüsü sayesinde EBS volume'ları dinamik olarak oluşturulmuş olmalıdır
kubectl get pvc -n efk
```

---

## Bölüm 6 – Kibana Dashboard ve Uygulama Logları

### 6.1 – Kibana'da Index Pattern Oluşturma

1. Kibana arayüzüne giriş yapın
2. Sol menüden **Discover** bölümüne gidin
3. **Create data view** butonuna tıklayın
4. Aşağıdaki ayarları yapın:

   | Alan | Değer |
   |---|---|
   | **Index pattern** | `filebeat-*` |
   | **Timestamp field** | `@timestamp` |

5. **Create index pattern** butonuna tıklayın

> `filebeat-*` pattern'i; Filebeat'in Elasticsearch'e gönderdiği tüm günlük indeksleri kapsar.

---

### 6.2 – Örnek Uygulamaları Deploy Etme

Test logları üretmek için iki örnek uygulama deploy edin:

```bash
# PHP Apache uygulamasını deploy eder
kubectl apply -f php_apache.yaml

# To-Do uygulamasını deploy eder
kubectl apply -f to_do.yaml

# Deploy edilen tüm kaynakları listeler
kubectl get all
```

Uygulamalara tarayıcıdan erişmek için EKS node'larının Security Group'una aşağıdaki portları açın:

| Port Aralığı | Açıklama |
|---|---|
| `30001` | php_apache uygulaması NodePort |
| `30002` | to_do uygulaması NodePort |

```
http://<EKS-NODE-PUBLIC-IP>:30001
http://<EKS-NODE-PUBLIC-IP>:30002
```

Uygulamalara tarayıcıdan istek attıkça Kibana'nın Discover bölümünde log kayıtlarının geldiğini görebilirsiniz.

---

## Kubernetes Log Komutları

Kibana'ya ek olarak terminal üzerinden de log görüntüleyebilirsiniz:

```bash
# Belirtilen Pod'un tüm loglarını stdout'a yazdırır
kubectl logs my-pod

# Belirli bir etiketle (label) eşleşen tüm Pod'ların loglarını getirir
kubectl logs -l name=myLabel

# Pod yeniden başlatılmadan önceki (bir önceki çalışmaya ait) logları getirir
kubectl logs my-pod --previous

# Çok container'lı Pod'larda belirli bir container'ın loglarını getirir
kubectl logs my-pod -c my-container

# Etiket filtresiyle belirli container'ın loglarını getirir
kubectl logs -l name=myLabel -c my-container

# Çok container'lı Pod'da önceki çalışmaya ait container loglarını getirir
kubectl logs my-pod -c my-container --previous

# Logları canlı olarak akışa alır (tail -f gibi); Ctrl+C ile durdurulur
kubectl logs -f my-pod

# Canlı akışta belirli container'ın loglarını takip eder
kubectl logs -f my-pod -c my-container

# Etiketle eşleşen tüm Pod'ların tüm container loglarını canlı olarak akışa alır
kubectl logs -f -l name=myLabel --all-containers
```

---

## Cluster Silme

Eğitim tamamlandığında gereksiz AWS maliyetlerinden kaçınmak için cluster'ı silin:

```bash
# Mevcut EKS cluster'larını ve bölgelerini listeler
eksctl get cluster --region us-east-1
```

Beklenen çıktı:

```
NAME          REGION
mycluster     us-east-1
```

```bash
# Cluster'ı ve ilişkili tüm kaynakları (node group, VPC, IAM rolleri vb.) siler
# Silme işlemi 10–15 dakika sürebilir
eksctl delete cluster mycluster --region us-east-1
```

> **Uyarı:** Cluster silinmeden EC2 instance'ı kapatmak, EBS volume'larını ve Load Balancer'ları silerek ücretlendirmeyi durdurmaz. Her zaman önce `eksctl delete cluster` komutunu çalıştırın.

---

## EFK Stack Mimarisinin Özeti

```
EKS Cluster (us-east-1)
  │
  ├── efk Namespace
  │     ├── Elasticsearch (StatefulSet)
  │     │     └── EBS PersistentVolume üzerinde veri saklar
  │     │
  │     ├── Filebeat (DaemonSet — her node'da 1 Pod)
  │     │     └── Tüm Pod loglarını toplar → Elasticsearch'e iletir
  │     │
  │     └── Kibana (Deployment)
  │           └── LoadBalancer Service → tarayıcıdan erişilir (port 5601)
  │
  └── default Namespace
        ├── php_apache (NodePort: 30001)
        └── to_do      (NodePort: 30002)
              └── Ürettikleri loglar Filebeat tarafından toplanır
```

---

## Kaynaklar

- [Amazon EKS Resmi Dokümantasyonu](https://docs.aws.amazon.com/eks/latest/userguide/)
- [eksctl Kullanım Kılavuzu](https://eksctl.io/)
- [Amazon EBS CSI Driver](https://docs.aws.amazon.com/eks/latest/userguide/managing-ebs-csi.html)
- [Elastic Helm Charts](https://helm.elastic.co)
- [Filebeat Kubernetes Dokümantasyonu](https://www.elastic.co/guide/en/beats/filebeat/current/running-on-kubernetes.html)
