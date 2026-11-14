# Güvenli Okul — Bildirim ve Takip Platformu (v2)

Okullarda akran zorbalığı bildirimlerini toplamak, **insan incelemesi** için önceliklendirmek ve süreci
izlenebilir kılmak için hazırlanmış bir Streamlit prototipidir. Karar veren bir sistem **değildir**.

## Çalıştırma
```bash
python -m venv .venv && source .venv/bin/activate      # Windows: .venv\Scripts\activate
pip install -r requirements.txt
cp .streamlit/secrets.toml.example .streamlit/secrets.toml
python tools/parola_hash.py                            # çıkan password_hash'i secrets.toml'a yazın
streamlit run app.py
```
Testler (ek bağımlılık gerekmez): `python -m unittest discover -s testler -v`

## Yapı
| Dosya | Görev |
|---|---|
| `app.py` | Arayüz ve sayfa akışı |
| `db.py` | SQLite katmanı: şema göçü, işlemler, istatistik, CSV, saklama süresi |
| `guvenlik.py` | Parola özeti/doğrulama, kişisel veri maskeleme, CSV enjeksiyon koruması |
| `hakaret_tespiti.py` | Kural tabanlı sinyal: kategori, öncelik, bağlam uyarısı |
| `degerlendirme.py` | Sentetik etiketli küme + precision/recall/F1 |
| `stil.py` | CSS (harici yazı tipi/CDN yok) |
| `ek_sayfalar.py` | Destek rehberi, trend/ortam analizi, gizlilik ve KVKK paneli (onam akışı simülasyonu) |
| `i18n.py` | 7 dil için merkezi çeviri altyapısı, temel arayüz sözlüğü, tarih-saat biçimi ve RTL yön bilgisi |
| `testler/` | Birim testleri |

## v1 → v2: düzeltilen hatalar
- Türkçe karakterli kullanıcı adı/parola `hmac.compare_digest` ile `TypeError` veriyordu → UTF-8 bayt karşılaştırması.
- Sayfa düğmeleri `session_state["ana_menu"]` değerini widget oluştuktan sonra değiştiriyordu → `on_click` geri çağrıları.
- `st.success(); st.rerun()` mesajı yutuyordu → `flash` mekanizması.
- `clear_on_submit=True` doğrulama hatasında da yazılan metni siliyordu → yalnızca başarıda form sıfırlanır.
- `with sqlite3.connect()` bağlantıyı kapatmıyordu → her işlemde kapatan bağlam yöneticisi.
- Saatler sunucu saatindeydi (bulutta UTC) → Europe/Istanbul.
- `ilk_acilista_kayitlari_temizle` bayrak yoksa TÜM kayıtları siliyordu → kaldırıldı; yerine şema sürümü + göç.
- Algılayıcı "gerizekalı" yazımını kaçırıyordu (sözlükte ASCII) → Türkçe'ye duyarlı katlama.
- Bozuk Markdown (`** Gizlilik**`), boş ikon kutuları, çift "10." bölüm numarası, harici Google Fonts çağrısı.

## Yeni özellikler
Öncelik sinyali ve kuyruk sıralaması · kişisel veri maskeleme (T.C. no sağlama toplamı dâhil) · takip kodu ile
durum sorgulama · hatalı giriş kilidi + oturum zaman aşımı + PBKDF2 parola özeti · denetim kaydı ·
yönetici işlemlerinde "kim yaptı" bilgisi · CSV dışa aktarım (kimlik varsayılan hariç) · saklama süresi temizliği ·
anonim kayıtlarda saniye hassasiyetli zaman damgası · yanıt süresi metrikleri · otomatik algılayıcı değerlendirmesi.

## Bilimsel doğrulama yol haritası (TÜBİTAK için)
1. Araştırma sorusu ve hipotezlerin yazılması (ör. "Kural tabanlı sinyal, yönetici inceleme sırasını iyileştirir mi?").
2. Etik kurul / okul ve veli izniyle, kimliksizleştirilmiş gerçek veya uzmanca üretilmiş bir veri kümesi.
3. En az iki uzmanın (rehber öğretmen/psikolojik danışman) bağımsız etiketlemesi; Cohen kappa raporu.
4. Eğitim/doğrulama/test ayrımı; kural tabanlı yöntemin başka bir yöntemle (ör. sınıflandırıcı) karşılaştırması.
5. Kullanılabilirlik testi (SUS) ve yanıt süresi ölçümü.
6. Yanlış alarm/kaçırma analizi ve sınırlılıkların açıkça raporlanması.

## Yayına almadan önce
KVKK (aydınlatma, hukuki sebep, saklama-imha, yurt dışı sunucu), kişi bazlı hesap + 2FA, HTTPS, şifreli yedek,
bağımsız güvenlik testi ve okul acil durum protokolü gereklidir. Acil hat numaralarını (`sabitler.py`) resmi
kaynaklardan yeniden doğrulayın.

## v2.1 — Tam anonim bildirim ve sadeleştirilmiş akış
Artık tek bir **Yeni bildirim** ve tek bir **Bildirim sorgula** sayfası var (eski yinelenen sayfalar kaldırıldı).

