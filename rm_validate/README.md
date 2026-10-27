# Validasi Tambahan Internal Transfer (WH → Virtual WH → Produksi)

Panduan langkah demi langkah untuk membangun addon sesuai POC
**"Additional Validation in Process Internal Transfer"** (POC VALIDATE II DARI PRODUKSI).

> Repo `RM-Validate` ini berisi addon `vit_internal_transfer_virtual/` (dibuat di langkah 4).
> Nama folder repo memakai tanda `-` sehingga tidak bisa langsung jadi nama modul Odoo;
> tambahkan folder repo ini ke `addons_path`, dan modulnya ada di subfolder `vit_internal_transfer_virtual/`.
> Dependency `vit_validate_int` dan `vit_custom_stock_barcode` ada di repo Metindo-18, jadi path itu juga harus ada di `addons_path`.

---

## 1. Ringkasan Skema

### Alur sekarang (existing)

```
Draft ──SCAN──► Waiting ──CONFIRM──► Ready ──VALIDATE──► Done
                                                         │
                                  WH  ─────────────────► PDM/Pre
                         (stok WH berkurang, stok PDM/Pre bertambah sekaligus)
```

### Alur usulan (POC)

```
Draft ──SCAN──► Waiting ──CONFIRM──► Ready ──VALIDATE──► Done ──CONFIRM (Produksi)──► Confirmed
                                                         │                            │
                                         WH ───────► Virtual WH ───────────────────► PDM/Pre
                                       (stok WH −, Virtual WH +)        (stok Virtual WH −, PDM/Pre +)
```

| No | Deskripsi                                   | Button   | Status          | Pergerakan stok              | Pelaku    |
|----|---------------------------------------------|----------|-----------------|------------------------------|-----------|
| 1  | Buat transaksi internal transfer            | -        | Draft           | -                            | Gudang    |
| 2  | Scan label (product packing & lot number)   | Confirm  | Waiting → Ready | Barang masuk ke list transfer | Gudang    |
| 3  | Final check barang yang dikirim ke produksi | Validate | Done            | Gudang → **Virtual Gudang**  | Gudang    |
| 4  | Barang diterima oleh Produksi               | Confirm  | **Confirmed**   | **Virtual Gudang** → Produksi | Produksi  |

Langkah 1–3 = **existing** (tidak diubah, hanya tujuan lokasinya yang menjadi Virtual WH).
Langkah 4 + status **Confirmed** + lokasi **Virtual WH** = **additional** (dibuat oleh addon ini).

---

## 2. Keputusan Desain (baca dulu sebelum coding)

1. **Jangan menambah nilai baru ke `stock.picking.state`.**
   `state` di Odoo 18.2 adalah field compute dari state `stock.move`
   ([stock_picking.py:575](../../../../odoo-saas18.2/addons/stock/models/stock_picking.py)).
   Menambah `'received'` ke selection itu akan ditimpa lagi oleh `_compute_state`.
   → Buat field terpisah `receive_state`, lalu field tampilan `display_state` untuk statusbar.

2. **Lokasi tujuan dokumen = Virtual WH; lokasi akhir (PDM/Pre) disimpan di field tersendiri.**
   Lebih sederhana dan aman daripada "membelokkan" `location_dest_id` diam-diam saat validate.
   User tetap memilih tujuan produksi di form (field `final_location_dest_id`).

3. **Langkah Confirm produksi membuat picking kedua (Virtual WH → PDM/Pre) lalu memvalidasinya.**
   Hasilnya ada jejak dokumen dan stock move yang jelas, lot number ikut terbawa persis
   (tidak di-reserve ulang secara FIFO dari Virtual WH).

4. **Virtual WH berupa lokasi `internal` milik company**, di bawah view gudang tapi **bukan anak dari WH/Stock**.
   - `internal` + company → tetap dihitung di valuasi
     (`_should_be_valued` di [stock_account/models/stock_location.py:25](../../../../odoo-saas18.2/addons/stock_account/models/stock_location.py)).
   - Bukan anak WH/Stock → stoknya tidak ikut di-reserve oleh operasi lain yang mengambil dari WH/Stock.

---

