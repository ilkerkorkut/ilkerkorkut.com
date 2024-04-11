---
title: Linux'ta Go Kullanarak Binary İsimi ile PID'lerin Etkin Bir Şekilde Bulunması
author: ilker
date: 2024-03-06
tags:
  - os
  - linux
  - go
  - golang
  - turkish
categories:
  - go
keywords:
  - go
  - linux
---

Birçok sistem işlevi için, Go uygulaması üzerinden execute edilen process'leri yönetmek ve bunların İşlem Kimliklerini (PIDs) bulmak önemlidir. Ancak, özellikle isimler uzunsa veya (işlem durumu) `ps` gibi araçları kullanırken doğru şekilde görüntelenemediğinde PID'leri almak zor olabilmektedir.

#### Sorun: Binary İsimleri Kullanarak Process'leri Bulabilmek

Linux sisteminde heterojen bir process grubuyla çalışırken, process'lerin binary isimleri kullanarak tanımlayabilmek önemlidir. Linux sistemleri, process adını belli bir genişlikteki bir process listeleme alanına sığdırmak için otomatik olarak kısaltır.
Birçok sistem yardımcı programı, `ps`, `top` ve `htop` gibi, process adını sıklıkla kısaltır. Bu araçlar, sıkça process adını kısaltarak komut satırı girişlerini process adının genişliği kadar kısaltırlar.

#### Çözüm: Process ID'leri (PID) Almak İçin Fonksiyon Oluşturma

Aşağıda yer alan Go fonksiyonu ile, binary isimlerine göre process'leri çok daha kolay bir şekilde bulabiliriz. Linux filesystem üzerinde `/proc` dizininde process adı ile birlikte PID'leri symbolic link olarak tutar. Aşağıdaki fonksiyon, bu symbolic linkleri okuyarak binary isimlerine göre process'leri bulabiliriz.

```go
const (
	procPath = "/proc"
	exePath  = "exe"
)

func findPID(binaryName string) ([]int, error) {
	processIDs := make([]int, 0)

	procs, err := os.ReadDir(procPath)
	if err != nil {
		return nil, err
	}

	for _, proc := range procs {
		pid, err := strconv.Atoi(proc.Name())
		if err != nil {
			continue
		}

		binPath := filepath.Join(procPath, proc.Name(), exePath)

		link, err := os.Readlink(binPath)
		if err != nil {
			continue
		}

		fileName := filepath.Base(link)

		if strings.EqualFold(fileName, binaryName) {
			processIDs = append(processIDs, pid)
		}
	}

	return processIDs, nil
}
```

1. *`/proc` Dizinini Okuma*: Fonksiyon, etkin process'ler, temsil eden dizinlerin bir listesini elde etmek için `/proc` dosya sistemini okuyarak başlar. Fonksiyon, `/proc` dosya sistemindeki dizinlerin listesini okur ve her bir process'i temsil eden dizin üzerinden iterasyon yapar. `/proc/<PID>/exe` binary dosya yolunu ve `os.Readlink` kullanarak sembolik bağı okur. Daha sonra, sembolik bağdan elde edilen dosya adını istenilen `binaryName` ile büyük harf küçük harf duyarlılıklı karşılaştırıyoruz.
2. *İterasyon ve Eşleştirme*: Her dizin üzerinden geçerek PID'yi ve process'in binary dosya yolunu elde ediyoruz.
3. *Binary İsimleri Karşılaştırma*: Binary dosya yolunun sembolik bağını okuyarak ve dosya adını çıkararak, binary ismi istenen isimle büyük küçük harf duyarlılığıyla karşılaştırıyoruz.
4. *Sonucu Oluşturma*: Eşleşme bulunduğunda, fonksiyon ilgili PID'yi PID listesine ekliyoruz.(Birden fazla aynı isimde process olabilir.)

Uygulama Yaklaşım Faydaları
* Process ID'lerin Kesin Tanımlanması: Process'lerin binary isimlerine doğru şekilde eşlenmesini sağlayarak standart araçların kısıtlamalarını aşar.
* Process İzleme ve Yönetme: Belirli process'leri izlemek veya bunları binary isme dayanarak yönetmek için idealdir.
* İyileştirilmiş Sistem Verimliliği: Belirli process'ler üzerinde hedeflenen eylemleri yaparak sistem verimliliğini artırır.