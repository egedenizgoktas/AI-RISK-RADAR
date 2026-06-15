# AI-RISK-RADAR

**Ülke bazlı AI altyapı riski analizi ve senaryo simülasyonu**

[English version](README.md)

AI-RISK-RADAR, yapay zeka sistemlerine olan talep arttıkça ülkelerin enerji kapasitesi ve dijital altyapı açısından ne kadar baskı altında kalabileceğini analiz etmeyi amaçlayan bir veri bilimi projesidir.

Bu proje; World Bank ülke bazlı göstergeleri, feature engineering, senaryo bazlı stres testi, makine öğrenmesi, Power BI dashboard ve Streamlit prototipini bir araya getirir.

Bu projedeki amaç gerçek bir altyapı çöküş olasılığı tahmin etmek değildir. Amaç, ülkeleri farklı AI talebi ve enerji kapasitesi varsayımları altında karşılaştırmaya yarayan **göreli bir AI altyapı baskı göstergesi** oluşturmaktır.

---

## Proje Motivasyonu

Yapay zeka sistemleri yaygınlaştıkça yalnızca yazılım ve model performansı değil; enerji sistemleri, dijital altyapı, data center kapasitesi ve ülke ölçeğindeki talep baskısı da önemli hale gelmektedir.

Bu projenin temel sorusu şudur:

> AI talebi arttığında hangi ülkeler altyapı ve enerji kapasitesi açısından diğer ülkelere göre daha fazla risk altında kalabilir?

Orijinal veri setinde doğrudan “AI altyapı riski” adında bir değişken bulunmadığı için bu proje, proxy değişkenler oluşturma, risk skoru üretme, stres senaryoları test etme ve sonuçları görselleştirme üzerine kurulmuştur.

Projenin ana katkısı, ham makro ülke göstergelerini yapılandırılmış bir AI altyapı riski analiz çerçevesine dönüştürmesidir.

---

## Proje Akışı

Projede izlenen genel akış:

1. World Bank API üzerinden ülke bazlı makro göstergeler çekildi.
2. Ülke-yıl seviyesindeki veri temizlendi ve ön işlendi.
3. Ülke olmayan bölgesel ve aggregate kayıtlar filtrelendi.
4. AI talebi ve enerji kapasitesi için proxy değişkenler üretildi.
5. Göreli AI altyapı risk skoru oluşturuldu.
6. Veri seti 3x3 senaryo stres testi yapısıyla genişletildi.
7. Ülkeler KMeans ile Low / Medium / High Risk gruplarına ayrıldı.
8. Supervised regression modelleri eğitildi ve karşılaştırıldı.
9. Leakage kontrolü yapılarak final model seçildi.
10. Çıktılar Power BI ile görselleştirildi.
11. Streamlit ile interaktif senaryo simülatörü geliştirildi.

---

## Veri Kaynağı

Veri seti **World Bank API** üzerinden toplanmıştır.

Seçilen dönem:

```text
2015–2025
```

Veri ülke-yıl bazlıdır. Ancak her ülke ve her değişken için tüm yıllar eksiksiz değildir. Bu nedenle proje, seçilen tarih aralığındaki en kullanılabilir gözlemler üzerinden ilerlemiştir.

---

## Ham Değişkenler

World Bank API’den çekilen göstergeler doğrudan AI altyapı riskini temsil etmez. Bu değişkenler makro seviyede proxy göstergeler olarak kullanılmıştır.

| Ham değişken                | Projedeki anlamı                          |
| --------------------------- | ----------------------------------------- |
| `gdp_per_capita`            | Ekonomik kapasite / gelir seviyesi        |
| `population`                | Potansiyel talep ölçeği                   |
| `internet_users_pct`        | Dijitalleşme ve AI adaptasyon potansiyeli |
| `electricity_kwh_pc`        | Enerji kapasitesi                         |
| `renewable_electricity_pct` | Enerji sürdürülebilirliği                 |
| `energy_imports_net_pct`    | Enerji bağımlılığı                        |

Örneğin `population` kelime anlamıyla nüfusu ifade eder. Ancak bu projede potansiyel AI talep ölçeğini temsil eden bir proxy değişken olarak yorumlanmıştır.

