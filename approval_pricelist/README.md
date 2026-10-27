# Approval Pricelist Vendor (Purchasing → Marketing → Finance)

Panduan langkah demi langkah untuk membangun addon sesuai dokumen
**"Proposed Price List Approval"** (Skema Approval Pricelist Vendor Rev 4, 24 Sep 26).

> Folder `approval_pricelist` ini berisi addon `vit_vendor_pricelist_approval/` (dibuat di langkah 4).
> Tambahkan folder ini ke `addons_path`; modulnya ada di subfolder `vit_vendor_pricelist_approval/`.
> Addon ini hanya bergantung pada modul standar (`purchase`, `mail`), tetapi **berinteraksi** dengan
> `vit_change_price_po` dan `vit_receipt_to_pay` di repo Metindo-18 — lihat bagian 2 poin 6.

---

## 1. Ringkasan Skema

### Alur sekarang (existing)

```
Modul Approval (soft approval)                       Modul Purchase Pricelist (no approval)
Membuat Pricelist Approval ─► Marketing Cek ─► FA Approve & Save di server    Upload Pricelist ─► Pricelist Vendor Active
```

Approval dan harga di Odoo **tidak terhubung**: approval dicatat di modul Approvals (bisa di-bypass),
sedangkan harga di-*import* langsung ke *Vendor Pricelists* dan langsung aktif dipakai PO.

### Alur usulan

```
Import Records ─► [Draft] ─SUBMIT─► [Approval 1: Purchasing] ─► [Approval 2: Marketing] ─► [Approval 3: Finance] ─► [Approved]
 (harga tersimpan,                        │ No                        │ No                      │ No              (harga AKTIF,
  TIDAK aktif)                            └────────────── kembali ke Draft (revisi lalu Submit ulang) ──┘             dipakai PO)
```

| No | Tahap | Pelaku | Tugas | Batas waktu |
|----|-------|--------|-------|-------------|
| 0 | Creator | Purchasing: Anggi, Faizal, Septi | Import harga + lampiran, lalu **Submit** | - |
| 1 | Approval 1 | Purchasing: **Rochman** | Cek harga & lampiran | 2 hari |
| 2 | Approval 2 | Marketing: **Ronny, Juster** | Cek perbandingan harga beli vs jual | 3 hari (Marketing + FA) |
| 3 | Approval 3 | FA: **Agung** (cek quotation, perbandingan harga antar vendor & follow up), **Ripai** (cek harga beli vs jual) | Approve final → harga aktif | ⤴ |

**Follower** (dapat notifikasi, tidak approve):
- Marketing: Noval, Putri, Wahyu F — mengunggah lampiran perbandingan harga beli & jual.
- FA: Pak Reza.

**Lampiran wajib:**
1. *Update Price List* — hanya item yang akan diupdate (dasar compare harga oleh Marketing).
2. *Quotation* bertanda tangan kedua belah pihak / nego / pitching (dasar double cek oleh FA).

**Aturan dari dokumen (Noted):**
1. Tahap awal hanya kategori **Raw Material, Component, Coating, Project**. Consumable & GA menyusul
   setelah price list-nya distandarisasi.
2. Di dalam satu departemen, approval muncul **bersamaan** (paralel, tidak berjenjang).
3. Antar departemen **berjenjang**: Approval 1 selesai baru masuk Approval 2, dst.
4. **Hard approval** — tidak bisa di-bypass.
5. Nomor dokumen otomatis, kode diambil dari Product Category: `PRICE/RM/2026/06/00001`.
6. Log note (chatter) dan lampiran tetap ada.

### Simulasi dampak ke PO & Bill (halaman 5 PDF)

| Kondisi saat pricelist baru (Item A 1000 → 1100) di-approve | PO | Bill |
|---|---|---|
| Barang **belum** di-receipt | Harga PO ikut diupdate → 1100 | 1100 |
| Barang **sudah** di-receipt | Harga PO tetap 1000 | Bill mengikuti pricelist terbaru → 1100 |

