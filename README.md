# Praktikum 11 PHP OOP Lanjutan

**Mata Kuliah:** Pemrograman Web1

**Dosen:** Agung Nugroho

**Nama:** Zaenal Maulana Rizki

**NIM:** 312410332

---

# 📁 **1. Persiapan Struktur Folder**

Saya membuat folder utama:

```
lab11_php_oop/
├── .htaccess
├── config.php
├── index.php
├── class/
│   ├── Database.php
│   └── Form.php
├── module/
│   ├
│   └── artikel/
│       ├── index.php
│       ├── tambah.php
│       └── ubah.php
└── template/
    ├── header.php
    ├── footer.php
    └── sidebar.php
```

---

<img width="345" height="430" alt="image" src="https://github.com/user-attachments/assets/78ccdbd0-a378-4bd7-b333-439ed6762c4f" />

---

# ⚙️ **2. Konfigurasi Routing – .htaccess**

Saya membuat file `.htaccess` agar URL bisa diakses seperti:

```
localhost/lab11_php_oop/artikel/tambah
```

Isi file:

```apache
<IfModule mod_rewrite.c>
    RewriteEngine On
    RewriteBase /Lab11Web/

    RewriteCond %{REQUEST_FILENAME} !-d
    RewriteCond %{REQUEST_FILENAME} !-f

    RewriteRule ^(.*)$ index.php/$1 [L]
</IfModule>
```

---

# 🗄 **3. Membuat File config.php**

File konfigurasi database:

```php
<?php
$config = [
    'host' => 'localhost',
    'username' => 'root',
    'password' => '',
    'db_name' => 'latihan_oop'
];
?>
```

---

# 💾 **4. Memasukkan Database.php & Form.php ke folder class**

Saya mengambil class dari praktikum sebelumnya dan memindahkannya ke:

class/Database.php  
```
<?php

class Database
{
    protected $host;
    protected $user;
    protected $password;
    protected $db_name;
    protected $conn;

    public function __construct()
    {
        $this->getConfig();
        $this->conn = new mysqli(
            $this->host,
            $this->user,
            $this->password,
            $this->db_name
        );

        if ($this->conn->connect_error) {
            die("Connection failed: " . $this->conn->connect_error);
        }
    }

    private function getConfig()
    {
        include("config.php");
        $this->host = $config['host'];
        $this->user = $config['username'];
        $this->password = $config['password'];
        $this->db_name = $config['db_name'];
    }

    public function query($sql)
    {
        return $this->conn->query($sql);
    }

    public function get($table, $where = null)
    {
        if ($where) {
            $where = " WHERE " . $where;
        }
        $sql = "SELECT * FROM " . $table . $where;
        $sql = $this->conn->query($sql);
        return $sql->fetch_assoc();
    }

    public function insert($table, $data)
    {
        foreach ($data as $key => $val) {
            $column[] = $key;
            $value[] = "'{$val}'";
        }

        $columns = implode(",", $column);
        $values = implode(",", $value);

        $sql = "INSERT INTO " . $table . " (" . $columns . ") VALUES (" . $values . ")";
        return $this->conn->query($sql);
    }

    public function update($table, $data, $where)
    {
        foreach ($data as $key => $val) {
            $update_value[] = "$key='{$val}'";
        }

        $update_value = implode(",", $update_value);

        $sql = "UPDATE " . $table . " SET " . $update_value . " WHERE " . $where;
        return $this->conn->query($sql);
    }
}
?>
```
class/Form.php
```
<?php

/**
 * Nama Class: Form
 * Deskripsi: Class untuk membuat form inputan dinamis
 */

class Form
{
    private $fields = array();
    private $action;
    private $submit = "Submit Form";
    private $jumField = 0;

    public function __construct($action, $submit)
    {
        $this->action = $action;
        $this->submit = $submit;
    }

    public function displayForm()
    {
        echo "<form action='" . $this->action . "' method='POST'>";
        echo '<table width="100%" border="0">';

        foreach ($this->fields as $field) {
            echo "<tr><td align='right' valign='top'>" . $field['label'] . "</td>";
            echo "<td>";

            switch ($field['type']) {

                case 'textarea':
                    echo "<textarea name='" . $field['name'] . "' cols='30' rows='4'></textarea>";
                    break;

                case 'select':
                    echo "<select name='" . $field['name'] . "'>";
                    foreach ($field['options'] as $value => $label) {
                        echo "<option value='" . $value . "'>" . $label . "</option>";
                    }
                    echo "</select>";
                    break;

                case 'radio':
                    foreach ($field['options'] as $value => $label) {
                        echo "<label><input type='radio' name='" . $field['name'] . "' value='" . $value . "'> " . $label . "</label> ";
                    }
                    break;

                case 'checkbox':
                    foreach ($field['options'] as $value => $label) {
                        echo "<label><input type='checkbox' name='" . $field['name'] . "[]' value='" . $value . "'> " . $label . "</label> ";
                    }
                    break;

                case 'password':
                    echo "<input type='password' name='" . $field['name'] . "'>";
                    break;

                default:
                    echo "<input type='text' name='" . $field['name'] . "'>";
                    break;
            }

            echo "</td></tr>";
        }

        echo "<tr><td colspan='2'>";
        echo "<input type='submit' value='" . $this->submit . "'></td></tr>";
        echo "</table>";
        echo "</form>";
    }

    public function addField($name, $label, $type = "text", $options = array())
    {
        $this->fields[$this->jumField]['name'] = $name;
        $this->fields[$this->jumField]['label'] = $label;
        $this->fields[$this->jumField]['type'] = $type;
        $this->fields[$this->jumField]['options'] = $options;
        $this->jumField++;
    }
}
?>
```
---

