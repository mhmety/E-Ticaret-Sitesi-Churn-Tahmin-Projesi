# 🚀 Türk E-Ticaret Sitesi İçin Churn Tahmini & Özellik Mühendisliği

Bu proje, ham e-ticaret işlem (transaction) verilerini kullanarak müşterilerin terk etme (churn) eğilimlerini tahmin etmek amacıyla geliştirilmiştir. Projenin ana odağı, büyük veri setlerinde iş yükü optimizasyonu (workload optimization) sağlamak amacıyla ham veriyi **SQL Server** üzerinde işleyip analitik özellikleri üretmek, ardından **Python (XGBoost)** kullanarak makine öğrenmesi modelini kurmaktır.

---

## 📊 Veri Seti Hakkında

Projede kullanılan veri setine aşağıdaki bağlantıdan ulaşabilirsiniz:
🔗 **[Kaggle - Sinem Dökmeci E-Ticaret Veri Seti](https://www.kaggle.com/datasets/sinemdokmeci/e-ticaret)**

> ⚠️ **Not:** Veri seti boyutu büyük olduğu için sektör standartlarına (best practices) uygun olarak `.gitignore` dosyası ile filtrelenmiş ve GitHub deposuna yüklenmemiştir. Projeyi lokalinizde çalıştırmak için yukarıdaki linkten veri setini indirip proje kök dizininde `archive/` adlı bir klasör oluşturarak içine yerleştirmeniz gerekmektedir.

---

## 🛠️ Uygulanan Mühendislik Adımları

### 1. SQL ile Veri Temizleme & Sanallaştırma
Gerçek hayat senaryolarına uygun olarak veri ambarındaki ham veriye doğrudan müdahale edilmemiş (`DELETE` veya `UPDATE` kullanılmamış), bunun yerine ham verinin bütünlüğünü korumak adına bir **SQL View (Sanal Tablo)** oluşturulmuştur.
* `CustomerID` (Müşteri ID) alanı boş (`NULL`) olan geçersiz satırlar elenmiştir.
* İptal ve iade işlemlerinden kaynaklanan, sistemi ve model kararlarını yanıltabilecek negatif miktar (`Quantity < 0`) ve fiyat (`Price < 0`) verileri temizlenmiştir.

### 2. SQL ile Özellik Mühendisliği (Feature Engineering)
Yüz binlerce satırdan oluşan işlem geçmişi verisi, **CTE (Common Table Expressions)** ve gruplama fonksiyonları kullanılarak benzersiz müşteri seviyesine indirgenmiştir. Her müşteri için makine öğrenmesi modelinin besleneceği temel analitik metrikler çıkarılmıştır:
* **Frequency (Sıklık):** Müşterinin toplam benzersiz sipariş sayısı (`COUNT(DISTINCT InvoiceNo)`).
* **Monetary (Maddi Değer):** Müşterinin şirkete kazandırdığı toplam ciro (`SUM(Quantity * Price)`).

### 3. Veri Sızıntısı (Data Leakage) Tespiti ve Çözümü
* **Kural:** Churn (terk etme) etiketi, veri setindeki sanal bugünden geriye doğru son 30 gün boyunca hiç alışveriş yapmama durumuna göre (`CASE WHEN` mantığı ile) kurgulanmıştır.
* **Kriz:** İlk model denemesinde, müşterinin son alışverişinden beri geçen gün sayısını ifade eden `Recency` metriği girdilere dahil edildiğinde model **%100 (1.00) Accuracy** skoru üretmiştir. Yapılan mimari incelemede, hedef değişkenin (`Is_Churn`) doğrudan `Recency > 30` kuralına bağlı olması sebebiyle modelin kurumsal bir hile olan **Data Leakage (Veri Sızıntısı)** durumuna düştüğü fark edilmiştir.
* **Çözüm:** Model mimarisi güncellenerek `Recency` sütunu girdilerden (`X`) arındırılmış, modelin sadece müşterinin alışveriş alışkanlıklarına (`Frequency` ve `Monetary`) bakarak geleceği gerçekçi bir şekilde tahmin etmesi sağlanmıştır.

### 4. Makine Öğrenmesi Modellemesi (XGBoost)
Veri seti %80 Eğitim, %20 Test olarak bölünmüş ve yapılandırılmış tablolarda çok güçlü sonuçlar veren **XGBoost Classifier** algoritması eğitilmiştir. Model, veri sızıntısından arındırıldıktan sonra şu gerçekçi ve başarılı metrikleri üretmiştir:
* **Genel Doğruluk (Accuracy):** %76
* **Churn Yakalama Başarısı (Recall / Duyarlılık):** **0.91**
  * *Yorum:* Model, bizi terk etme eğiliminde olan müşterilerin **%91'ini** henüz gitmeden doğru tespit etmeyi başarmaktadır. Churn problemlerinde şirkete asıl ciro kaybı yaşatan unsurları (kaçan müşterileri) ıskalamama noktasında model son derece kararlıdır.

---

## 💼 İş Değeri (Business Value)

Bu projenin çıktısı, e-ticaret şirketinin pazarlama ve müşteri ilişkileri (CRM) departmanları için doğrudan aksiyona dönüştürülebilir veri üretir:
1. **Erken Uyarı Sistemi:** Gitme ihtimali yüksek olan müşteriler (Sınıf 1) henüz platformu tamamen terk etmeden önce veri odaklı olarak listelenir.
2. **Proaktif Pazarlama:** Pazarlama birimi, modelin yüksek riskli olarak işaretlediği bu hedef kitleye özel sadakat programları, indirim kuponları veya kazanım (win-back) kampanyaları düzenleyerek müşteri kaybını (churn rate) minimuma indirebilir ve şirket cirosunu korur.