<?php
session_start();
// 1. CEK LOGIN - Jika belum login, dilempar ke login.php
if (!isset($_SESSION['admin'])) {
    header("Location: login.php");
    exit();
}

include 'koneksi.php';

// 2. DATA USER & TANGGAL
$nama_user = $_SESSION['admin']['nama_lengkap'];
$level_user = $_SESSION['admin']['level']; // admin atau kasir
$hari_ini = date('Y-m-d');

// 3. QUERY PEMASUKAN - Hanya dihitung jika yang login adalah ADMIN
$jumlah_tampil = 0;
if ($level_user == 'admin') {
    $q_hari_ini = mysqli_query($conn, "SELECT SUM(total_harga) as total FROM transaksi WHERE DATE(tanggal) = '$hari_ini'");
    $data_pemasukan = mysqli_fetch_assoc($q_hari_ini);
    $jumlah_tampil = $data_pemasukan['total'] ?? 0;
}
?>

<!DOCTYPE html>
<html lang="id">
<head>
    <meta charset="UTF-8">
    <title>Dashboard BUMDes Cikahuripan</title>
    <link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/bootstrap@5.3.0/dist/css/bootstrap.min.css">
    <style>
        body { background-color: #f4f7f6; }
        .menu-card { transition: 0.3s; cursor: pointer; border-radius: 15px; }
        .menu-card:hover { transform: translateY(-10px); background-color: #e9ecef; }
    </style>
</head>
<body>

<nav class="navbar navbar-dark bg-dark mb-4">
    <div class="container">
        <span class="navbar-brand">Halo, <b><?= $nama_user; ?></b> (<?= ucfirst($level_user); ?>)</span>
        <a href="logout.php" class="btn btn-outline-danger btn-sm">Keluar</a>
    </div>
</nav>

<div class="container">
    <?php if ($level_user == 'admin') : ?>
        <div class="card shadow-sm border-0 mb-4">
            <div class="card-body p-4">
                <div class="d-flex justify-content-between align-items-center">
                    <h5 class="text-muted mb-0">Total Pemasukan Seluruh Unit (Hari Ini)</h5>
                    <span class="badge bg-success"><?= date('d F Y'); ?></span>
                </div>
                <h1 class="display-3 fw-bold text-success mt-2">Rp <?= number_format($jumlah_tampil); ?></h1>
            </div>
        </div>
    <?php endif; ?>

    <div class="row g-4 justify-content-center">
        
        <div class="col-md-3">
            <div class="card menu-card h-100 text-center p-4 shadow-sm border-0" onclick="location.href='kasir.php'">
                <div class="display-5 mb-2">🛒</div>
                <h6 class="fw-bold">Kasir / Transaksi</h6>
            </div>
        </div>

        <div class="col-md-3">
            <div class="card menu-card h-100 text-center p-4 shadow-sm border-0" onclick="location.href='catatan_hutang.php'">
                <div class="display-5 mb-2">📝</div>
                <h6 class="fw-bold">Catatan Hutang</h6>
            </div>
        </div>

        <?php if ($level_user == 'admin') : ?>
            <div class="col-md-3">
                <div class="card menu-card h-100 text-center p-4 shadow-sm border-0" onclick="location.href='master_barang.php'">
                    <div class="display-5 mb-2">📦</div>
                    <h6 class="fw-bold">Master Barang</h6>
                </div>
            </div>
            <div class="col-md-3">
                <div class="card menu-card h-100 text-center p-4 shadow-sm border-0" onclick="location.href='laporan.php'">
                    <div class="display-5 mb-2">📊</div>
                    <h6 class="fw-bold">Laporan Keuangan</h6>
                </div>
            </div>
        <?php endif; ?>

    </div>
</div>
<?php include 'koneksi.php'; ?>
<!DOCTYPE html>
<html>
<head>
    <title>Kasir BUMDes Mart</title>
    <link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/bootstrap@5.3.0/dist/css/bootstrap.min.css">
</head>
<body class="bg-light">
<div class="container mt-4">
    <div class="row">
        <div class="col-md-8">
            <div class="card shadow-sm mb-4">
                <div class="card-header bg-dark text-white">Kasir I-POS</div>
                <div class="card-body">
                    <div class="mb-3">
                        <label>Pilih Barang</label>
                        <select id="pilih_barang" class="form-select" onchange="tambahBarang()">
                            <option value="">-- Cari Nama Barang / Barcode --</option>
                            <?php
                            $b = mysqli_query($conn, "SELECT * FROM barang WHERE stok > 0");
                            while($row = mysqli_fetch_assoc($b)){
                                echo "<option value='".$row['id']."' data-nama='".$row['nama_barang']."' data-harga='".$row['harga_jual']."'>".$row['nama_barang']." (Rp ".number_format($row['harga_jual']).")</option>";
                            }
                            ?>
                        </select>
                    </div>
                    <table class="table table-bordered" id="tabel_keranjang">
                        <thead class="table-secondary">
                            <tr>
                                <th>Barang</th>
                                <th>Harga</th>
                                <th>Qty</th>
                                <th>Subtotal</th>
                            </tr>
                        </thead>
                        <tbody></tbody>
                    </table>
                </div>
            </div>
        </div>

        <div class="col-md-4">
        <div class="d-flex justify-content-between align-items-center mb-3">
             <a href="index.php" class="btn btn-outline-secondary">
               <i class="bi bi-arrow-left"></i> Kembali ke Dashboard
                 </a>
                  <h4 class="mb-0 text-primary fw-bold">Kasir I-POS BUMDes Mart</h4>
               </div>
            
            <div class="card shadow-sm">
                <div class="card-body">
                    <form action="proses_mart.php" method="POST">
                        <h3 class="text-end mb-4" id="total_cart">Rp 0</h3>
                        <input type="hidden" name="total_harga" id="input_total">
                        
                        <div class="mb-3">
                            <label>Metode Pembayaran</label>
                            <select name="metode_bayar" class="form-select" required>
                                <option value="Tunai">Kas Tunai</option>
                                <option value="BRI">Transfer BRI</option>
                                <option value="BJB">Transfer BJB</option>
                                <option value="Mandiri">Transfer Mandiri</option>
                                <option value="Dana">Dana BUMDes</option>
                                <option value="Hutang">Hutang (Cash Bon)</option>
                            </select>
                        </div>
                       
                        <div class="mb-3">
                            <label>Bayar (Rp)</label>
                            <input type="number" name="bayar" id="bayar" class="form-control" oninput="hitungKembalian()">
                        </div>
                        <div class="mb-3">
                            <label>Kembalian</label>
                            <input type="text" id="kembali" class="form-control" readonly>
                        </div>
                                        
                        <button type="submit" class="btn btn-primary w-100 btn-lg">Proses Transaksi</button>
                    </form>
                </div>
            </div>
        </div>
    </div>
</div>

<script>
let totalSemua = 0;
function tambahBarang() {
    let sel = document.getElementById("pilih_barang");
    let nama = sel.options[sel.selectedIndex].getAttribute("data-nama");
    let harga = sel.options[sel.selectedIndex].getAttribute("data-harga");
    
    if(!nama) return;

    let table = document.getElementById("tabel_keranjang").getElementsByTagName('tbody')[0];
    let row = table.insertRow();
    row.innerHTML = `<td>${nama}</td><td>${harga}</td><td>1</td><td>${harga}</td>`;
    
    totalSemua += parseInt(harga);
    document.getElementById("total_cart").innerText = "Rp " + totalSemua.toLocaleString('id-ID');
    document.getElementById("input_total").value = totalSemua;
}

function hitungKembalian() {
    let bayar = document.getElementById("bayar").value;
    let kembali = bayar - totalSemua;
    document.getElementById("kembali").value = "Rp " + kembali.toLocaleString('id-ID');
}
</script>
</body>
</html>
<?php
session_start();
include 'koneksi.php';

// Jika sudah login, langsung lempar ke dashboard
if (isset($_SESSION['admin'])) {
    header("Location: index.php");
    exit();
}

if (isset($_POST['login'])) {
    $username = mysqli_real_escape_string($conn, $_POST['username']);
    $password = mysqli_real_escape_string($conn, $_POST['password']);

    $query = mysqli_query($conn, "SELECT * FROM admin WHERE username='$username' AND password='$password'");
    if (mysqli_num_rows($query) > 0) {
        $_SESSION['admin'] = mysqli_fetch_assoc($query);
        header("Location: index.php");
        exit();
    } else {
        $error = "Username atau Password Salah!";
    }
}
?>
<!DOCTYPE html>
<html lang="id">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Login Admin - BUMDes CMMS</title>
    <link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/bootstrap@5.3.0/dist/css/bootstrap.min.css">
    <style>
        body { 
            background: #f0f2f5; 
            height: 100vh; 
            display: flex; 
            align-items: center; 
            justify-content: center;
            overflow: auto !important; /* Paksa agar layar bisa scroll dan diklik */
        }
        .login-card { 
            width: 100%; 
            max-width: 400px; 
            border: none; 
            border-radius: 15px; 
            z-index: 9999; /* Pastikan kartu berada di paling depan */
            position: relative;
        }
        /* MENGHAPUS PENGHALANG LAYAR GELAP */
        .modal-backdrop { display: none !important; }
        .modal-open { overflow: auto !important; padding-right: 0 !important; }
    </style>
</head>
<body>

    <div class="card login-card shadow-lg p-4">
        <div class="text-center mb-4">
            <img src="logo cmmsc.jpg" width="80" class="mb-2">
            <h4 class="fw-bold mb-0">Login Admin</h4>
            <small class="text-muted text-uppercase">BUMDes Cikahuripan - Klapanunggal</small>
        </div>

        <?php if(isset($error)): ?>
            <div class="alert alert-danger py-2 small text-center"><?= $error; ?></div>
        <?php endif; ?>

        <form method="POST" action="">
            <div class="mb-3">
                <label class="form-label fw-bold">Username</label>
                <input type="text" name="username" class="form-control" placeholder="admin" required autofocus>
            </div>
            <div class="mb-3">
                <label class="form-label fw-bold">Password</label>
                <input type="password" name="password" class="form-control" placeholder="•••••" required>
            </div>
            <button type="submit" name="login" class="btn btn-primary w-100 py-2 fw-bold shadow-sm">Masuk ke Sistem</button>
        </form>
    </div>

</body>
</html>
<script src="https://cdn.jsdelivr.net/npm/bootstrap@5.3.0/dist/js/bootstrap.bundle.min.js"></script>
</body>
</html>