Catatan tambahan dari dokumen:
- Perubahan harga → buat Pricelist Approval **baru** hanya untuk produk yang harganya berubah.
- Jika invoice sudah di-bill lalu ada harga baru, Purchasing wajib memberi tahu FA invoice mana yang berubah
  (proses manual / cross cek; addon cukup menyediakan daftar bill terdampak, lihat 4.7).

---

## 2. Keputusan Desain (baca dulu sebelum coding)

1. **Jangan pakai modul Approvals (`approval.request`).**
   Itu yang dipakai sekarang dan sifatnya *soft* — di Metindo bahkan ada `vit_approvals_bypass_groups`.
   Buat model sendiri `vendor.pricelist.approval` dengan state machine, sehingga harga hanya bisa aktif
   lewat tombol Approve tahap terakhir.

2. **"Tidak aktif" = tambahkan field `active` ke `product.supplierinfo`.**
   Di Odoo 18.2 `product.supplierinfo` **tidak punya** field `active`
   ([product_supplierinfo.py](../../../../../odoo-saas18.2/addons/product/models/product_supplierinfo.py)).
   Menambahkannya (default `True`) punya efek yang kita mau tanpa override di banyak tempat:
   - Baris lama otomatis terisi `True` saat install → tidak ada data yang hilang.
   - `seller_ids` (One2many) dan semua `search()` otomatis mengabaikan record `active=False`,
     jadi `_select_seller` di PO tidak akan memakai harga yang belum di-approve.
   - Tambahkan juga `approval_state` (draft/to_approve/approved/rejected) dan `pricelist_approval_id`
     untuk pelacakan.

3. **Harga hasil import dipaksa tidak aktif lewat context `import_file`.**
   `base_import` memanggil `model.with_context(import_file=True).load(...)`
   ([base_import.py:1454](../../../../../odoo-saas18.2/addons/base_import/models/base_import.py)).
   Override `create` → bila `import_file` ada, set `active=False, approval_state='draft'`.
   Override `load` → kumpulkan record hasil import ke dokumen approval per **kategori produk**.
   Tombol **Test** di layar import men-*rollback* savepoint, jadi aman membuat dokumen di dalam `load`.

4. **Import yang meng-update record lama harus diblokir.**
   Mockup halaman 7 memakai kolom `id` (External ID `__export__.product_supplierinfo_...`). Kalau kolom itu
   menunjuk record yang sudah aktif, import akan **menulis langsung** ke harga aktif → bypass approval.
   Di `write`, bila `import_file` dan record sudah `approved`, raise `UserError`
   ("Hapus kolom id; harga baru harus dibuat sebagai baris baru"). Template import yang dibagikan ke user
   **tanpa** kolom `id`.

5. **Saat approve final, harga lama untuk vendor+produk yang sama harus ditutup.**
   Urutan seller standar adalah `sequence, min_qty DESC, price, id`
   ([product_product.py:713](../../../../../odoo-saas18.2/addons/product/models/product_product.py)) — yang
   **paling murah** menang, bukan yang paling baru. Jika 1000 (lama) dan 1100 (baru) sama-sama aktif, PO tetap
   mengambil 1000. Maka saat approve: arsipkan (`active=False`) atau isi `date_end` record lama
   (vendor, produk/variant, min_qty, company, UoM sama) = `date_start` baru − 1 hari.
   Ingat juga di DB Metindo hampir semua `date_start/date_end` kosong, jadi mengarsipkan lebih aman.

6. **Kode lain yang membaca `product.supplierinfo`** (cek setelah addon terpasang):
   - `vit_change_price_po/model/purchase_order.py` — `search()` langsung ke supplierinfo; otomatis aman
     karena `active=False` tersaring, tapi urutannya `date_end desc` (lihat catatan data pricelist).
   - `vit_receipt_to_pay/models/stock_move_line.py::_get_vendor_pricelists` — sama, otomatis tersaring.
   - `vit_rmos/model/purchase.py` — periksa apakah ada `with_context(active_test=False)`.

7. **Approver dikonfigurasi di Settings, bukan dari group** (pola yang sama dengan `vit_receipt_to_pay`).
   Wewenang approve = user ada di daftar approver tahap berjalan. Group hanya untuk akses menu/baca.
   Dengan begitu pergantian orang (cuti, mutasi) cukup diubah di Settings.