---

## Veri Seti Boyutları

Başlangıçtaki birleştirilmiş World Bank veri seti:

```text
2.926 ülke-yıl satırı
8 kolon
```

Ön işleme ve feature engineering sonrası model-ready veri seti:

```text
1.475 ülke-yıl gözlemi
14 kolon
```

3x3 senaryo yapısı sonrası senaryo veri seti:

```text
13.275 ülke-yıl-senaryo gözlemi
21 kolon
```

Bu genişleme şu şekilde hesaplanır:

```text
1.475 model-ready gözlem × 9 senaryo = 13.275 senaryo gözlemi
```

Ülke bazlı özet veri seti:

```text
155 ülke
```

Ülke-senaryo bazlı özet veri seti:

```text
1.395 satır
```

Bu değer şu şekilde hesaplanır:

```text
155 ülke × 9 senaryo = 1.395 ülke-senaryo kaydı
```

---

## Veri Ön İşleme

World Bank verisi doğrudan analiz ve modelleme için hazır değildi. Bu nedenle çeşitli ön işleme adımları uygulandı.

### Wide Format → Long Format

Ham veride yıllar ayrı kolonlar halinde bulunuyordu:

```text
YR2015, YR2016, YR2017, ...
```

Bu yapı ülke-yıl bazlı analiz yapabilmek için long format yapısına dönüştürüldü.

### Göstergelerin Birleştirilmesi

Her gösterge seçildikten sonra `country` ve `year` alanları üzerinden birleştirildi. Böylece her satırı bir ülke-yıl gözlemi olan tek bir veri seti oluşturuldu.

### Eksik Değer Yönetimi

Eksik değerler, değişkenin projedeki rolüne göre farklı şekilde ele alındı.

#### Talep tarafı değişkenleri

AI talep skorunu oluşturmak için kritik olan değişkenler:

* `gdp_per_capita`
* `internet_users_pct`
* `population`

Bu değişkenlerde eksik değer bulunan satırlar veri setinden çıkarıldı. Çünkü talep tarafındaki eksiklikler AI talep skorunu ve risk skorunu bozabilirdi.

#### Kapasite tarafı değişkeni

Enerji kapasitesi temel olarak şu değişkenle temsil edildi:

* `electricity_kwh_pc`

Bu değişken için kontrollü bir imputasyon yöntemi kullanıldı:

1. `gdp_per_capita` eksikleri median ile dolduruldu.
2. Ülkeler GDP seviyelerine göre `low`, `mid`, `high` gruplarına ayrıldı.
3. `electricity_kwh_pc` eksikleri önce ilgili GDP grubunun ortalamasıyla dolduruldu.
4. Kalan eksikler genel median ile tamamlandı.

Bu yaklaşım, kişi başı elektrik tüketimi eksiklerini ülkelerin ekonomik benzerliğini dikkate alarak doldurmayı amaçladı.

---

## Ülke Olmayan Kayıtların Filtrelenmesi

World Bank API yalnızca ülkeleri değil, bölgesel ve aggregate kayıtları da döndürmektedir.

Filtrelenen ülke dışı kayıtlara örnekler:

* World
* High income
* Low income
* Middle income
* Europe & Central Asia
* Sub-Saharan Africa
* Latin America & Caribbean
* Fragile states
* Small states

Bu kayıtlar gerçek ülke olmadığı için analizden çıkarılmıştır. Böylece modelleme ve harita görselleştirmeleri gerçek ülkeler üzerinden yapılmıştır.

---

## Feature Engineering

Orijinal World Bank göstergeleri doğrudan AI altyapı riskini ölçmez. Bu nedenle feature engineering projenin en kritik adımlarından biridir.

Amaç, ham makro göstergeleri proje problemine daha uygun değişkenlere dönüştürmektir.

---

## AI Talep Skoru

AI talebi veri setinde doğrudan bulunmadığı için proxy bir AI talep skoru oluşturuldu.

Mantık:

* Kişi başı GDP arttıkça ekonomik ve yatırım kapasitesi artabilir.
* İnternet kullanım oranı arttıkça dijitalleşme ve AI adaptasyon potansiyeli artabilir.
* Nüfus arttıkça potansiyel kullanıcı ve talep ölçeği büyüyebilir.

