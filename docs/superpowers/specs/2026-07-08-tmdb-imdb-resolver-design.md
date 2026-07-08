# TMDB Üzerinden Otomatik IMDb ID Çözümü — Tasarım

**Tarih:** 2026-07-08
**Durum:** Onaylandı (kullanıcı tarafından)
**Repo:** kişisel fork of [opensubtitles/vlsub-opensubtitles-com](https://github.com/opensubtitles/vlsub-opensubtitles-com) (v1.2.9) → [Greonzhin/vlsub-opensubtitles-com](https://github.com/Greonzhin/vlsub-opensubtitles-com) — **PR hedefi yok, kişisel kullanım.**

## Amaç

Kurulu VLSub OpenSubtitles.com extension'ı, dosya adından GuessIt API ile title/season/episode/year çıkarıyor, ama bunu OpenSubtitles'ın kendi isim-arama algoritmasına gönderiyor. Kısa/jenerik başlıklarda (ör. "From") bu yanlış eşleşmeye yol açıyor (gerçek örnek: "From" S02E05 araması "Arifureta: From Commonplace to World's Strongest" döndürdü). Çözüm: GuessIt'in çıkardığı title/season/episode'u TMDB üzerinden kesin bir IMDb ID'ye çevirip, extension'ın zaten var olan "IMDb ID doluysa sadece ID ile ara" davranışını tetiklemek — isim-arama adımını tamamen atlayarak.

## Kanıt (bu oturumda doğrulandı)

- **Sorun gerçek ve tekrarlayan:** `%APPDATA%\vlc\lua\extensions\userdata\vlsub.com\` klasöründe "From" dizisinin 5 farklı bölümü (S01E09, S01E10, S02E01, S02E02, S02E04) için GuessIt cache dosyası bulundu — kullanıcı bu sorunla defalarca karşılaşmış.
- **Enjeksiyon noktası:** `vlsubcom.lua:3538-3639` (`getMovieInfo()`), GuessIt verisini `openSub.movie.{title,year,seasonNumber,episodeNumber}`'a yazıyor. `openSub.movie.imdbId` ayrı bir alan, `vlsubcom.lua:4017-4029`'daki mantık bu alan doluysa isim-aramayı tamamen atlayıp sadece ID ile arıyor (`"If IMDB ID is provided, use IMDB search exclusively"`).
- **Çözüm zinciri canlı test edildi:** `GET /search/tv?query=From&api_key=X` → tmdb_id 124364 → `GET /tv/124364/season/2/episode/5/external_ids?api_key=X` → `{"imdb_id":"tt19815566",...}` — doğru sonuç (bağımsız olarak TMDB find-by-id ile de doğrulanmıştı: bu ID "Lullaby" S02E05).

## Mimari

`vlsubcom.lua`'ya (kişisel fork, 302KB tek dosya, mevcut yapı korunuyor) ~40-60 satır eklenir: yeni bir TMDB API key config alanı + `resolveImdbIdViaTMDB()` fonksiyonu + `getMovieInfo()`'nun sonuna tek bir çağrı. Arama-yürütme kodunun kendisi (SearchSubtitles vb.) hiç değişmez — sadece `openSub.movie.imdbId` alanını, kullanıcı elle girmiş gibi, otomatik doldurur.

## Bileşenler

### 1. TMDB API key alanı
Mevcut Config dialoguna yeni bir text-input eklenir (mevcut ayarların yanına). Ayrı bir dosyada (`vlsub-tmdb-key.conf`, OpenSubtitles'ın kendi config dosyalarından bağımsız) kalıcı tutulur. **Boşsa özellik tamamen devre dışı** — mevcut davranış hiç değişmez.

### 2. `resolveImdbIdViaTMDB(title, year, season, episode)`
- `season` ve `episode` doluysa (dizi): `GET /search/tv?query=<title>&api_key=<key>` → `results[1].id` → `GET /tv/<id>/season/<season>/episode/<episode>/external_ids?api_key=<key>` → `imdb_id`.
- `season`/`episode` boş ama `title`+`year` doluysa (film): `GET /search/movie?query=<title>&year=<year>&api_key=<key>` → `results[1].id` → `GET /movie/<id>/external_ids?api_key=<key>` → `imdb_id`.
- Herhangi bir adımda ağ hatası / boş sonuç / `imdb_id` alanı yok → fonksiyon `nil` döner, hiçbir şey değiştirmez.
- JSON çözümleme: `vlsubcom.lua:609` (`local json_ok, json_module = pcall(require, "dkjson")`) ile zaten yüklenen global `json` nesnesi yeniden kullanılır (`json.decode(body, 1, true)` — kod tabanında `vlsubcom.lua:913,1402,1526,1628` gibi yerlerde aynı çağrı kalıbı defalarca kullanılıyor). Yeni bağımlılık eklenmez.

### 3. Enjeksiyon — `getMovieInfo()`'nun sonu
GuessIt/fallback-parsing bloğu bittikten hemen sonra: TMDB key doluysa VE `openSub.movie.imdbId` henüz boşsa (yani kullanıcı elle bir şey girmemişse) `resolveImdbIdViaTMDB(...)` çağrılır. Sonuç `nil` değilse `openSub.movie.imdbId`'ye yazılır.

## Veri Akışı

Video açılır → GuessIt çalışır → `getMovieInfo()` title/season/episode/year'ı doldurur → (yeni) TMDB key varsa ve imdbId boşsa `resolveImdbIdViaTMDB` çağrılır → başarılıysa `imdbId` dolar → mevcut auto-search mantığı `imdbId` dolu olduğu için ID-bazlı arar (isim-arama hiç devreye girmez, "Arifureta" karışıklığı oluşmaz) → başarısızsa (key yok / TMDB'de yok / ağ hatası) mevcut isim-arama davranışı aynen çalışır, hiçbir şey bozulmaz.

## Hata Yönetimi

Her başarısızlık modu (key yok, ağ hatası, TMDB'de eşleşme yok, `external_ids` boş/imdb_id alanı yok) sessizce bir sonraki adıma/mevcut davranışa düşer. Kullanıcıya hiçbir hata diyaloğu gösterilmez — extension'ın zaten mevcut sessiz-loglama konvansiyonu (`vlc.msg.dbg`) kullanılır.

## Kapsam Dışı

- OpenSubtitles'a PR — kişisel fork, kullanıcı açıkça onayladı.
- Belirsiz TMDB eşleşmesi için etkileşimli netleştirme UI'ı (`results[1]` yeterli kabul edildi, TheIntroDB projesindeki aynı kabul edilmiş sınırlama).
- turkcealtyazi.org veya başka bir altyazı kaynağı entegrasyonu (premise-check ile reddedildi — OpenSubtitles zaten Türkçe altyazı kapsıyor, IMDB ID alanı zaten manuel çözüm sağlıyordu).

## Test

Lua'da otomatik test yok — bu 302KB upstream dosyanın kendisi hiç test içermiyor, eklenen kısım için de framework eklenmeyecek (mevcut konvansiyona uyum). TMDB endpoint zinciri bu oturumda canlı doğrulandı (yukarıya bakın). Gerçek VLC'de manuel doğrulama: "From" S02E05 dosyasıyla arama yapıp Mesajlar penceresinde isim-arama yerine ID-arama tetiklendiğini ve doğru sonucun (Arifureta değil) döndüğünü görmek.
