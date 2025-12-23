# CTF IDSECCONF 2025 Registration Challenge Writeup


{{< param description >}}

> Hanya 10 tim pertama yang berhasil menyelesaikan registration challenge yang dapat mengikuti offline jeopardy di Makassar.
> 
> Beruntung bagi 3 tim teratas yang mendapatkan potongan subsidi Rp2.000.000

# Web + Misc

## Purbyte

### Deskripsi

{{< admonition type=info title="Click to show the desc" open=false >}}
  Tantangannya disini:
  
  https://reg.hackincelebes.org/
{{< /admonition >}}

### Langkah Penyelesaian

- Diberikan chall web dukun. Langsung saja akses webnya dan ini yg tampil.

![Purbyte-01](/images/ctf-idsecconf2025_web1-01.png)

- Tak ada apa pun ketika halamannya diinspeksi. Namun, ada hint bahwa “**Pastikan Kamu berada dijaringan yang benar!**”.
- Saya langsung berpikir untuk coba request dengan menambahkan HTTP Header `X-Forwarded-For: 127.0.0.1`. Hal itu untuk membuat request seolah-olah dikirim dari localhost. Berikut adalah respons dari request tersebut.

```bash
curl -H "X-Forwarded-For: 127.0.0.1" \
	https://reg.hackincelebes.org/

# Output:
<title>Bukan Kortek</title><style>
body {
  background-image: url("purbyte.jpg");
  background-repeat: no-repeat;
  background-position: center;
  background-size: 40%;
  padding-top: 85vh;
  text-align: center;
}
p {
  font-size: 1.5em;
  color: #333;
  margin: 0;
}
</style><p>Kamu di localhost bro, bukan itu!</p><p>AS45732</p
```

- Hmm… apa itu **AS4573**. Langsung aja saya coba tanyakan ke Claude dan ini responsnya.

```txt
Apa itu ASN?
ASN (Autonomous System Number) adalah nomor identifikasi unik yang diberikan kepada organisasi besar seperti ISP, institusi pemerintah, atau perusahaan yang mengelola blok alamat IP tertentu di internet.

Detail AS45732
Berdasarkan informasi yang tersedia:

Organisasi: Pusat Sistem Informasi dan Teknologi Keuangan (Pusintek)
Website: depkeu.go.id
Negara: Indonesia
Registry: IDNIC
Jumlah IPv4: 1,792 alamat IP
Range IP yang termasuk dalam AS45732 antara lain:
103.12.81.0/24
103.93.188.0/24
103.165.248.0/24
103.196.166.0/24
202.61.126.0/24
202.137.230.0/24
```