Ağırlıklandırmadan önce değişkenlere `log1p` dönüşümü ve standardizasyon uygulandı.

Ağırlıklı AI talep skoru şu şekilde hesaplandı:

| Değişken             | Ağırlık |
| -------------------- | ------: |
| `gdp_per_capita`     |     %40 |
| `internet_users_pct` |     %35 |
| `population`         |     %25 |

Oluşturulan değişken:

```text
ai_demand_score_weighted
```

---

## Enerji Kapasitesi Skoru

Enerji kapasitesi temel olarak şu değişken üzerinden temsil edildi:

```text
electricity_kwh_pc
```

AI altyapısı, data center’lar ve hesaplama sistemleri enerjiye bağlı olduğu için kişi başı elektrik tüketimi enerji kapasitesi proxy’si olarak kullanıldı.

Enerji kapasitesi önce standardize edildi, sonra pozitif ölçeğe taşındı.

Önemli türetilmiş değişkenler:

```text
energy_capacity_score
energy_capacity_pos
```

`energy_capacity_pos`, standardizasyon sonrası oluşabilecek negatif değerler nedeniyle oluşturuldu. Risk formülünde enerji kapasitesi paydada yer aldığı için pozitif bir kapasite ölçeği kullanmak daha güvenli hale geldi.

---

## AI Risk Score

Temel risk mantığı:

```text
AI Talebi ↑
Enerji Kapasitesi ↓
= AI Altyapı Riski ↑
```

Basitleştirilmiş formül:

```text
AI Risk Score = AI Demand Score / Energy Capacity
```

İlk senaryosuz risk değişkeni:

```text
ai_risk_score
```

Bu skor gerçek bir çöküş olasılığı değildir. Ülkeler arası göreli AI altyapı baskısını temsil eden karşılaştırmalı bir göstergedir.

---

## Senaryo Analizi

Projede 3x3 senaryo stres testi kullanıldı.

### AI Talebi Senaryoları

| Senaryo           | Çarpan | Anlamı                 |
| ----------------- | -----: | ---------------------- |
| `baseline_demand` |   1.00 | Mevcut talep           |
| `high_demand`     |   1.50 | AI talebi %50 artıyor  |
| `extreme_demand`  |   2.00 | AI talebi %100 artıyor |

### Enerji Kapasitesi Senaryoları

| Senaryo             | Çarpan | Anlamı                             |
| ------------------- | -----: | ---------------------------------- |
| `baseline_energy`   |   1.00 | Mevcut enerji kapasitesi           |
| `low_energy`        |   0.50 | Enerji kapasitesi %50 baskılanıyor |
| `severe_low_energy` |   0.25 | Enerji kapasitesi %75 baskılanıyor |

İki boyut çaprazlandı:

```text
3 talep senaryosu × 3 enerji senaryosu = 9 senaryo
```

Bu yapı veri setini şu şekilde genişletti:

```text
1.475 ülke-yıl gözlemi
```

şu hale geldi:

```text
13.275 ülke-yıl-senaryo gözlemi
```

Ana senaryo bazlı risk değişkeni:

```text
scenario_risk_score
```

---

## Risk Segmentasyonu

Ülkeler KMeans clustering yöntemiyle üç risk grubuna ayrıldı:

* Low Risk
* Medium Risk
* High Risk

Segmentasyon ülke bazlı ortalama senaryo risk skorları üzerinden yapıldı.

Bu yapı, Power BI haritalarında ve dashboard görsellerinde sonuçların daha anlaşılır yorumlanmasını sağladı.

---

## Supervised Modelleme

Senaryo risk skorunu tahmin etmek için supervised regression modelleri karşılaştırıldı.

Denenen model aileleri:

* Linear Regression
* Ridge Regression
* Lasso Regression
* KNN
* Decision Tree
* Random Forest
* Extra Trees
* Gradient Boosting
* XGBoost
* LightGBM
* CatBoost
* Voting Regressor
* Stacking Regressor

Değerlendirme metrikleri:

* RMSE
* MAE
* R²

Final model:

```text
Tuned LightGBM Regressor
```

Final model performansı:

| Model          |   RMSE |    MAE |     R² |
| -------------- | -----: | -----: | -----: |
| LightGBM Tuned | 0.0665 | 0.0428 | 0.9991 |

---

## Leakage Kontrolü

Projede target leakage riskine özellikle dikkat edildi.

Risk skorunu doğrudan oluşturan bazı ara değişkenler final model inputu olarak kullanılmadı.

Leakage riski nedeniyle dışarıda bırakılan değişkenler:

```text
ai_demand_score_weighted
energy_capacity_score
energy_capacity_pos
ai_risk_score
scenario_demand
scenario_energy_capacity
scenario_risk_score
risk_cluster
risk_level
```

Final modelde kullanılan 6 açıklanabilir input değişken:

| Final model değişkeni | Anlamı                      |
| --------------------- | --------------------------- |
| `gdp_per_capita`      | Ekonomik kapasite           |
| `internet_users_pct`  | Dijitalleşme                |
| `population`          | Talep ölçeği                |
| `electricity_kwh_pc`  | Enerji kapasitesi           |
| `demand_multiplier`   | AI talep senaryosu          |
| `energy_multiplier`   | Enerji kapasitesi senaryosu |

Bu yaklaşım leakage riskini azaltmak ve modeli daha açıklanabilir hale getirmek için tercih edildi.

---

## Feature Importance

Final modelin hangi değişkenlere daha çok dayandığını görmek için feature importance analizi yapıldı.

Permutation feature importance sonuçlarına göre en önemli değişkenler:

1. `internet_users_pct`
2. `energy_multiplier`
3. `gdp_per_capita`
4. `demand_multiplier`
5. `population`
6. `electricity_kwh_pc`

`internet_users_pct` değişkeninin yüksek öneme sahip olması, dijitalleşme seviyesinin modelin risk tahmininde güçlü bir rol oynadığını gösterir.

`energy_multiplier` ve `demand_multiplier` değişkenlerinin önemli olması, modelin stres senaryolarına güçlü şekilde tepki verdiğini gösterir.

`population` değişkeninin önemli olması ise “kalabalık ülkeler otomatik risklidir” anlamına gelmez. Bu projede population, potansiyel AI talep ölçeğini temsil eden bir proxy değişken olarak kullanılmıştır.

---

## Power BI Dashboard

Power BI ile proje çıktıları görselleştirildi.

Dashboard içinde:

* Risk seviyesi dünya haritası
* Senaryo bazlı AI altyapı risk haritası
* En riskli ilk 10 ülke
* Senaryo bazlı en riskli ilk 10 ülke
* Low / Medium / High risk segmentasyonu

yer almaktadır.

İki farklı bakış açısı kullanıldı:

1. **Baseline / mevcut risk görünümü**
   Mevcut veya stres uygulanmamış risk profilini gösterir.

2. **Senaryo bazlı risk görünümü**
   AI talebi ve enerji kapasitesi stres senaryoları dahil edildiğinde risk sıralamasının nasıl değiştiğini gösterir.

Bu yapı, ülkelerin mevcut koşullarda ve stres senaryoları altında nasıl farklılaştığını görmeyi sağlar.

---

## Streamlit Prototipi

Streamlit ile interaktif bir senaryo simülatörü geliştirildi.

Streamlit uygulamasında kullanıcı:

* ülke seçebilir,
* AI talep değişimini ayarlayabilir,
* enerji kapasitesi değişimini ayarlayabilir,
* tahmini risk skorunu görebilir,
* normalize edilmiş risk indeksini inceleyebilir,
* ülke bazlı senaryo yorumunu görebilir.

Bu yapı projeyi yalnızca statik bir notebook analizi olmaktan çıkarıp interaktif bir senaryo simülasyon prototipine dönüştürür.

---

## Çıktı Dosyaları

Repository içinde yer alan çıktı dosyaları:

| Dosya                                   | Açıklama                                             |
| --------------------------------------- | ---------------------------------------------------- |
| `01_merged_worldbank_data.csv`          | Birleştirilmiş World Bank ülke-yıl verisi            |
| `02_model_ready_data.csv`               | Temizlenmiş model-ready veri seti                    |
| `03_scenario_results.csv`               | Senaryo ile genişletilmiş ülke-yıl-senaryo veri seti |
| `04_country_risk_results.csv`           | Ülke bazlı ortalama risk sonuçları ve segmentler     |
| `05_top10_risk_countries.csv`           | Risk skoruna göre en riskli ilk 10 ülke              |
| `06_high_risk_top10.csv`                | High Risk segmentindeki öne çıkan ülkeler            |
| `07_medium_risk_top10.csv`              | Medium Risk segmentindeki öne çıkan ülkeler          |
| `08_low_risk_top10.csv`                 | Low Risk segmentindeki öne çıkan ülkeler             |
| `09_country_scenario_risk_results.csv`  | Ülke-senaryo bazlı risk sonuçları                    |
| `10_initial_model_comparison.csv`       | İlk model karşılaştırma sonuçları                    |
| `11_all_model_comparison.csv`           | Tüm model karşılaştırma sonuçları                    |
| `12_final_model_comparison.csv`         | Final model karşılaştırma sonuçları                  |
| `13_feature_importance_native.csv`      | Native feature importance sonuçları                  |
| `14_feature_importance_permutation.csv` | Permutation feature importance sonuçları             |

---

## Kullanılan Teknolojiler

* Python
* Pandas
* NumPy
* Scikit-learn
* LightGBM
* XGBoost
* CatBoost
* KMeans
* World Bank API
* Streamlit
* Power BI
* GitHub

---

## Risk Skoru Nasıl Yorumlanmalı?

Risk skoru şu anlama gelmez:

```text
“Bu ülkede %X ihtimalle altyapı çöker.”
```

Doğru yorum şudur:

```text
“Bu ülke, seçilen varsayımlar altında diğer ülkelere göre daha yüksek veya daha düşük AI altyapı baskısı taşıyor.”
```

Bu skor şu amaçlarla kullanılabilir:

* ülkeleri karşılaştırmak,
* göreli altyapı baskısını görmek,
* farklı senaryoları test etmek,
* risk sıralamalarındaki değişimi incelemek,
* erken aşama altyapı risk analizi yapmak.

---

## Sınırlılıklar

Bu proje bir prototiptir ve bazı sınırlılıklara sahiptir.

Mevcut projede yer almayan önemli boyutlar:

* gerçek data center lokasyonları,
* su stresi,
* soğutma ihtiyacı,
* elektrik şebekesi dayanıklılığı,
* enerji fiyatları,
* karbon yoğunluğu,
* gerçek zamanlı AI yatırım verileri,
* ülke içi bölgesel farklılıklar.

Bu nedenle mevcut versiyon, nihai ticari karar sistemi olarak değil; erken aşama risk skorlama ve senaryo simülasyonu çalışması olarak değerlendirilmelidir.

---

## Future Work

Gelecek versiyonlarda şu geliştirmeler yapılabilir:

* su stresi ve soğutma ihtiyacı göstergelerinin eklenmesi,
* gerçek data center lokasyonlarının dahil edilmesi,
* elektrik şebekesi dayanıklılığı metriklerinin eklenmesi,
* enerji fiyatı ve karbon yoğunluğu verilerinin eklenmesi,
* yenilenebilir enerji kapasitesinin daha güçlü temsil edilmesi,
* pozitif altyapı iyileştirme senaryolarının test edilmesi,
* bölge veya şehir bazlı analiz yapılması,
* otomatik güncellenen veri pipeline’ı kurulması,
* daha gelişmiş senaryo simülasyonu yapılması.

Uzun vadede AI-RISK-RADAR, ülkelerin AI büyümesini enerji ve dijital altyapı açısından ne kadar taşıyabileceğini izleyen bir erken uyarı ve senaryo simülasyon platformuna dönüşebilir.

---

## Projenin Konumu

Bu proje doğrudan yatırım önerisi veren bir sistem değildir.

Daha doğru tanım:

> Ülke bazlı AI altyapı baskısını veriyle görünür hale getiren bir risk skorlama ve senaryo simülasyon prototipi.

Projenin ana katkısı, ham makroekonomik ve altyapı göstergelerini kullanarak AI altyapı riskini temsil eden ölçülebilir bir analiz yapısı kurmasıdır.