8. **Paralel di dalam departemen = satu baris `approval.line` per approver.**
   Tahap dianggap selesai bila **semua** approver tahap itu sudah approve (sesuai kolom tanda tangan
   di halaman 8 — tiap orang punya kotak sendiri). Satu reject di tahap mana pun → dokumen kembali Draft.
   (Konfirmasi ke user: "semua" atau "salah satu" — lihat bagian 6.)

---

## 3. Setting di Odoo (tanpa coding)

### 3.1 Kode kategori produk
*Inventory → Configuration → Product Categories* → isi field (dari addon) **Kode Pricelist** dan centang
**Wajib Approval Pricelist**:

| Kategori | Kode Pricelist | Wajib Approval |
|----------|----------------|----------------|
| Raw Material | `RM` | ✔ |
| Component | `COMP` | ✔ |
| Coating | `COAT` | ✔ |
| Project | `PRJ` | ✔ |
| Consumable / GA | - | (belum, menunggu standarisasi) |

> Kode diambil dari kategori produk itu sendiri atau parent terdekat yang punya kode
> (mis. `Raw Material / Plate` mewarisi `RM`). Sesuaikan dengan struktur kategori aktual.

Produk dengan kategori yang **tidak** wajib approval tetap mengikuti alur lama (import langsung aktif).

### 3.2 Approver & follower
*Purchase → Configuration → Settings* → bagian **Vendor Pricelist Approval**:

| Setting | User |
|---------|------|
| Approver Tahap 1 (Purchasing) | Rochman |
| Approver Tahap 2 (Marketing) | Ronny Indrayanto, Juster Siagian |
| Approver Tahap 3 (Finance) | Agung Santoso, Ahmad Ripai |
| Follower | Noval, Putri, Wahyu F, Muhammad Reza |
| Batas waktu Tahap 1 (hari) | 2 |
| Batas waktu Tahap 2+3 (hari) | 3 |

### 3.3 Group
Setelah addon terpasang ada dua group:
- **Vendor Pricelist / User** — creator (Anggi, Faizal, Septi): import, submit, lihat dokumen sendiri.
- **Vendor Pricelist / Manager** — lihat semua dokumen (approver & follower otomatis bisa membuka
  dokumen yang melibatkan mereka lewat record rule).

User di luar group Manager **tidak boleh** membuat/mengubah `product.supplierinfo` aktif secara manual
dari form produk — kalau tidak, approval bisa di-bypass lewat tab *Purchase → Vendors* di form produk.

---

## 4. Membuat Addon — Step by Step

### 4.1 Struktur folder

```
approval_pricelist/                          # folder ini
├── README.md                                # dokumen ini
└── vit_vendor_pricelist_approval/
    ├── __init__.py
    ├── __manifest__.py
    ├── data/
    │   ├── ir_sequence.xml
    │   ├── mail_activity_type.xml
    │   └── ir_cron.xml                      # reminder lewat batas waktu
    ├── model/
    │   ├── __init__.py
    │   ├── product_category.py
    │   ├── product_supplierinfo.py
    │   ├── vendor_pricelist_approval.py
    │   ├── vendor_pricelist_approval_line.py
    │   ├── purchase_order_line.py
    │   └── res_config_settings.py
    ├── security/
    │   ├── security.xml
    │   └── ir.model.access.csv
    └── view/
        ├── product_category.xml
        ├── product_supplierinfo.xml
        ├── vendor_pricelist_approval.xml
        └── res_config_settings.xml
```

### 4.2 `__manifest__.py`

```python
{
    'name': "Vendor Pricelist Approval",
    'summary': "Hard approval Purchasing → Marketing → Finance sebelum harga vendor aktif",
    'author': "Iqbalasaleh [iqbalasaleh.com]",
    'website': "http://www.vitraining.com",
    'category': 'Inventory/Purchase',
    'version': '1.0',
    'depends': ['purchase', 'mail'],
    'data': [
        'security/security.xml',
        'security/ir.model.access.csv',
        'data/ir_sequence.xml',
        'data/mail_activity_type.xml',
        'data/ir_cron.xml',
        'view/product_category.xml',
        'view/product_supplierinfo.xml',
        'view/vendor_pricelist_approval.xml',
        'view/res_config_settings.xml',
    ],
    'license': 'LGPL-3',
}
```

