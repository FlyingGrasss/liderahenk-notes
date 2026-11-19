# Liderahenk — mimari notları

TÜBİTAK BİLGEM'in açık kaynak merkezi yönetim sistemi. Kurumsal ağdaki sınırsız sayıda
istemciyi ve kullanıcıyı tek merkezden yönetmek için tasarlanmış. Bizim panel/agent
fikirleri için okunmaya değer, çünkü "sunucu + ajan + politika" modelini uçtan uca kurmuş.

## Bileşenler

- **Lider** — merkez. İçinde LDAP istemcisi, görev yöneticisi ve XMPP istemcisi var.
  Karaf (OSGi) konteynerinde çalışır; eklentiler bundle olarak eklenir. REST API sunar,
  Eclipse RCP tabanlı **Lider Konsol** bu API'ye bağlanır.
- **Ahenk** — istemci ajanı. Python, süper kullanıcı olarak çalışır. Lider'den gelen
  mesajları ayrıştırır, eklentilere dağıtır, sonucu geri bildirir. Sonuçlar iki tarafta
  da saklanır (kopuklukta tekrar gönderim).
- **XMPP (Ejabberd)** — Lider ↔ Ahenk iletişimi. Komutlar mesaj olarak gider; büyük
  dosyalar için ayrı dosya sunucusu (SSH) kullanılır. Neden XMPP: açık standart,
  gerçek zamanlı, ölçeklenebilir, çift yönlü.
- **LDAP (OpenLDAP/AD)** — kullanıcı, makine ve grup bilgisinin tek kaynağı.
  Politikalar kullanıcı gruplarına bağlanır; oturum açılışında Lider'e sorularak uygulanır.
- **MariaDB** — Lider'in kendi verisi (görev geçmişi, eklenti DB'si, loglar).

## Kavram modeli

| Kavram | Anlamı |
|---|---|
| Görev (Task) | Tek seferlik iş: paket kur/kaldır, script çalıştır, servis yönet, dosya kopyala, USB kilitle. Tek istemciye ya da gruba. Zamanlanabilir. |
| Profil | Bir eklentinin yapılandırma kümesi (tarayıcı profili, USB profili, mesaj profili). Tek başına çalışmaz. |
| Politika | Bir veya daha fazla profilin birleşimi. Kullanıcı grubuna atanır, oturum açılışında uygulanır. |
| Rapor | İstemci sorgusu + görev geçmişi; sonuçtan istemci grubu üretilebilir. |
| Eklenti | 30+ eklenti; her biri kendi sorgu/rapor ekranıyla gelir. Yeni eklenti Maven archetype ile yazılır. |

## Alınacak dersler

1. **Tek veri kaynağı**: kullanıcı/makine/grup LDAP'te; politika ataması da orada.
   Uygulama kendi kopyasını tutmuyor.
2. **Komut/olay ayrımı**: görev = anlık komut, politika = bildirimsel durum. İkisini
   karıştırmamak, "neden bu makine böyle" sorusunu cevaplanabilir kılıyor.
3. **Kopukluk toleransı**: ajan sonucu saklar, bağlantı gelince tekrar gönderir;
   sunucu da görev geçmişini tutar. İdempotent görev kimliği şart.
4. **Eklenti sınırı**: çekirdek (LDAP, görev, XMPP, log) sabit; her yetenek eklenti.
   Bizim Kavram CMS'teki "collection + typed entries" yaklaşımının kurumsal karşılığı.
5. **Arayüz ayrı süreç**: konsol REST üzerinden konuşur; çekirdek UI'a bağımlı değil.
6. **Büyük dosya ayrı kanal**: mesajlaşma kanalı küçük komutlar için; dosya transferi
   ayrı sunucu. WebSocket'e dosya sıkıştırmamak gerektiğinin kanıtı.

## Bizim tarafa uyarlama

- **Kavram CMS** zaten 1'i (tek veri kaynağı) ve 4'ü (typed collections) karşılıyor.
- **Meydan/ajan fikri** için 2 ve 3 geçerli: komutlar (task) ile durum (desired state)
  ayrılmalı; ajan komut sonucunu kalıcı kuyrukta tutmalı.
- **NFC/okul ekranı** gibi cihaz filoları için XMPP yerine MQTT/WS + kalıcı kuyruk
  (SQLite outbox) yeterli; Lider'in LDAP bağımlılığını kopyalamaya gerek yok.