## 3. Setting di Odoo (tanpa coding)

Lakukan di database dev dulu, catat semuanya untuk diulang di production.

### 3.1 Aktifkan fitur Inventory
*Inventory → Configuration → Settings*
- [x] **Storage Locations**
- [x] **Multi-Step Routes** (opsional, tapi memudahkan pengaturan operation type)
- [x] **Lots & Serial Numbers** (sudah aktif di Metindo)

### 3.2 Buat lokasi Virtual WH
*Inventory → Configuration → Locations → New*, satu per gudang pengirim:

| Nama lokasi         | Parent Location | Location Type | Company        |
|---------------------|-----------------|---------------|----------------|
| `Virtual RM`        | `WH-RM` (view)  | Internal      | (company Metindo) |
| `Virtual Comp`      | `WH-COMP` (view)| Internal      | (company Metindo) |
| `Virtual Proc`      | `WH-PROC` (view)| Internal      | (company Metindo) |

> Sesuaikan dengan struktur gudang aktual (Dept Head Gudang RM / Comp / Proc di lembar approval).
> Kalau hanya satu gudang, cukup satu lokasi `WH/Virtual`.
> Jangan centang *Is a Scrap Location* / *Is a Return Location*.

### 3.3 Atur Operation Type pengiriman ke produksi
*Inventory → Configuration → Operation Types* → buka operation type internal transfer ke produksi
(mis. `WH: Kirim ke Produksi`):
- **Default Source Location**: WH/Stock (gudang terkait)
- **Default Destination Location**: lokasi **Virtual** dari langkah 3.2
- (field dari addon) **Gunakan Konfirmasi Produksi**: ✔
- (field dari addon) **Lokasi Produksi Default**: PDM/Pre
- (field dari addon) **Operation Type Penerimaan**: operation type pada langkah 3.4

### 3.4 Buat Operation Type penerimaan produksi
*Operation Types → New*:
- **Name**: `Terima di Produksi`
- **Type of Operation**: Internal Transfer
- **Sequence Prefix**: mis. `PRD/IN/`
- **Default Source**: lokasi Virtual · **Default Destination**: PDM/Pre
- **Show Detailed Operations**: ✔ (supaya lot terlihat)

### 3.5 User & hak akses
Setelah addon terpasang, akan ada group **"Internal Transfer / Penerima Produksi"**:
*Settings → Users* → tambahkan group ini ke:
- Admin Produksi Plant 1 (Prepared)
- Dept Head Prod P1 / P2 / P3 (Irphan Cahyadi, Tulus / Eko P, Dwi Artanto)

User gudang **tidak** diberi group ini, supaya pengirim tidak bisa mengonfirmasi penerimaannya sendiri.

> Lembar tanda tangan di halaman 4 PDF adalah persetujuan POC, bukan approval berjenjang di sistem.
> Kalau nanti dibutuhkan approval per dokumen, ikuti pola approver dari Settings seperti `vit_receipt_to_pay`.

---

## 4. Membuat Addon — Step by Step

### 4.1 Struktur folder

```
RM-Validate/                           # repo ini
├── README.md                          # dokumen ini
└── vit_internal_transfer_virtual/
    ├── __init__.py
    ├── __manifest__.py
    ├── model/
    │   ├── __init__.py
    │   ├── stock_picking_type.py
    │   └── stock_picking.py
    ├── security/
    │   ├── security.xml
    │   └── ir.model.access.csv        # hanya bila ada model baru (wizard dsb.)
    └── view/
        ├── stock_picking_type.xml
        └── stock_picking.xml
```

(Pakai `model/` dan `view/` mengikuti konvensi addon `vit_*` di repo Metindo-18.)

### 4.2 `__manifest__.py`

```python
{
    'name': "Internal Transfer via Virtual WH",
    'summary': "Konfirmasi penerimaan produksi setelah internal transfer divalidasi gudang",
    'author': "Iqbalasaleh [iqbalasaleh.com]",
    'website': "http://www.vitraining.com",
    'category': 'Inventory',
    'version': '1.0',
    'depends': ['stock', 'vit_validate_int', 'vit_custom_stock_barcode'],
    'data': [
        'security/security.xml',
        'view/stock_picking_type.xml',
        'view/stock_picking.xml',
    ],
    'license': 'LGPL-3',
}
```