### 4.3 Kategori — `model/product_category.py`

```python
class ProductCategory(models.Model):
    _inherit = "product.category"

    pricelist_code = fields.Char("Kode Pricelist")
    pricelist_approval_required = fields.Boolean("Wajib Approval Pricelist")

    def _get_pricelist_category(self):
        """Kategori terdekat (diri sendiri / parent) yang wajib approval."""
        self.ensure_one()
        categ = self
        while categ and not categ.pricelist_approval_required:
            categ = categ.parent_id
        return categ
```

### 4.4 Harga vendor — `model/product_supplierinfo.py`

**Field baru:**

| Field | Tipe | Keterangan |
|-------|------|------------|
| `active` | Boolean, `default=True` | Harga hanya dipakai PO bila `True` |
| `approval_state` | Selection `draft`/`to_approve`/`approved`/`rejected`, `default='approved'`, `copy=False` | Default `approved` supaya data lama dianggap sudah sah |
| `pricelist_approval_id` | Many2one `vendor.pricelist.approval`, `copy=False`, `index=True` | Dokumen approval asal |
| `replaced_supplierinfo_ids` | Many2many `product.supplierinfo` | Harga lama yang akan ditutup saat approve (diisi otomatis) |

**Logika:**

1. **`create`** — bila `self.env.context.get('import_file')` dan kategori produk wajib approval:
   `active=False`, `approval_state='draft'`. Kategori dihitung dari `product_id` atau `product_tmpl_id`.

2. **`load`** — kelompokkan record baru ke dokumen approval:
   ```python
   def load(self, fields, data):
       res = super().load(fields, data)
       if self.env.context.get('import_file') and res.get('ids'):
           pending = self.browse(res['ids']).filtered(
               lambda s: s.approval_state == 'draft' and not s.pricelist_approval_id)
           self.env['vendor.pricelist.approval']._attach_imported_prices(pending)
       return res
   ```
   `_attach_imported_prices` mengelompokkan per kategori (`_get_pricelist_category`) dan memasukkan ke
   dokumen **Draft milik user yang sama untuk kategori yang sama** bila sudah ada — karena file besar
   di-import per batch (opsi `limit`) sehingga `load` bisa terpanggil beberapa kali untuk satu file.

3. **`write`** — dua pengaman:
   - Bila `import_file` dan ada record `approval_state == 'approved'` → `UserError` (lihat desain poin 4).
   - Record yang `to_approve` tidak boleh diubah harga/vendor/produknya (harga yang dicek approver harus sama
     dengan yang diaktifkan).
   - Field `active` dan `approval_state` hanya boleh diubah dari method approval
     (pakai context internal, mis. `pricelist_approval_write=True`).

4. **`unlink`** — tolak bila record `to_approve`.

### 4.5 Dokumen approval — `model/vendor_pricelist_approval.py`

`_inherit = ['mail.thread', 'mail.activity.mixin']` (log note + lampiran + activity).

| Field | Tipe | Keterangan |
|-------|------|------------|
| `name` | Char, readonly, `default='New'` | `PRICE/RM/2026/06/00001` |
| `category_id` | Many2one `product.category` | Kategori wajib approval |
| `partner_ids` | Many2many `res.partner` compute | Vendor di dokumen (info / filter) |
| `supplierinfo_ids` | One2many `product.supplierinfo` ↔ `pricelist_approval_id` | Harga dari import, `context="{'active_test': False}"` |
| `state` | Selection | `draft`, `approval_1`, `approval_2`, `approval_3`, `approved`, `rejected` |
| `line_ids` | One2many `vendor.pricelist.approval.line` | Satu baris per approver per tahap |
| `can_approve` | Boolean compute | User login adalah approver tahap berjalan yang belum approve |
| `submit_date`, `stage_deadline` | Datetime / Date | Untuk batas waktu 2 hari / 3 hari |
| `user_id` | Many2one `res.users`, default user login | Creator |

