# Kod İnceleme Raporu — pvc-erp

İncelenen dosyalar: `index.html`, `index (1).html`, `README.md` ve git geçmişi.

Uygulama tamamen istemci tarafında çalışan tek bir HTML sayfası: sunucu, veritabanı ya da derleme adımı yok. Hesaplama mantığının tamamı `index.html` içindeki satır içi `<script>` bloğunda.

Bu PR'da **yapılan** küçük ve güvenli düzeltmeler en altta, "Uygulanan Düzeltmeler" bölümünde. Aşağıdaki maddeler ise risk taşıdığı, mimari değişiklik gerektirdiği ya da iş kararı olduğu için **yapılmadı**, sadece raporlandı.

---

## 🔴 KRİTİK

### K1. Giriş bilgileri kaynak kodda açık metin olarak duruyor
- **Yer:** `index.html`, `doLogin()` fonksiyonu
- **Sorun:** İki kullanıcının e-posta adresi (`erdal@geoser.net`, `cengizhan@geoser.net`) ve ortak şifre (`geoser2026`) JavaScript içinde düz metin olarak yazılı. Sayfayı açan herkes "Kaynağı Görüntüle" ile bunları okuyabilir. Bu bilgiler git geçmişine de girdi. Kod şimdi düzeltilse bile eski commit'lerde kalmaya devam eder.
- **Öneri:**
  1. Bu şifre başka bir yerde (e-posta, sunucu, muhasebe yazılımı vb.) kullanılıyorsa **hemen değiştirilmeli**.
  2. Repo herkese açıksa gizli (private) yapılmalı.
  3. Gerçek bir kimlik doğrulama için sunucu tarafı gerekli (aşağıdaki K2'ye bakın).
  4. Geçmişi temizlemek (`git filter-repo` / BFG) mümkün ama yıkıcı bir işlem. Şifre değiştirildikten sonra buna gerek kalmayabilir.
- **Neden yapılmadı:** Şifreyi hash'e çevirmek (ör. SHA-256 ile karşılaştırmak) güvenlik sağlamaz. Hash'in kendisi kaba kuvvetle kırılabilir, zaten K2'deki yöntemle giriş tamamen atlanabilir. Doğru çözüm backend gerektiriyor.

### K2. Giriş ekranı gerçek bir koruma sağlamıyor
- **Yer:** `index.html`, `#loginScreen` / `#appMain`
- **Sorun:** "Giriş" işlemi sadece bir `div`'i gizleyip diğerini gösteriyor. Tarayıcı konsolunda `document.getElementById('appMain').style.display='block'` yazmak ya da dosyayı açıp HTML'i okumak tüm uygulamaya erişmek için yeterli. Hammadde fiyatları, reçete (PHR oranları) ve kâr marjı gibi ticari sırlar sayfaya giriş yapmadan da okunabiliyor.
- **Ek kanıt:** `index (1).html` dosyası, giriş ekranı `display:none` ve uygulama `display:block` hâliyle kaydedilmiş bir sayfa görüntüsü. Yani korumanın atlanabildiği fiilen görülüyor.
- **Öneri:** Hassas veriler (reçete, maliyetler) istemciye gönderilmemeli ya da sayfa bir kimlik doğrulama katmanının arkasında sunulmalı. Örnekler: sunucu tarafı oturum; Cloudflare Access / Netlify Identity / basit bir HTTP Basic Auth. Bu bir mimari değişiklik olduğu için yapılmadı.

---

## 🟠 YÜKSEK

### Y1. `index (1).html` bozuk ve eski bir kopya
- **Sorun:** Bu dosya bir tarayıcı "sayfayı farklı kaydet" çıktısı ve **hiç `<script>` içermiyor**. `doLogin`, `rc`, `uM`, `tglM`, `gQ`, `eX` fonksiyonları tanımlı değil. Tüm butonlar ve giriş alanları tıklandığında `ReferenceError` veriyor. Giriş ekranı da gizli kaydedilmiş (bkz. K2). Ayrıca dosya adındaki boşluk ve parantez URL'lerde sorun çıkarır.
- **Öneri:** Bu tasarım (emoji başlıklar, Türkçe karakterler, `₺`/`€` sembolleri) isteniyorsa görsel kısmı `index.html`'e taşınmalı ve `index (1).html` silinmeli. İstenmiyorsa dosya doğrudan silinmeli.
- **Neden yapılmadı:** Dosya silmek ve iki tasarımı birleştirmek hangi sürümün "doğru" olduğuna dair bir karar gerektiriyor.

### Y2. Döviz kurları sabit ve çok eski
- **Yer:** `#uR` (USD/TL = 18.79), `#eR` (EUR/TL = 20.38)
- **Sorun:** Bu değerler 2022–2023 dönemine ait. Kullanıcı her seferinde elle güncellemezse TL ve € fiyatları, teklifler ve kâr hesabı **ciddi şekilde yanlış** çıkar. Değerler sayfa yenilendiğinde de sıfırlanıyor.
- **Öneri:** Kur, güvenilir bir kaynaktan (ör. TCMB XML) otomatik çekilmeli ya da en azından son girilen değer `localStorage`'da saklanmalı. Ekranda "kur tarihi" de gösterilmeli.
- **Neden yapılmadı:** Varsayılan değerleri değiştirmek mevcut hesap çıktılarını değiştirir. Harici API eklemek de kapsam dışı.

---

## 🟡 ORTA

### O1. Üçüncü taraf ses dosyası (hotlink)
- **Yer:** `tglM()` → `https://cdn.pixabay.com/...mp3` (`index (1).html` içinde de `soundhelix.com`)
- **Sorun:** Müzik butonu başka bir sitenin sunucusundan dosya çekiyor. Bu, kullanıcının IP adresini üçüncü tarafa iletir. Link her an kırılabilir ya da içerik değişebilir. İş uygulamasında gereksiz bir bağımlılık.
- **Öneri:** Özellik kaldırılmalı ya da ses dosyası repoya eklenip yerelden sunulmalı.

### O2. Content-Security-Policy uygulanamıyor
- **Sorun:** Tüm olaylar satır içi `onclick=` / `onchange=` ile bağlanmış ve JavaScript satır içi `<script>` içinde. Bu yapıyla XSS'e karşı etkili bir CSP (`script-src 'self'`) tanımlanamaz.
- **Öneri:** JS ayrı bir `app.js` dosyasına taşınmalı, olaylar `addEventListener` ile bağlanmalı, sonra CSP eklenmeli. Bu, kodun büyük kısmının yeniden yazılması demek.

### O3. Teklif formundaki alanların çoğu kullanılmıyor
- **Yer:** `gQ()`
- **Sorun:** Firma, Telefon, E-posta, Proje, Tarih ve Not alanları dolduruluyor ama oluşturulan teklifte **yalnızca Müşteri adı** görünüyor. Kullanıcı bu bilgilerin teklife girdiğini sanabilir.
- **Öneri:** Alanlar teklif çıktısına eklenmeli (`esc()` ile kaçırılarak).

### O4. "Yazdır" tüm uygulamayı yazdırıyor
- **Sorun:** `window.print()` için bir `@media print` stili yok. Sadece teklif değil, koyu arka planlı bütün sayfa (maliyetler ve kâr marjı dahil) yazdırılıyor. Teklif müşteriye gidecekse iç maliyet bilgisi de sızmış olur.
- **Öneri:** Yazdırırken yalnızca `#qP` alanını gösteren bir `@media print` kuralı eklenmeli.

### O5. Kullanıcı girişleri doğrulanmıyor
- **Sorun:** PHR, $/kg, kâr % ve metraj alanlarına negatif değer girilebiliyor. Negatif değerler grafikte ve toplamlarda anlamsız sonuç veriyor. Kur alanı `0` ya da boş bırakılırsa sessizce `1` kabul ediliyor (bu davranış korundu), bu yüzden kullanıcı yanlış kurla hesap yapıldığını fark etmeyebilir.
- **Öneri:** `min="0"` ve uyarı mesajları eklenmeli.

---

## 🟢 DÜŞÜK

- **D1. Kullanılmayan alan:** "Rulo" (`#rw`) girdisi hiçbir hesapta kullanılmıyor. Ya kaldırılmalı ya da hesaba (rulo başına m²/kg) dahil edilmeli.
- **D2. Liste fiyatı sabit:** Fiyat listesindeki "L$" sütunu koda gömülü `×1.1` çarpanıyla hesaplanıyor. Bu oran bir ayar alanı olmalı.
- **D3. Veriler kalıcı değil:** Hammadde fiyatları ve reçetede yapılan değişiklikler sayfa yenilenince kayboluyor.
- **D4. Mobil uyum:** `<meta name="viewport">` yok ve yerleşim sabit `flex` yapısında. Telefonda okunması zor.
- **D5. Türkçe karakter eksikliği:** `index.html`'deki metinler ASCII'ye çevrilmiş ("Sifre", "Giris", "Muzik", "Ozgul"). `charset=UTF-8` zaten tanımlı olduğu için Türkçe karakterler güvenle kullanılabilir.
- **D6. Grafik lejantı taşabilir:** Kalem sayısı arttıkça lejant, sabit 200 px yüksekliğindeki canvas'ın dışına taşar.
- **D7. Erişilebilirlik:** Giriş alanlarının `<label>` karşılığı yok ve giriş bir `<form>` içinde değil. Tarayıcı şifre yöneticileri düzgün çalışmayabilir.
- **D8. README boş:** Projenin ne olduğu, nasıl açılacağı ve hesaplama formülleri (PHR → kg maliyeti → m² maliyeti) belgelenmeli.
- **D9. Kod okunabilirliği:** Tüm JS tek satırlık fonksiyonlarda ve 2–3 harfli değişken adlarıyla yazılmış (`rc`, `gQ`, `kU`, `cU2`...). Bakım ve hata ayıklama zor. Test yok.

---

## ✅ Uygulanan Düzeltmeler (bu PR)

Tümü `index.html` içinde. Hesaplama sonuçları değişmedi: varsayılan değerlerle 1 kg = ₺41.24, satış = $3.48/m². Bu, önceki çıktıyla birebir aynı.

| # | Düzeltme | Tür |
|---|----------|-----|
| 1 | Teklifte **Müşteri** alanı `innerHTML`'e ham olarak ekleniyordu, HTML/JS enjeksiyonuna (XSS) açıktı. Yeni `esc()` fonksiyonuyla kaçırılıyor. Hammadde adları da tabloya kaçırılarak yazılıyor. | Güvenlik |
| 2 | CSV dışa aktarımında hücre `= + - @` ile başlıyorsa önüne `'` ekleniyor (CSV/Excel formül enjeksiyonu önlemi). | Güvenlik |
| 3 | Tüm PHR değerleri 0 yapılınca `0/0` sonucu tüm ekranda `NaN` görünüyordu. Artık 0 kabul ediliyor. Grafikte toplam 0 iken çizim atlanıyor. | Hata |
| 4 | `gQ()` (Teklif) ve `eX()` (Excel) boş/geçersiz alanlarda `NaN` üretiyordu. `rc()` ile aynı varsayılanlar (`\|\|1`, `\|\|0`) eklendi. | Hata |
| 5 | CSV'ye UTF-8 BOM eklendi (Excel'de karakterlerin doğru açılması için). İndirme linki DOM'a eklenip kaldırılıyor (Firefox uyumluluğu). Oluşturulan Blob URL serbest bırakılıyor (bellek sızıntısı). | Hata |
| 6 | Tabloda üretilen `onchange=uM(0,'p',...)` gibi tırnaksız HTML nitelikleri tırnak içine alındı (geçersiz HTML). | Hata |
| 7 | Şifre alanında **Enter** tuşuyla giriş yapılabiliyor. | Küçük iyileştirme |

> Not: Bu düzeltmeler K1/K2'deki temel güvenlik sorununu **çözmez**. Açık şifre ve istemci taraflı giriş, sunucu tarafı bir çözüm gelene kadar risk olmaya devam eder.