- Kemungkinannya, AS45732 itu merupakan range IP address tertentu yang dikelola oleh ISP ataupun institusi terkait. Dari informasi yang diberikan oleh LLM Claude, AS itu terdaftar sebagai domain website “[https://depkeu.go.id](https://depkeu.go.id)”. Namun, kita tidak fokus ke website tersebut karena tidak masuk akal jika kita meretas web pemerintahan. Kita akan fokus pada range IP pada AS45732 tersebut.
- Di sini, saya coba ganti HTTP Header sebelumnya menjadi IP range yang tertera.

```bash
curl -H "X-Forwarded-For: 103.12.81.0" https://reg.hackincelebes.org/

# Output:
<title>Bukan Kortek</title><style>
body {
  background-image: url("purbyte.jpg");
  background-repeat: no-repeat;
  background-position: center;
  background-size: 40%;
  padding-top: 85vh;
  text-align: center;
}
p {
  font-size: 1.5em;
  color: #333;
  margin: 0;
}
</style><p>Kamu dijaringan yang benar!</p><p>flag disini: <code>nc korteks.hackincelebes.org 5001</code></p>
```

- Dan ya it works! Pada respons sebelumnya kita disuruh untuk remote connection dengan `nc korteks.hackincelebes.org 5001`.
- Muncul pertanyaan ke-1 (Q1) yang merupakan tantangan kuis perhitungan pajak. Untuk menjawabnya kita **hanya diberikan 3 detik**, tentunya ini ga masuk akal karena kalaupun bisa sampai berapa pertanyaan kita harus jawab ?
- Setelah beberapa kali percobaan, saya menemukan suatu pola jika hanya terdapat **10 buah pertanyaan yang berulang sampai 50x**.
- Berikut adalah beberapa 10 pola pertanyaan yang mirip-mirip (hanya diganti saja angka-angkanya).
    - Q1. Ada 4 dokumen yang masing-masing bernilai di atas Rp5.000.000. Bea materai per dokumen Rp10,000. Berapa Pajak Total?
    Hint: Pajak = Rp10.000 × Jumlah Dokumen
    - Q2. Sebuah pembangkit menghasilkan 5,000 kg CO2e di atas batas. Tarif pajak karbon Rp30/kg. Berapa pajaknya?
    Hint: Pajak = Rp30 × Jumlah Emisi (kg)
    - Q3. Sebuah perusahaan memperkirakan PPh terutang tahun ini Rp480,000,000. Berapa angsuran PPh 25 per bulan?
    Hint: Pajak = Total PPh ÷ 12 bulan
    - Q4. Toko kecil memiliki omzet bulan ini Rp300,000,000. Tarif PPh Final UMKM 0.5%. Berapa pajak bulan ini?
    Hint: Pajak = 0.5% × Omzet
    - Q5. Seseorang memiliki Penghasilan Kena Pajak (PKP) sebesar Rp4,000,000. Hitung PPh 21 dengan tarif 5%.
    Hint: Pajak = 5% × PKP
    - Q6. Harga sewa ruko per tahun Rp240,000,000. Tarif PPh Pasal 4 ayat (2) untuk sewa 10%. Berapa pajaknya?
    Hint: Pajak = 10% × Nilai Sewa
    - Q7. Nilai barang Rp1,200,000. Tarif PPN 11%. Berapa PPN yang harus dipungut?
    Hint: Pajak = 11% × Harga Barang/Jasa
    - Q8. Harga barang mewah Rp1,000,000,000. Tarif PPnBM 125%. Berapa PPnBM?
    Hint: Pajak = Tarif PPnBM × Harga Jual
    - Q9. Perusahaan membagikan dividen sebesar Rp80,000,000. Tarif PPh 23 untuk dividen 15%. Berapa pajaknya?
    Hint: Pajak = 15% × Jumlah Dividen
    - Q10. Perusahaan mengimpor barang senilai Rp350,000,000. Tarif PPh 22 impor adalah 2.5%. Berapa pajak?
    Hint: Pajak = 2.5% × Nilai Impor
- Enaknya kita diberikan hint oleh probset, jadi tinggal bikin template jawab pertanyaan sesuai dengan rumus/hint yang diberikan. Yang rumit di sini yaitu parsing data string. Contohnya seperti Rp480,000,00 (nilai Rupiah) menjadi integer value atau 120% (persentase) ke float.
- Gampangnya tinggal kita replace aja karakter seperti [ `,` , `Rp` , `.`  , dan `/kg` ] menjadi string kosong aja `""`. Lalu, untuk presentase ke float tinggal ubah karakter `%` di belakang angka menjadi string kosong `""` dan **dibagi dengan 100.0**.
- Berikut adalah final solver untuk dapat flag:

```python
#!/usr/bin/env python3

from pwn import *
import re

host: str = 'korteks.hackincelebes.org'
port: int = 5001
r = remote(host, port)

####################
# FUNCTION HELPERS #
####################

# 1. Convert "RpX.XXX.XXX" or "RpX,XXX,XXX" or "X,XXX" or "X.XXX.XXX" to valid integer value
def convert_to_int(rp) -> int:
  return int(rp.replace("Rp", "").replace(".","").replace(",","").replace("/kg","").strip())

# 2. Convert percentage string "X%" to float 0.X
def convert_to_percent(percent_str) -> float:
  percent_value = percent_str.replace("%", "").strip()
  return float(percent_value) / 100.0

# 3. Clean question list - remove everything after "Hint:" or "Jawaban:"
def clean_question(quest_list: list) -> list:
  # Find index of "Hint:" or "Jawaban:" if exists, then remove everything after it
  try:
    hint_idx = next(i for i, word in enumerate(quest_list) if word in ["Hint:", "Jawaban:", "Answer:"])
    return quest_list[:hint_idx]
  except StopIteration:
    return quest_list

##############
# MAIN LOGIC #
##############

def get_ans(question: list) -> int:
  """Known pattern questions:
  Q1. Ada 4 dokumen yang masing-masing bernilai di atas Rp5.000.000. Bea materai per dokumen Rp10,000. Berapa Pajak Total?
    Hint: Pajak = Rp10.000 × Jumlah Dokumen
  Q2. Sebuah pembangkit menghasilkan 5,000 kg CO2e di atas batas. Tarif pajak karbon Rp30/kg. Berapa pajaknya?
    Hint: Pajak = Rp30 × Jumlah Emisi (kg)
  Q3. Sebuah perusahaan memperkirakan PPh terutang tahun ini Rp480,000,000. Berapa angsuran PPh 25 per bulan?
    Hint: Pajak = Total PPh ÷ 12 bulan
  Q4. Toko kecil memiliki omzet bulan ini Rp300,000,000. Tarif PPh Final UMKM 0.5%. Berapa pajak bulan ini?
    Hint: Pajak = 0.5% × Omzet
  Q5. Seseorang memiliki Penghasilan Kena Pajak (PKP) sebesar Rp4,000,000. Hitung PPh 21 dengan tarif 5%.
    Hint: Pajak = 5% × PKP
  Q6. Harga sewa ruko per tahun Rp240,000,000. Tarif PPh Pasal 4 ayat (2) untuk sewa 10%. Berapa pajaknya?
    Hint: Pajak = 10% × Nilai Sewa
  Q7. Nilai barang Rp1,200,000. Tarif PPN 11%. Berapa PPN yang harus dipungut?
    Hint: Pajak = 11% × Harga Barang/Jasa
  Q8. Harga barang mewah Rp1,000,000,000. Tarif PPnBM 125%. Berapa PPnBM?
    Hint: Pajak = Tarif PPnBM × Harga Jual
  Q9. Perusahaan membagikan dividen sebesar Rp80,000,000. Tarif PPh 23 untuk dividen 15%. Berapa pajaknya?
    Hint: Pajak = 15% × Jumlah Dividen
  Q10. Perusahaan mengimpor barang senilai Rp350,000,000. Tarif PPh 22 impor adalah 2.5%. Berapa pajak?
    Hint: Pajak = 2.5% × Nilai Impor
  """
  tax_ans: int = 0

  # Q1. Ada X dokumen yang masing-masing bernilai di atas RpX. Bea materai per dokumen Rp10,000. Berapa Pajak Total?
  if question[0] == 'Ada':
    total_docs = convert_to_int(question[1])
    tax_ans = 10000 * total_docs
  # Q2. Sebuah pembangkit menghasilkan X kg CO2e di atas batas. Tarif pajak karbon RpX/kg. Berapa pajaknya?
  elif question[0] == 'Sebuah' and question[1] == 'pembangkit':
    total_emission = convert_to_int(question[3])
    tarif = convert_to_int(question[12])
    tax_ans = tarif * total_emission
  # Q3. Sebuah perusahaan memperkirakan PPh terutang tahun ini RpX. Berapa angsuran PPh 25 per bulan?
  elif question[0] == 'Sebuah' and question[1] == 'perusahaan':
    pph_25 = convert_to_int(question[7])
    tax_ans = pph_25 // 12
  # Q4. Toko kecil memiliki omzet bulan ini RpX. Tarif PPh Final UMKM 0.5%. Berapa pajak bulan ini?
  elif question[0] == 'Toko' and question[1] == 'kecil':
    omzet = convert_to_int(question[6])
    tax_ans = 0.005 * omzet
  # Q5. Seseorang memiliki Penghasilan Kena Pajak (PKP) sebesar RpX. Hitung PPh 21 dengan tarif 5%.
  elif question[0] == 'Seseorang' and question[1] == 'memiliki':
    pkp = convert_to_int(question[7])
    tax_ans = 0.05 * pkp
  # Q6. Harga sewa ruko per tahun RpX. Tarif PPh Pasal 4 ayat (2) untuk sewa 10%. Berapa pajaknya?
  elif question[0] == 'Harga' and question[1] == 'sewa':
    tax_ans = 0.1 * convert_to_int(question[5])
  # Q7. Nilai barang RpX. Tarif PPN 11%. Berapa PPN yang harus dipungut?
  elif question[0] == 'Nilai' and question[1] == 'barang':
    n_barang = convert_to_int(question[2])
    tax_ans = 0.11 * n_barang
  # Q8. Harga barang mewah RpX. Tarif PPnBM 125%. Berapa PPnBM?
  elif question[0] == 'Harga' and question[1] == 'barang' and question[2] == 'mewah':
    selling_price = convert_to_int(question[3])
    PPnBM = convert_to_percent(question[6])
    tax_ans = PPnBM * selling_price
  # Q9. Perusahaan membagikan dividen sebesar RpX. Tarif PPh 23 untuk dividen 15%. Berapa pajaknya?
  elif question[0] == 'Perusahaan' and question[1] == 'membagikan':
    dividend = convert_to_int(question[4])
    tax_rate = convert_to_percent(question[10])
    tax_ans = dividend * tax_rate
  # Q10. Perusahaan mengimpor barang senilai RpX. Tarif PPh 22 impor adalah 2.5%. Berapa pajak?
  elif question[0] == 'Perusahaan' and question[1] == 'mengimpor':
    import_value = convert_to_int(question[4])
    tax_ans = 0.025 * import_value
  # Unknown question.
  else:
    log.error(f"Unknown question pattern: {question[:3]}")
    return 0

  return int(tax_ans)

def solve() -> None:
  # Total questions is 50
  for q_num in range(1, 51):
    r.recvuntil(f"Q{q_num}. ".encode())
    raw_quest = r.recvline().decode().split()
    
    quest = clean_question(raw_quest)
    # log.info(f"Raw Question {q_num}: {' '.join(raw_quest[:10])}...")
    
    log.info(f"Question {q_num}: {' '.join(quest)}")
    
    tax_ans = get_ans(quest)
    log.info(f"Answer: {tax_ans}")
    
    r.sendline(str(tax_ans).encode())
    
    # Check response
    response = r.recvline(timeout=2).decode()
    
  flag = re.search(r"flag\{.*\}", r.recvall().decode())
  if flag:
    log.success(f"FLAG --> {flag.group()}")
  else:
    log.error("FLAG NOT FOUND")
  r.close()

if __name__ == "__main__":
  solve()
```

- POC dapat flag:

```bash
# ... output omitted ...
[*] Question 50: Sebuah pembangkit menghasilkan 50,000 kg CO2e di atas batas. Tarif pajak karbon Rp30/kg. Berapa pajaknya?
[*] Answer: 1500000
[+] Receiving all data: Done (78B)
[*] Closed connection to korteks.hackincelebes.org port 5001
[+] FLAG --> flag{_w3lc0me_t0_kota_daeng_pur_}
```

### Flag

{{< admonition type=question title="Click to show the flag" open=false >}}
  `flag{_w3lc0me_t0_kota_daeng_pur_}`
{{< /admonition >}}