# 🚏 **5. Membuat Router Utama – index.php**

File `index.php` berfungsi sebagai *front controller* (router):

```php
<?php
include "config.php";
include "class/Database.php";
include "class/Form.php";

session_start();

// Ambil PATH_INFO → /artikel/tambah
$path = isset($_SERVER['PATH_INFO']) ? $_SERVER['PATH_INFO'] : '/home/index';

$segments = explode('/', trim($path, '/'));

$mod  = $segments[0] ?? 'home';
$page = $segments[1] ?? 'index';

$file = "module/{$mod}/{$page}.php";

include "template/header.php";

if (file_exists($file)) {
    include $file;
} else {
    echo "<h3>Modul tidak ditemukan: $mod / $page</h3>";
}

include "template/footer.php";
```

---

# 🧱 **6. Membuat Template (header, footer)**

### **header.php**

```php
<!DOCTYPE html>
<html>
<head>
    <title>Lab 11 PHP OOP</title>
    <style>
        body { font-family: Arial; padding: 20px; }
        .menu a { margin-right: 20px; }
    </style>
</head>
<body>

<div class="menu">
    <a href="/Lab11Web/home/index">Home</a>
    <a href="/Lab11Web/artikel/index">Artikel</a>
    <a href="/Lab11Web/artikel/tambah">Tambah Artikel</a>
</div>
<hr>
```

### **footer.php**

```php
</div>
</div>
</body>
</html>
```
---

# 📰 **7. Membuat Modul Artikel**

## ✔ Modul: artikel/index.php

Menampilkan seluruh data:

```php
<?php
$db = new Database();
$data = $db->query("SELECT * FROM artikel");
?>

<h2>Data Artikel</h2>
<a href="/lab11_php_oop/artikel/tambah">+ Tambah</a><br><br>

<table border="1" cellpadding="5">
<tr>
    <th>ID</th>
    <th>Judul</th>
    <th>Isi</th>
    <th>Aksi</th>
</tr>

<?php while($row = $data->fetch_assoc()): ?>
<tr>
    <td><?= $row['id'] ?></td>
    <td><?= $row['judul'] ?></td>
    <td><?= $row['isi'] ?></td>
    <td>
        <a href="/lab11_php_oop/artikel/ubah?id=<?= $row['id'] ?>">Ubah</a>
    </td>
</tr>
<?php endwhile; ?>
</table>
```
---

<img width="1919" height="353" alt="image" src="https://github.com/user-attachments/assets/0146a19a-6d6f-48ae-8781-21bfed26ec61" />

---

## ✔ Modul: artikel/tambah.php

Form menambah data artikel:

```php
<?php
$form = new Form("", "Simpan Artikel");
$db = new Database();

if ($_POST) {
    $db->insert("artikel", [
        'judul' => $_POST['judul'],
        'isi'   => $_POST['isi']
    ]);
    echo "<script>location='/lab11_php_oop/artikel/index';</script>";
}
?>

<h2>Tambah Artikel</h2>

<?php
$form->addField("judul", "Judul Artikel");
$form->addField("isi", "Isi Artikel", "textarea");
$form->displayForm();
?>
```
---

<img width="1919" height="493" alt="image" src="https://github.com/user-attachments/assets/6c349c9e-d505-461c-aaa2-105623aad8dd" />

---

## ✔ Modul: artikel/ubah.php

Form edit artikel:

```php
<?php
$db = new Database();
$id = $_GET['id'];

$artikel = $db->get("artikel", "id=$id");

$form = new Form("", "Update Artikel");

if ($_POST) {
    $db->update("artikel", [
        'judul' => $_POST['judul'],
        'isi'   => $_POST['isi']
    ], "id=$id");

    echo "<script>location='/lab11_php_oop/artikel/index';</script>";
}
?>

<h2>Ubah Artikel</h2>

<?php
$form->addField("judul", "Judul", "text");
$form->addField("isi", "Isi", "textarea");
$form->displayForm();
?>
```
---

<img width="1919" height="372" alt="image" src="https://github.com/user-attachments/assets/a53f6537-069d-4f00-98c9-9c7ba5337f18" />

---

# 🗂 **8. Database MySQL (artikel.sql)**

```
CREATE TABLE artikel (
    id INT AUTO_INCREMENT PRIMARY KEY,
    judul VARCHAR(255),
    isi TEXT
);
```
---

<img width="643" height="353" alt="image" src="https://github.com/user-attachments/assets/6b2e06b4-f917-407c-9bc6-b4b42464c360" />

---

# 🎯 **9. Kesimpulan Praktikum**

Pada praktikum ini, saya berhasil:

✔ Mengimplementasikan routing manual menggunakan `index.php`

✔ Membuat framework modular sederhana

✔ Menggunakan `Form` sebagai dynamic form builder

✔ Menggunakan Class `Database` untuk query CRUD

✔ Membuat modul artikel (tambah, tampil, ubah)

✔ Menggunakan template layout (header, sidebar, footer)

Praktikum ini memperkenalkan konsep dasar bagaimana framework modern seperti Laravel atau CodeIgniter bekerja.

---