- **Tam anonim:** formda ad, soyad, sınıf, telefon istenmez ve kaydedilmez; zaman dakika/saniye hassasiyetinde saklanır.
- **Günlük kota:** `app.py` içindeki küçük bir yerel bileşen (çalışma anında geçici klasöre yazılır, ayrı dosya gerekmez); rastgele cihaz jetonunu tarayıcının localStorage'ında
  (yedek: çerez) saklayıp uygulamaya iletir, sayfa yenilense de aynı kalır. Veritabanında jetonun SHA-256 özeti ve gün (`anonim_cihaz`) saklanır; rapora
  bağlanmaz, gün bitince silinir. Aynı gün ikinci bildirimde “Günlük bildirim hakkınız dolmuştur.” görünür.
- **Gizli takip kodu:** `GO-XXXXXXXXXX` (~50 bit). Sorgu sayfası durumu, kısa bir açıklamasını ve tarihleri gösterir.
- **Ek bilgi:** aynı sorgu sayfasında; yalnızca anonim kayıtlarda, bildirimin günü; 60 sn bekleme ve günde en çok 3 ek
  **sunucuda** uygulanır. Ek metin maskelenir, analiz edilir; daha yüksek sinyal çıkarsa kayıt önceliği yükselir.
  İnceleme ekranında “Anonim gönderenin ek bilgileri” ve kuyrukta “+N ek bilgi” görünür.
- **Mesajlaşma yok:** öğrenci yalnızca durumu görür; yönetici notları gösterilmez.
- **Telefon uyumu:** seçenekler alt alta dokunması kolay kartlar, büyük düğmeler, iOS'ta yakınlaştırmayı önleyen yazı
  boyutu, dar ekranda tek sütun.

Bilinen sınırlar: kota tarayıcı çerezine dayanır; çerez silme/gizli sekme ile aşılabilir (hesapsız anonimlikte kesin
kota mümkün değildir). Streamlit Cloud'da SQLite dosyası kalıcı değildir; gerçek kullanımda harici veritabanı gerekir.

## v2.3 — Modern arayüz ve ek sayfalar
- Gradyanlı karşılama alanı ve satır içi SVG illüstrasyon/ikon rozetleri (harici resim, CDN, yazı tipi yok).
- Logo kenar çubuğuna gömülü (base64) olarak yerleştirilir; ek istek yapılmaz.
- Acil hatlar: 112, 183, 444 0 632 (MEBİM). Yayından önce resmî kaynaklardan yeniden doğrulayın.
- Yeni sayfalar: **Destek rehberi** (herkese açık), **Trend ve ortam** (yönetici; küçük hücre bastırmalı),
  **Gizlilik ve KVKK** (aydınlatma taslağı, veri envanteri, saklama/imha, onam akışı simülasyonu).
- Güvenlik düzeltmeleri: kullanıcı metni artık Markdown olarak yorumlanmaz (görsel/bağlantı enjeksiyonu önlenir);
  ek bilgisi olan sonuçlanmış kayıtlar saklama süresi sonunda hatasız silinir; günlük kota denetimi salt okunurdur.


## Ek geliştirmeler
- Arayüzde yedi dil desteği: Türkçe, English, Deutsch, Français, Español, العربية ve Azərbaycan dili.
- Yönetici genel bakış ve inceleme ekranında günlük özet ve yüksek/acil uyarı bandı.
- İnceleme kuyruğunda tarih aralığı filtresi ve bekleme süresi / 24 saat gecikme göstergesi.
- Kategoriye göre rehberlik kontrol listesi.
- Yeni anonim bildirimler ve ek bilgiler için Europe/Istanbul saatinde saniye hassasiyetli zaman damgası.

Not: Eski veritabanında daha önce saat başına yuvarlanmış zaman damgalarının kaybolan dakika/saniye kısmı geriye dönük üretilemez. Yeni kayıtlar doğru hassasiyetle saklanır.


### Uygulama notları
- Dil seçimi Streamlit oturumunda korunur. Menü, formlar, seçenekler, yönetici ekranları, rehberlik içerikleri, uyarılar, tablolar ve bilgilendirme metinleri merkezi `i18n.py` kataloğundan çevrilir.
- SLA süresi her sayfa yenilemesinde/etkileşimde yeniden hesaplanır. Canlı saniyelik otomatik yenileme kullanılmaz; gereksiz arka plan yenilemesi ve veritabanı yükü oluşturulmaz.
- Rehberlik önerileri karar veya klinik yönlendirme değildir; okul psikolojik danışmanı tarafından onaylanmalıdır.


## Çok dilli arayüz desteği

