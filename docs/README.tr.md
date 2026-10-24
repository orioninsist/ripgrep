# ripgrep (`rg`) — Fedora Linux, Bash ve SELinux rehberi

> [!NOTE]
> Bu belge, [orioninsist/ripgrep](https://github.com/orioninsist/ripgrep) deposu için hazırlanmış Türkçe kullanıcı rehberidir. Ripgrep'in resmî upstream projesi [BurntSushi/ripgrep](https://github.com/BurntSushi/ripgrep)'tir. Bu belgeyi okurken kurulum veya ayarları zaten uyguladığınız varsayılmaz.

## İçindekiler

1. [Kapsam ve temel kavramlar](#kapsam-ve-temel-kavramlar)
2. [Fedora'da kurulumu doğrulama](#fedorada-kurulumu-doğrulama)
3. [Tek yapılandırma klasörü](#tek-yapılandırma-klasörü)
4. [Bash bağlantısı](#bash-bağlantısı)
5. [Arama örnekleri](#arama-örnekleri)
6. [Regex ve dosya filtreleri](#regex-ve-dosya-filtreleri)
7. [SELinux ile güvenli kullanım](#selinux-ile-güvenli-kullanım)
8. [Hata ayıklama ve geri alma](#hata-ayıklama-ve-geri-alma)
9. [Sık sorulan sorular](#sık-sorulan-sorular)
10. [Kaynaklar](#kaynaklar)

## Kapsam ve temel kavramlar

- Sistem: **Fedora Linux**, etkin olabilen **SELinux**, kabuk: **Bash**.
- Kullanıcıya ait ripgrep ayarlarının hedef dizini: **`/home/murat/.config/ripgrep`**.
- Komut örnekleri mevcut dosyaları silmez; `rm`, `chmod -R`, `chcon`, `setenforce 0` veya SELinux politikası değiştiren adımlar yoktur.
- `rg`, satır odaklı hızlı metin arama aracıdır. Varsayılan olarak alt dizinlere iner; `.gitignore`, `.ignore`, `.rgignore` kurallarını, gizli dosya filtrelemesini ve ikili dosya sınırlamalarını gözetir.
- `rg` komutunun kurulacağı yürütülebilir dosya sistem paketine aittir; **bunu `~/.config/ripgrep` içine taşımayın**. Bu klasör yalnızca kişisel yapılandırma içindir.
- Bu rehberdeki örnek yollar Murat kullanıcı hesabı içindir. `whoami` ve `printf '%s\n' "$HOME"` çıktıları beklenenden farklıysa önce yolu doğrulayın.

## Fedora'da kurulumu doğrulama

Önce **salt okunur** kontrol:

```bash
command -v rg
rg --version
rpm -q ripgrep
```

`rg` bulunamazsa Fedora'nın resmî deposundan kurulum için **isteğe bağlı** komut:

```bash
sudo dnf install ripgrep
```

Bu son komut paket kurar; yalnızca kurmak istediğinizde çalıştırın. Kuruluysa yeniden kurmaya gerek yok. Kurulumdan sonra `rg --version` ile sürümü kontrol edin. PCRE2 desteği gibi özellikler dağıtım paketinin derlemesine bağlı olabilir.

## Tek yapılandırma klasörü

Hedef düzen:

```text
/home/murat/.config/ripgrep/
├── config          # rg'nin okuduğu seçenekler
└── README.md       # isteğe bağlı kişisel notlar; rg okumaz
```

> [!IMPORTANT]
> Ripgrep kendiliğinden bu dizinde bir `config` aramaz. `RIPGREP_CONFIG_PATH` **bir dizine değil, gerçek bir dosyaya** işaret etmelidir.

İlgili klasörün durumunu önce inceleyin:

```bash
ls -ld /home/murat/.config /home/murat/.config/ripgrep
printenv RIPGREP_CONFIG_PATH
```

Klasör mevcut değilse yalnızca kullanıcının kişisel dizinini oluşturun:

```bash
mkdir -p /home/murat/.config/ripgrep
```

Dosyayı örneğin `nano /home/murat/.config/ripgrep/config` ile açın. **Var olan ayarları silip üzerine yazmayın**; önce okuyun. Yeni/boş bir dosya için önerilen muhafazakâr içerik:

```text
# Her satır tek bir rg argümanıdır.
--line-number
--color=auto
--smart-case
```

Seçeneklerin anlamı:

| Satır | Etki |
| --- | --- |
| `--line-number` | Eşleşmenin satır numarasını gösterir. |
| `--color=auto` | Terminal uygunsa renklendirir; dosya yönlendirmesinde renk dayatmaz. |
| `--smart-case` | Desen tamamen küçük harfliyse büyük/küçük harfi önemsemez; büyük harf varsa önemser. |

Bu örnek **`--hidden`, `--no-ignore`, `--text`, `-uuu` içermez**. Böylece gizli, görmezden gelinen veya ikili dosyaları istemeden taramazsınız. Ayrıca her satır tek bağımsız argümandır; kabuk betiği, JSON veya TOML formatında değildir. Örneğin `--glob=*.rs` tek satır olabilir; `--glob *.rs` tek satırda iki argüman oluşturmaz.

## Bash bağlantısı

Önce yalnızca mevcut terminal oturumunda deneyin:

```bash
export RIPGREP_CONFIG_PATH=/home/murat/.config/ripgrep/config
printf '%s\n' "$RIPGREP_CONFIG_PATH"
rg --version
```

Bu oturumluk ayar, dosya yerindeyse hemen etkilidir. Yeni Bash oturumlarında da etkin olması için `~/.bashrc` dosyanızı **önce kontrol edin**:

```bash
rg -n 'RIPGREP_CONFIG_PATH' /home/murat/.bashrc
```

Bir eşdeğer satır zaten varsa ikinci kez eklemeyin. Yoksa dosyayı editörde açıp **tek satır** ekleyin:

```bash
export RIPGREP_CONFIG_PATH=/home/murat/.config/ripgrep/config
```

Yeni terminal açmak ayarı yüklemeye yeterlidir. Mevcut terminal için yalnızca Bash betiğinin içeriğini kontrol ettikten sonra `source /home/murat/.bashrc` kullanın. Bash başlangıç dosyası `~/.config/ripgrep` dışında kalır: bu, Bash'in standart çalışma biçimi nedeniyle **tek zorunlu bağlantı noktasıdır**; gerçek ripgrep ayarları bütünüyle hedef klasördedir.

Geçerli ayarlarla sorun yaşarsanız tek komutluk yapılandırmasız test:

```bash
RIPGREP_CONFIG_PATH= rg -n 'TODO' .
```

## Arama örnekleri

Tüm komutları kendi proje dizininizde çalıştırın. `.` geçerli dizindir.

| İhtiyaç | Komut | Açıklama |
| --- | --- | --- |
| Temel arama | `rg 'TODO' .` | İçerikte TODO geçen satırlar. |
| Sabit metin | `rg -F 'a.b' .` | Nokta regex jokeri değil, düz nokta olur. |
| Büyük/küçük harf duyarsız | `rg -i 'hata' .` | `HATA` gibi biçimleri de arar. |
| Tam kelime | `rg -w 'main' .` | Kelime sınırına göre arar. |
| Satır numarası | `rg -n 'error' .` | Satır numaralarını gösterir. |
| Dosya ve satır | `rg -n --no-heading 'error' .` | Her sonuçta yol ve satırı görmek için uygun. |
| Yalnızca dosya adları | `rg -l 'TODO' .` | Eşleşme içeren yolları listeler. |
| Eşleşmeyen dosyalar | `rg --files-without-match 'TODO' .` | Okunabilen aday dosyalardan eşleşmeyenler. |
| Kaç eşleşme satırı | `rg -c 'TODO' .` | Dosya başına eşleşen satır sayısı. |
| Önce/sonra bağlamı | `rg -C 2 'panic' .` | İki satır önce/sonra. |
| Birden çok desen | `rg -e 'TODO' -e 'FIXME' .` | Desenlerden herhangi biri. |
| Yalnızca yollar | `rg --files .` | Varsayılan filtrelere göre aranan dosyalar. |
| Gizli dosyaları dahil et | `rg --hidden 'anahtar' .` | Gizli yolları da tarar; ignore kuralları devam eder. |
| Neden yok sayılıyor? | `rg --files --debug .` | Çok ayrıntılı hata ayıklama çıktısı üretebilir. |

Terminalde çıktı çok fazlaysa önce `rg -l` veya `rg --count-matches` kullanarak kapsamı daraltın; gerekirse `rg -m 5 'desen' dosya.txt` ile **dosya başına** eşleşen satır sayısını sınırlayın. `-m` tüm dizin boyunca toplam beş sonuç anlamına gelmez.

### Dosya türleri ve glob

```bash
rg -t rust 'fn main' .
rg -t py 'def ' .
rg -g '*.md' 'SELinux' .
rg -g '!target/**' 'TODO' .
rg -g '!node_modules/**' 'TODO' .
rg --type-list
```

`-t` tür adları `rg --type-list` çıktısında görülebilir; tür adlarının sürüme göre değişebileceğini unutmayın. `-g` glob desenlerini tek tırnakla koruyun ki kabuk önce genişletmesin.

### Regex kısa başvuru

| Desen | Anlam |
| --- | --- |
| `^ERROR` | Satır başında ERROR |
| `done$` | Satır sonunda done |
| `foo|bar` | foo veya bar |
| `[0-9]+` | Bir ya da daha fazla ASCII rakam |
| `\bcat\b` | Kelime sınırlarında cat |
| `(?i)fedora` | Desenin içinde duyarsız eşleme |
| `\p{L}+` | Unicode harflerden oluşan bölüm |

Varsayılan regex motoru **look-around** ve **backreference** desteklemez. Sisteminizde PCRE2 varsa `rg -P '(?<=ID:)\d+' dosya.txt` türü gelişmiş ifadeler kullanabilirsiniz. `rg -P` hata verirse özellik derlemede bulunmayabilir; önce `rg --version` ve `rg --help` inceleyin.

### Standart girdi, sıfır eşleşme ve çıkış kodu

```bash
printf '%s\n' 'Fedora Linux' 'Bash terminal' | rg -i 'fedora'
rg -q 'TODO' README.md
printf 'durum=%s\n' "$?"
```

`rg` çıkış kodu: **0** en az bir eşleşme, **1** eşleşme yok, **2** hata. Bu nedenle `rg` komutundan hemen sonraki `$?` değerini yorumlayın. `rg -q` sessizdir; komut sonunda eşleşme metni basmaz.

## SELinux ile güvenli kullanım

**SELinux'u kapatmayın.** Ripgrep normal kullanıcı hesabıyla, o hesabın okuyabildiği dosyaları arar. SELinux zorunlu erişim kontrolü veya klasik Linux dosya izinleri bir yolu yasaklıyorsa ripgrep bu engeli atlamamalıdır.

Salt okunur tanılama:

```bash
getenforce
id -Z
ls -Zd /home/murat/.config/ripgrep
ls -Z /home/murat/.config/ripgrep
```

- `getenforce` çıktısı `Enforcing`, `Permissive` veya `Disabled` olabilir. Etkin/enforcing durum normaldir.
- `id -Z` ve `ls -Z` güvenlik bağlamını gösterir; bu komutlar bağlamı değiştirmez.
- `Permission denied` yalnızca SELinux nedeniyle olmaz: dosya sahibini, izinleri ve ilgili yolun erişilebilirliğini ayrıca kontrol edin.
- Gerekli olmayan `sudo rg` kullanmayın. Yönetici yetkisiyle tarama, gereksiz hassas dosyaları sonuçlara dahil edebilir.
- `setenforce 0`, `chcon`, `semanage fcontext`, `restorecon -R`, `chmod -R 777` gibi müdahaleler bu kurulumun parçası **değildir**. Gerçek bir AVC reddi görülürse, bağlama uygun çözümü ayrı değerlendirin.
- Arama sonuçları sır veya belirteç içerebilir. Paylaşmadan önce gözden geçirin; `--hidden` ve `-uuu` kullanırken özellikle dikkat edin.

## Hata ayıklama ve geri alma

| Belirti | Salt okunur kontrol / çözüm |
| --- | --- |
| `rg: command not found` | `command -v rg`, `rpm -q ripgrep` |
| Ayarlar etkisiz | `printenv RIPGREP_CONFIG_PATH`; değer **dosyaya** mı işaret ediyor? |
| Config hatası | `rg --help`, `rg --version`; config içindeki her satır tek argüman mı? |
| Beklenen dosya bulunmuyor | `rg --files .`, `rg --files --hidden .`, gerekiyorsa `rg --files --debug .` |
| Eşleşme yok | `rg -F` ile düz metni ve `rg -i` ile harf duyarlılığını kontrol edin. |
| `Permission denied` | `ls -ld` ve SELinux bağlamlarıyla erişimi teşhis edin; güvenliği kapatmayın. |
| Renkler yönlendirmede sorun | `--color=auto` kullanın; gerekirse tek komutta `--color=never`. |

Geri alma için **dosya silmek gerekmez**. Bu terminalde:

```bash
unset RIPGREP_CONFIG_PATH
```

Kalıcı kullanım iptali için `/home/murat/.bashrc` içindeki **yalnızca sizin eklediğiniz** `export RIPGREP_CONFIG_PATH=...` satırını editörde kaldırın; sonraki Bash oturumunda etkin olmayacaktır. Mevcut konfigürasyon dosyası yerinde kalır.

## Sık sorulan sorular

**Bütün ayarlar `/home/murat/.config/ripgrep` içinde olabilir mi?**  
Evet, ripgrep'e ait seçeneklerin hepsi `config` içinde bulunabilir. Ancak Bash'e bu dosyanın yerini göstermek için ya mevcut terminalde `export` gerekir ya da genellikle `~/.bashrc` içine tek bir bağlantı satırı yazılır.

**`rg` dosyaları değiştirir mi?**  
Yukarıdaki arama komutları dosyaların içeriğini değiştirmez. `--replace` seçeneği bile kaynak dosyayı yerinde düzenlemez; eşleşme çıktısının biçimini değiştirir.

**`rg -uuu` neden varsayılan değil?**  
Gizli, yok sayılan ve ikili adayları daha geniş kapsar; aramayı yavaşlatabilir ve hassas içerikleri açığa çıkarabilir. Yalnızca bilinçli, dar kapsamlı ihtiyaçlarda kullanın.

**Config dosyasında shell komutu çalışır mı?**  
Hayır. Satırlar ripgrep argümanlarıdır. `export`, `alias`, komut ikamesi veya kabuk yönlendirmeleri config içinde çalıştırılmaz.

**Fedora güncellemesini nasıl kontrol ederim?**  
`rpm -q ripgrep` yüklü sürümü gösterir. Güncelleme yönetimi Fedora paket yöneticisinin işidir; rehber otomatik güncelleme veya sistem değişikliği yapmaz.

**Bu belge man sayfasının tüm seçeneklerini kapsıyor mu?**  
Hayır. Kullanıcının güvenli başlangıcı için seçilmiş açıklamalı örnekler içerir. Tüm bayrakların ve ayrıntıların kaynakları aşağıdadır.

## Kaynaklar

1. [ripgrep upstream README](https://github.com/BurntSushi/ripgrep/blob/master/README.md) — kurulum, özellikler, platform desteği.
2. [Resmî User Guide](https://github.com/BurntSushi/ripgrep/blob/master/GUIDE.md) — filtreleme, dosya türleri, config.
3. [Resmî FAQ](https://github.com/BurntSushi/ripgrep/blob/master/FAQ.md) — davranış ve yaygın sorular.
4. [Resmî man sayfası kaynağı](https://github.com/BurntSushi/ripgrep/blob/master/crates/core/flags/defs.rs) ve bu çalışmaya eklenen `rg.1.md` — ayrıntılı seçenekler.
5. [Rust regex sözdizimi](https://docs.rs/regex/latest/regex/#syntax) — varsayılan düzenli ifadeler.
6. [Fedora SELinux User Guide](https://docs.fedoraproject.org/en-US/quick-docs/selinux-getting-started/) — SELinux temelleri.
7. [Bu depo](https://github.com/orioninsist/ripgrep) — dokümantasyonun uygulandığı depo.

> **Not:** Kaynaklara ve kullanıcı tarafından sağlanan `rg.1.md` kılavuzuna dayanılarak 24 Ekim 2026 tarihinde düzenlenmiştir. Komutların sonuçları Fedora sürümüne, kurulu ripgrep paketine, dosya izinlerine ve aktif SELinux politikasına bağlıdır.