`vendor.pricelist.approval.line`: `approval_id`, `stage` (1/2/3), `user_id`, `role` (Approved / Acknowledged),
`state` (`pending`/`approved`/`rejected`), `date`, `note`. Baris ini sekaligus menjadi
"lembar tanda tangan" di form (halaman 8 PDF).

**Penomoran** — di `create`:
```python
code = vals_category.pricelist_code or 'GEN'
seq = self.env['ir.sequence'].next_by_code('vendor.pricelist.approval')   # padding 5
vals['name'] = f"PRICE/{code}/{fields.Date.today():%Y/%m}/{seq}"
```
Satu sequence dipakai bersama semua kategori. Jika user ingin nomor urut **per kategori**, buat satu
`ir.sequence` per kode (`vendor.pricelist.approval.RM`, dst.).

**Method:**

1. **`action_submit`** (creator, dari `draft` atau setelah reject):
   - Validasi: ada `supplierinfo_ids`; **minimal 2 lampiran** (pricelist update + quotation bertanda tangan);
     setiap harga > 0 dan punya `date_start`.
   - Isi `replaced_supplierinfo_ids` tiap harga baru (harga aktif lama vendor+produk+min_qty+UoM yang sama) —
     berguna juga sebagai kolom "Harga Lama" untuk dibandingkan approver.
   - `supplierinfo_ids.approval_state = 'to_approve'`, masuk tahap 1 → `_start_stage(1)`.

2. **`_start_stage(stage)`**:
   - Hapus baris `pending` tahap itu (kalau ada sisa reject sebelumnya), buat baris baru dari Settings.
   - Buat `mail.activity` untuk **setiap** approver tahap itu sekaligus (paralel), `date_deadline` =
     hari ini + 2 hari (tahap 1) / + 3 hari (tahap 2, sisa untuk tahap 3 dihitung dari `stage_deadline`).
   - `message_subscribe` follower dari Settings.

3. **`action_approve`**:
   - Hard check di server (bukan hanya `invisible` di tombol): `state` sedang di tahap N dan
     `env.user` punya baris `pending` di tahap N. Kalau tidak → `UserError`.
   - Tandai baris user itu `approved`, `action_feedback` activity miliknya ("Approval done").
   - Bila semua baris tahap N sudah `approved` → tahap berikutnya, atau `_action_final_approve()` di tahap 3.

4. **`action_reject`** — wizard wajib isi alasan:
   - Baris user = `rejected`, batalkan semua activity yang tersisa, `state='draft'`
     (alur flowchart: No → kembali ke Submit for Approval), `supplierinfo_ids.approval_state='draft'`,
     post alasan di chatter, beri activity ke creator.

5. **`_action_final_approve`** (dipanggil sistem, bukan tombol):
   ```python
   new = self.supplierinfo_ids.with_context(pricelist_approval_write=True)
   new.replaced_supplierinfo_ids.write({'active': False})       # tutup harga lama (desain poin 5)
   new.write({'active': True, 'approval_state': 'approved'})
   self.state = 'approved'
   self._update_open_purchase_lines()                            # 4.7
   ```

6. **Tidak ada tombol "Force Approve"/bypass** untuk siapa pun, termasuk admin — sesuai syarat hard approval.
   Kalau perlu koreksi, reject lalu submit ulang.

### 4.6 Reminder batas waktu — `data/ir_cron.xml`

Cron harian: dokumen yang lewat `stage_deadline` → post pesan ke chatter dan tambah activity ke approver tahap
berjalan. Gunakan tanggal WIB eksplisit (`datetime.now(pytz.UTC).astimezone(pytz.timezone('Asia/Jakarta')).date()`),
**jangan** `fields.Date.context_today` di cron karena user cron ber-timezone default (lihat pola `vit_rmos`).

### 4.7 Dampak ke PO & Bill — `model/purchase_order_line.py`

Sesuai simulasi halaman 5:

1. **PO belum receipt** — `_update_open_purchase_lines` dipanggil saat final approve:
   ```python
   lines = self.env['purchase.order.line'].search([
       ('product_id', 'in', products.ids),
       ('partner_id', 'in', vendors.ids),
       ('order_id.state', 'in', ('draft', 'sent', 'to approve', 'purchase')),
       ('display_type', '=', False),
       ('qty_received', '=', 0),
       ('qty_invoiced', '=', 0),
   ])
   ```
   Hitung ulang harga dengan `product._select_seller(...)` per line (supaya UoM/min_qty dihormati), tulis
   `price_unit`, dan `message_post` di PO berisi harga lama → baru serta nomor dokumen PRICE.
   Perhatikan `display_type` bernilai `False` untuk line produk (tidak ada nilai `'line'`).

2. **PO sudah receipt, Bill mengikuti harga terbaru** — override `_prepare_account_move_line`
   ([purchase_order_line.py:488](../../../../../odoo-saas18.2/addons/purchase/models/purchase_order_line.py)):
   bila ada supplierinfo aktif yang lebih baru dari tanggal PO untuk produk & vendor ini, pakai harganya
   untuk `price_unit` bill. Selisih dengan harga receipt akan masuk *price difference* di valuasi —
   **konfirmasi ke FA** sebelum mengaktifkan bagian ini.

3. **Bill yang sudah terbit** — tidak diubah otomatis. Sediakan menu/filter "Bill terdampak pricelist baru"
   (bill posted dengan product+vendor yang ada di dokumen PRICE dan harga berbeda) untuk cross cek
   Purchasing ↔ FA.

4. Poin 8 dokumen ("saat pembuatan PO harga otomatis ambil dari pricelist untuk RM, Component, Coating")
   sudah dilakukan standar Odoo lewat `_compute_price_unit_and_date_planned_and_name`; addon ini memastikan
   yang diambil hanya harga yang sudah di-approve.

### 4.8 Settings — `model/res_config_settings.py`

Ikuti pola `vit_receipt_to_pay` — simpan sebagai `ir.config_parameter` berisi id user:

```python
pricelist_approver_1_ids = fields.Many2many('res.users', relation='vit_pl_appr1_rel', string="Approver Tahap 1")
pricelist_approver_2_ids = fields.Many2many('res.users', relation='vit_pl_appr2_rel', string="Approver Tahap 2")
pricelist_approver_3_ids = fields.Many2many('res.users', relation='vit_pl_appr3_rel', string="Approver Tahap 3")
pricelist_follower_ids   = fields.Many2many('res.partner', relation='vit_pl_follower_rel', string="Follower")
pricelist_deadline_1     = fields.Integer(default=2, config_parameter='vit_vendor_pricelist_approval.deadline_1')
pricelist_deadline_2     = fields.Integer(default=3, config_parameter='vit_vendor_pricelist_approval.deadline_2')
```
Many2many tidak bisa langsung `config_parameter`; simpan/baca manual di `set_values` / `get_values`
sebagai daftar id (lihat implementasi di `vit_receipt_to_pay`).

### 4.9 View

**`view/product_supplierinfo.xml`** — inherit list & search *Vendor Pricelists* (`product.product_supplierinfo_tree_view`):
- Kolom `approval_state` (badge) dan `pricelist_approval_id`.
- Filter **Menunggu Approval** (`[('active','=',False), ('approval_state','in',('draft','to_approve'))]`)
  dan **Archived**; tanpa filter ini harga hasil import "hilang" dari list karena tidak aktif.

**`view/vendor_pricelist_approval.xml`** — form sesuai mockup halaman 8:
- Header: tombol **Submit** (`state == 'draft'`), **Approve** / **Reject** (`invisible="not can_approve"`),
  statusbar `draft,approval_1,approval_2,approval_3,approved`.
- Judul: `name` (PRICE/RM/2026/06/00001).
- Tab **Harga**: list `supplierinfo_ids` — Vendor, Product Variant, Product, Start Date, End Date, Unit,
  Unit Price, Currency, + kolom **Harga Lama** (dari `replaced_supplierinfo_ids`) dan selisih %.
  Readonly setelah submit.
- Tab **Approval**: list `line_ids` — Tahap, Nama, Peran (Prepared / Acknowledged / Approved), Status, Tanggal, Catatan.
- `<chatter/>` (lampiran + log note + activity).