- Kenar çubuğundan Türkçe, English, Deutsch, Français, Español, العربية veya Azərbaycan dili seçilebilir.
- Menü, bildirim formu ve seçenekleri, yönetici inceleme ekranları, istatistikler, trend/ortam analizi, Etki Merkezi, destek rehberi, güvenli kullanım, gizlilik/KVKK ve onam simülasyonu için yerelleştirme kataloğu kullanılır.
- Kullanıcıların yazdığı bildirim açıklamaları, yönetici notları ve Etki Merkezi çalışma başlığı/açıklaması/sorumlu kişi alanları otomatik çevrilmez; özgün metinleri korunur.
- Okulun resmî adı (**Bozüyük Fen Lisesi**) ve proje adı (**Güvenli Okul**) korunur. Arapça arayüz RTL yönünü, takip kodları ise LTR yönünü kullanır.
- Bilinen arayüz anahtarında hedef dil çevirisi henüz yoksa Türkçe metin sızdırmak yerine İngilizce karşılık gösterilir. Katalogda bulunmayan serbest metin olduğu gibi bırakılır; otomatik makine çevirisi yapılmaz.
- Tarih/saat gösterimleri dil bazında biçimlendirilir; veritabanındaki zaman damgaları değiştirilmez.
- Çeviriler uygulama içi kontrollerle sınanmıştır; yayımlanmadan önce özellikle hukuki ve öğrenci destek metinlerinin ilgili dillerde ana dili konuşan kişilerce son okuması önerilir.

## v2.4 — Çoklu dil altyapısı (TR / EN / DE / FR / ES / AR / AZ)
- Dil seçimi Türkçe, English, Deutsch, Français, Español, العربية ve Azərbaycan dili seçeneklerini içerir.
- Çeviri çağrıları merkezi `i18n.py` modülüne yönlendirilir; `set_language()` ile oturumun seçili dili uygulama genelinde kullanılır.
- Arapça seçildiğinde arayüz için RTL (sağdan sola) yön kuralları uygulanır; takip kodu gibi Latin karakterli alanlar LTR tutulur.
- Tarih/saat gösterimleri dil bazında biçimlendirilir; veritabanındaki zaman damgaları değiştirilmez.
- Serbest kullanıcı açıklamaları, yönetici notları ve okul/proje adları otomatik çevrilmez.

**Çeviri kapsamı ve doğrulama:** Bu sürümde beş ek dilin katalogları ana sayfa ile sınırlı kalmayacak şekilde genişletilmiştir. Destek rehberi, trend/ortam, yönetici inceleme, Etki Merkezi, güvenli kullanım, araştırma, gizlilik/KVKK bildirimi ve onam formu metinleri de kapsama alınmıştır. Otomatik testler çeviri anahtarlarının doluluğunu, önemli ekran metinlerini ve serbest kullanıcı metinlerinin değiştirilmemesini denetler. Bu kontroller profesyonel ana dili redaksiyonunun yerine geçmez; özellikle hukuki ve öğrenci destek metinleri gerçek kullanım öncesinde yetkili kişilerce gözden geçirilmelidir.

### Çok dilli doğrulama
- Olay türü, ortam, zaman, tekrar sıklığı, birim, durum ve öncelik seçenekleri beş ek dilde kontrol edilir.
- Takip kodu, ek bilgi, günlük sınır, yönetici oturumu, inceleme, trend, Etki Merkezi, gizlilik ve onam akışı için çeviri testleri bulunur.
- Kullanıcı tarafından yazılan açıklama ve notlar otomatik çevrilmez.
- Son kontrol: `python -m unittest discover -s testler -v`.

## v2.5 — Etki Merkezi (yöneticiye özel)
- Yönetim menüsüne yalnızca yönetici oturumuyla açılan **Etki Merkezi** eklendi.
- Yönetici önleyici çalışma başlığı, açıklama/hedef, ilişkili konu, hedef ortam, sorumlu öğretmen/yetkili kişi, sorumlu birim ve planlanan tarihi kendisi girer.
- Çalışmalar planlandı / devam ediyor / tamamlandı / iptal edildi durumlarıyla takip edilir; sorumlu ve durum değişiklikleri ile değerlendirme notları ayrı işlem geçmişine yazılır.
- Tamamlanma tarihinden önceki ve sonraki 30 günlük toplu bildirim sayıları, her iki dönemde de en az 5 kayıt varsa gösterilir. Sonraki 30 gün tamamlanmadan kıyaslama yapılmaz; küçük sayılar gizlenir.
- Bu karşılaştırma yalnızca bildirim adetlerindeki gözlemsel değişimi gösterir. Olayların gerçek sıklığını veya çalışmanın nedensel etkisini kanıtlamaz.
- Öğrenci isimleri/sınıfları ve tekil bildirim açıklamaları Etki Merkezi analizinde kullanılmaz. Bu demo sürümünde sorumlu alanları için de gerçek kişi bilgisi yerine örnek veri kullanılmalıdır.
- Etki Merkezi arayüzü ve durum metinleri yedi dilde yerelleştirilir. Yönetici tarafından girilen çalışma başlığı, açıklaması, sorumlu adı ve notlar otomatik çevrilmez.
- Veritabanına mevcut kayıtları silmeden iki yeni tablo eklenir: `onleyici_calismalar` ve `onleyici_calisma_gecmisi`.