`vit_validate_int` dijadikan dependency karena addon itu memblokir pembuat picking internal
untuk memvalidasi picking-nya sendiri — lihat langkah 4.5.

### 4.3 Group keamanan — `security/security.xml`

```xml
<odoo>
    <record id="group_production_receiver" model="res.groups">
        <field name="name">Internal Transfer / Penerima Produksi</field>
        <field name="category_id" ref="base.module_category_inventory_inventory"/>
        <field name="implied_ids" eval="[(4, ref('stock.group_stock_user'))]"/>
    </record>
</odoo>
```

### 4.4 Field di Operation Type — `model/stock_picking_type.py`

```python
class StockPickingType(models.Model):
    _inherit = "stock.picking.type"

    use_production_confirm = fields.Boolean("Gunakan Konfirmasi Produksi")
    production_location_id = fields.Many2one("stock.location", "Lokasi Produksi Default",
                                             domain="[('usage', '=', 'internal')]")
    receive_picking_type_id = fields.Many2one("stock.picking.type", "Operation Type Penerimaan",
                                              domain="[('code', '=', 'internal')]")
```

Tambahkan ketiganya ke form operation type (`stock.view_picking_type_form`), tampilkan dua field terakhir
hanya bila `use_production_confirm` dicentang, dan beri constraint: kalau dicentang, keduanya wajib diisi.

### 4.5 Logika di Picking — `model/stock_picking.py`

**Field baru:**

| Field                    | Tipe                                   | Keterangan |
|--------------------------|----------------------------------------|------------|
| `use_production_confirm` | Boolean, related ke picking type       | Penanda alur ini aktif |
| `final_location_dest_id` | Many2one `stock.location`              | Tujuan akhir (PDM/Pre), default dari picking type |
| `receive_state`          | Selection `to_receive` / `received`, `copy=False` | Status tahap produksi |
| `receive_picking_id`     | Many2one `stock.picking`, `copy=False` | Picking kedua (Virtual → PDM/Pre) |
| `received_by_id`         | Many2one `res.users`, `copy=False`     | Siapa yang klik Confirm |
| `received_date`          | Datetime, `copy=False`                 | Kapan diterima |
| `display_state`          | Selection compute                      | `state` + `received` untuk statusbar |

**Langkah logika:**

1. **Setelah Validate (langkah 3)** — override `_action_done` (bukan `button_validate`, karena
   `button_validate` bisa berhenti di wizard backorder/immediate transfer):
   ```python
   def _action_done(self):
       res = super()._action_done()
       self.filtered(lambda p: p.use_production_confirm and p.state == 'done') \
           .write({'receive_state': 'to_receive'})
       return res
   ```
   Pada titik ini stok sudah pindah WH → Virtual WH karena `location_dest_id` picking = Virtual.

2. **Tombol Confirm produksi (langkah 4)** — method `action_production_confirm`:
   - Cek user punya group `group_production_receiver`, `state == 'done'`, `receive_state == 'to_receive'`.
   - Buat picking baru dengan `picking_type_id = receive_picking_type_id`,
     `location_id = self.location_dest_id` (Virtual), `location_dest_id = final_location_dest_id`,
     `origin = self.name`.
   - Untuk setiap `move_ids` yang done, buat move dengan produk/qty/UoM yang sama.
   - `action_confirm()`, lalu **ganti move line hasil reservasi otomatis** dengan move line yang
     menyalin `lot_id`, `package_id`, `quantity` dari move line picking asal, set `picked=True`
     — supaya lot yang diterima persis lot yang dikirim.
   - Validasi dengan context:
     ```python
     receive.with_context(skip_internal_validation=True, skip_backorder=True).button_validate()
     ```
     `skip_internal_validation` **wajib**: picking kedua dibuat oleh user produksi itu sendiri,
     sehingga tanpa context ini `vit_validate_int` akan menolak dengan
     *"You are not allowed to validate this picking"*.
   - Tulis `receive_state='received'`, `receive_picking_id`, `received_by_id`, `received_date`,
     dan `message_post` di chatter.