Menu: *Purchase → Products → Pricelist Approval* (Semua, Menunggu Saya, Lewat Batas Waktu).

---

## 5. Install & Uji

### 5.1 Install
```bash
./odoo-bin -c <config> -d <db_dev> -i vit_vendor_pricelist_approval --stop-after-init
```
Lalu lakukan setting bagian 3.1–3.3. Cek setelah install: semua `product.supplierinfo` lama
`active = True` dan `approval_state = 'approved'`:
```sql
SELECT active, approval_state, count(*) FROM product_supplierinfo GROUP BY 1, 2;
```

### 5.2 Skenario uji

| # | Skenario | Hasil yang diharapkan |
|---|----------|-----------------------|
| 1 | Anggi import 12 harga RM (tanpa kolom `id`), klik **Test** | Tidak ada dokumen PRICE tersisa (rollback) |
| 2 | Import yang sama, klik **Import** | Harga tersimpan `active=False`; dokumen `PRICE/RM/2026/..` Draft berisi 12 baris |
| 3 | Buat PO produk tersebut | Harga masih harga lama (harga baru belum dipakai) |
| 4 | Import file campuran RM + Coating | Dua dokumen: `PRICE/RM/...` dan `PRICE/COAT/...` |
| 5 | Import file berisi kolom `id` milik harga aktif | Ditolak dengan pesan hapus kolom `id` |
| 6 | Submit tanpa lampiran / 1 lampiran | Ditolak |
| 7 | Submit lengkap | Tahap 1, activity muncul untuk Rochman, deadline +2 hari |
| 8 | Ronny coba Approve saat masih tahap 1 (via RPC) | Ditolak di server |
| 9 | Rochman approve | Tahap 2; activity muncul **bersamaan** untuk Ronny & Juster |
| 10 | Ronny approve, Juster belum | Masih tahap 2 |
| 11 | Juster reject dengan alasan | Kembali Draft, alasan di chatter, harga tetap tidak aktif |
| 12 | Revisi harga, submit ulang, semua approve sampai tahap 3 | Approved; harga baru aktif; harga lama vendor+produk sama diarsipkan |
| 13 | PO open (belum receipt) item A 1000 | Harga PO jadi 1100, ada log di PO |
| 14 | PO item A sudah receipt, belum bill | Harga PO tetap 1000; bill (bila 4.7.2 aktif) 1100 |
| 15 | Admin coba aktifkan harga `to_approve` dari list / form produk | Ditolak |
| 16 | Import produk kategori Consumable | Langsung aktif seperti alur lama |
| 17 | Dokumen lewat batas waktu | Cron menambah reminder di chatter & activity |

---

## 6. Hal yang perlu dikonfirmasi ke user (Purchasing / Marketing / FA)

- **Paralel dalam departemen**: harus **semua** approver (Ronny **dan** Juster) atau cukup **salah satu**?
  Halaman 8 memberi kotak tanda tangan per orang, README ini mengasumsikan semua.
- **Peran di lembar tanda tangan halaman 8 berbeda dengan tabel halaman 3**: di halaman 8 Agung Santoso tertulis
  *Prepared* dan Muhammad Reza *Approved*, sedangkan di halaman 3 Agung adalah Approval 3 (FA) dan Pak Reza
  follower. Lim Antonius juga muncul di halaman 8 tapi tidak ada di halaman 3. Mana yang benar?
- Batas waktu "3 hari di Marketing dan FA" — 3 hari gabungan, atau 3 hari masing-masing?
  Bila lewat batas waktu, cukup reminder atau ada eskalasi / auto-reject?
- Apakah **update harga PO yang belum di-receipt** dilakukan otomatis, atau cukup daftar PO terdampak
  lalu Purchasing yang update? Termasuk PO yang sudah *confirmed*?
- **Bill pakai harga baru untuk barang yang sudah di-receipt** — FA setuju dengan selisih valuasi
  (price difference) yang muncul?
- Kode kategori final (`RM`, `COMP`, `COAT`, `PRJ`?) dan apakah nomor urut dipisah per kategori.
- Apakah input harga manual dari form produk (tab Vendors) juga wajib lewat approval, atau ditutup sama sekali
  untuk user non-manager?
