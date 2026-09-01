# GitHub Repository Popularity Predictor

Bu proje, bir GitHub açık kaynak reposunun özelliklerine (kullanılan diller, etiket sayısı, dokümantasyon durumu vb.) bakarak başarılı bir topluluk projesine (400+ yıldız) dönüşüp dönüşmeyeceğini tahmin eden uçtan uca bir makine öğrenmesi boru hattıdır (pipeline).

## Proje Hedefi ve Yaklaşım Değişimi
Başlangıçta projenin net yıldız sayısını tahmin etmesi hedeflenerek bir **Regresyon (Sayı Tahmini)** modeli kurulmuştur. Ancak GitHub ekosistemindeki yıldız dağılımının aşırı sağa çarpık (power-law) olması nedeniyle (projelerin büyük çoğunluğu 0-10 yıldız alırken azınlık bir grubun yüz binlerce yıldız alması), regresyon algoritması ortalama 4600 yıldız (RMSE) hata payı ile çalışmıştır. 

Bu istatistiksel engeli aşmak için problem bir **Sınıflandırma (Classification)** modeline dönüştürülmüştür. 400 yıldız barajı eşik olarak belirlenmiş ve repolar "Popüler/Başarılı (1)" veya "Sıradan (0)" olarak iki net kategoriye ayrılarak hedefe çok daha yüksek bir isabet oranıyla ulaşılmıştır.

## Veri Mühendisliği ve Ön İşleme
Modelin gerçek dünya verilerinde sızıntı (data leakage) olmadan, kararlı bir şekilde çalışabilmesi için `Scikit-Learn Pipeline` kullanılarak modüler bir ön işleme mimarisi kurulmuştur:

*   **Veri Sızıntısının Önlenmesi:** Projenin başlangıç anında bilinmesi imkansız olan `forks` ve `open_issues` gibi hileli özellikler (gelecek verisi) eğitim setinden tamamen çıkarılmıştır.
*   **Özellik Mühendisliği (Feature Engineering):** `description`, `topics` ve `dependencies` gibi doğrudan işlenemeyen karmaşık metin alanlarından `description_length`, `topics_count`, `dependencies_count` gibi makinenin anlayabileceği matematiksel ağırlıklar (özellikler) üretilmiştir.
*   **Mantıksal Dönüşümler:** `has_readme`, `has_license` gibi boolean özellikler standardize edilerek `0` ve `1` formatına çevrilmiştir.
*   **ColumnTransformer Entegrasyonu:**
    *   *Sayısal Veriler:* Eksik değerler veri setinin medyanı (median) ile doldurulup, `StandardScaler` ile ölçeklendirilmiştir.
    *   *Kategorik Veriler:* Eksik veriler en sık tekrar eden (most frequent) değerlerle doldurulmuş; modelin canlı ortamda daha önce hiç görmediği bir yazılım diliyle karşılaşma ihtimaline karşı `OrdinalEncoder` içerisindeki `handle_unknown='use_encoded_value'` parametresiyle bilinmeyen veriler `-1` olarak işaretlenmiştir.

## Makine Öğrenmesi Mimarisi
Tahmin motoru olarak, sınıf dengesizliklerine (imbalanced data) karşı oldukça dirençli olan **XGBoost Classifier** kullanılmıştır. 

Modelin performansı, yanıltıcı olabilen temel `Accuracy` (Doğruluk) metriği üzerinden değil, nadir görülen popüler repoları yakalama gücünü şeffafça gösteren metrikler üzerinden değerlendirilmiştir:
*   **Duyarlılık (Recall):** Potansiyeli olan projeleri gözden kaçırmamak (False Negative hatalarını küçültmek) sistemin birincil önceliği olarak belirlenmiştir.
*   **F1-Score:** Sınıf dengesizliğinin olduğu bu senaryoda, gereksiz yanlış alarmlar (Precision) ile cevherleri yakalama oranı (Recall) arasındaki altın dengeyi ölçmek için hedef metrik olarak konumlandırılmıştır.
*   **Evrensel Karmaşa Matrisi (Confusion Matrix):** Sektör standartlarına uygun olarak Pozitif sınıfın (1) eksenlerde başa alındığı `[[TP, FN], [FP, TN]]` formatında oluşturulmuş ve sonuçların doğrudan okunabilirliği artırılmıştır.

## Optimizasyon ve Doğrulama
Modelin rastgele bir veri bölünmesinde şans eseri yüksek skor almadığından emin olmak ve karar sınırlarını en iyi noktaya çekmek için iki aşamalı bir sınama yapılmıştır:
1.  **5 Katlı Çapraz Doğrulama (Cross-Validation):** Model, veri setinin farklı parçalarında 5 ayrı teste tabi tutulmuş; iterasyonlar arasındaki F1 skorlarının birbirine çok yakın çıkmasıyla sistemin stabilitesi ve ezber (overfitting) yapmadığı kanıtlanmıştır.
2.  **GridSearchCV ile Hiperparametre Ayarı:** `n_estimators` (ağaç sayısı) ve `learning_rate` (öğrenme hızı) gibi kritik parametreler ızgara araması (grid search) yöntemiyle otomatik olarak test edilmiş, F1 skorunu zirveye taşıyan en optimal konfigürasyon tespit edilmiştir.

---