3. **`display_state`**:
   ```python
   @api.depends('state', 'receive_state')
   def _compute_display_state(self):
       for p in self:
           p.display_state = 'received' if p.receive_state == 'received' else p.state
   ```
   Selection-nya = selection `state` standar + `('received', 'Confirmed')`.

4. **Pengaman tambahan:**
   - Tolak `action_production_confirm` dua kali (cek `receive_state`).
   - Tolak *Return* pada picking yang sudah `received` (return harus dari picking kedua),
     atau minimal beri peringatan.
   - Picking yang di-unlock & di-edit setelah done: pastikan tidak bisa mengubah qty kalau sudah `received`.

### 4.6 View — `view/stock_picking.xml`

Inherit `stock.view_picking_form`:
- Tombol **Confirm Penerimaan** di header:
  `invisible="not use_production_confirm or state != 'done' or receive_state != 'to_receive'"`,
  `groups="vit_internal_transfer_virtual.group_production_receiver"`, `class="btn-primary"`.
- Statusbar: sembunyikan `state` standar bila `use_production_confirm`, tampilkan
  `display_state` dengan `widget="statusbar"` dan `statusbar_visible="draft,confirmed,assigned,done,received"`.
- Field `final_location_dest_id` di samping *Destination Location* (readonly setelah done).
- Smart button / field ke `receive_picking_id`, info `received_by_id` & `received_date`.

Inherit tree & search view:
- Kolom `receive_state` (badge).
- Filter **"Menunggu Diterima Produksi"**: `[('receive_state', '=', 'to_receive')]` — ini daftar kerja Admin Produksi.

### 4.7 Barcode app (opsional, tahap 2)
Langkah 1–3 dilakukan di app Barcode (dikustom di `vit_custom_stock_barcode`).
Langkah 4 cukup dari form backend dulu. Kalau produksi ingin konfirmasi dengan scan,
tambahkan tombol di barcode view sebagai pengembangan berikutnya.

---

## 5. Install & Uji

### 5.1 Install
```bash
./odoo-bin -c <config> -d <db_dev> -i vit_internal_transfer_virtual --stop-after-init
```
Lalu lakukan setting bagian 3.2–3.5.

### 5.2 Skenario uji

| # | Skenario | Hasil yang diharapkan |
|---|----------|-----------------------|
| 1 | Gudang buat transfer, scan 2 lot, Confirm | Status Ready, lot masuk list |
| 2 | Gudang Validate | Status Done, `receive_state` = Menunggu Diterima; stok WH −, Virtual +; stok PDM/Pre belum berubah |
| 3 | Cek *Inventory → Reporting → Stock* lokasi Virtual | Lot yang dikirim ada di Virtual |
| 4 | User gudang (tanpa group) buka dokumen | Tombol Confirm Penerimaan tidak muncul |
| 5 | Admin Produksi klik Confirm Penerimaan | Statusbar **Confirmed**; picking `PRD/IN/...` done; stok Virtual −, PDM/Pre +; lot sama persis |
| 6 | Klik Confirm lagi (via RPC / double click) | Ditolak |
| 7 | Validate sebagian → backorder | Hanya picking yang done yang menunggu diterima; backorder tetap Ready |
| 8 | Operation type tanpa centang Konfirmasi Produksi | Alur lama berjalan seperti biasa |
| 9 | Valuasi (bila produk AVCO/FIFO otomatis) | Tidak ada jurnal valuasi saat WH → Virtual → PDM/Pre (semua internal) |

---

## 6. Hal yang perlu dikonfirmasi ke user/Produksi

- Apakah Produksi boleh menerima **sebagian** (qty kurang/rusak)? Kalau ya, perlu wizard selisih —
  POC saat ini mengasumsikan diterima penuh.
- Satu lokasi Virtual per gudang (RM/Comp/Proc) atau satu untuk semua?
- Apakah penerima harus sesuai plant (P1/P2/P3)? Kalau ya, group per plant atau record rule berdasarkan
  `final_location_dest_id`.
- Berapa lama barang boleh tertahan di Virtual WH sebelum perlu reminder/activity?
