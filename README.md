# VPS-OLCUM

Bir VPS'in ilanda yazanı verip vermediğini ölçmek için yapay zekâ ajanına yazılmış bir talimat dosyası: [VPS-OLCUM.md](VPS-OLCUM.md). Program değildir; terminal kullanabilen bir ajan (ör. Claude Code) okur ve adımları sırayla uygular.

**Kullanım:** Bu dosyayı ajana ver, "şu sunucumu ölç" de. Ajan önce sunucunun size ait olup olmadığını sorar, ne kuracağını söyleyip onay alır, testleri tek tek çalıştırır, kurduklarını temizler ve sonunda karne, sonuç tablosu ve skor verir. Tam ölçüm yaklaşık 1 saat, hızlı mod yaklaşık 10 dakika. Ubuntu/Debian için yazıldı.

Testler diskteki verilere dokunmaz (yalnız geçici bir test dosyası), port açmaz (test sitesi yalnız 127.0.0.1), şifrenizi hiçbir yere yazmaz.

**İki rica:** Ölçtüğünüz sunucu sizin olsun. Sonuçlarınızı paylaşırken yöntemi de paylaşın. Tek bir ölçüm, o günün sonucudur.

Lisans: MIT
