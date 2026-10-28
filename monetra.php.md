# Codingan
<?php
/**
 * ==============================================================================
 * Nama Aplikasi : Money Tracker (Web-based Financial Log)
 * Mata Kuliah   : Algoritma & Struktur Data / Pemrograman Web
 * Sub-CPMK-3    : Implementasi Algoritma Dasar, Searching, Sorting, Stack, Queue
 * ==============================================================================
 * Alur data:
 *   Tambah transaksi -> QUEUE (antrean, FIFO)
 *   Proses antrean   -> dequeue dari QUEUE, lalu push ke STACK (riwayat resmi)
 *   Undo             -> pop dari STACK (transaksi terakhir dibatalkan, LIFO)
 *   Tampilan riwayat -> Bubble Sort (nominal) / Selection Sort (keterangan)
 *   Pencarian        -> Binary Search (nominal), Linear Search (keterangan)
 * ==============================================================================
 */

session_start();

/* ==========================================================================
 * FUNGSI BANTU
 * ========================================================================== */

// Escape output HTML agar aman dari XSS
function e($teks)
{
    return htmlspecialchars((string) $teks, ENT_QUOTES, 'UTF-8');
}

function rupiah($angka)
{
    return 'Rp ' . number_format($angka, 0, ',', '.');
}

// Redirect ke halaman ini sendiri (apa pun nama file / foldernya), tanpa query string
function redirectSelf()
{
    $path = strtok($_SERVER['REQUEST_URI'], '?');
    header('Location: ' . $path);
    exit;
}

function setFlash($tipe, $pesan)
{
    $_SESSION['flash'] = ['tipe' => $tipe, 'pesan' => $pesan];
}

// Buat URL halaman ini dengan parameter GET tertentu diganti
function urlWith(array $ganti)
{
    $path = strtok($_SERVER['REQUEST_URI'], '?');
    $query = http_build_query(array_merge($_GET, $ganti));
    return $path . ($query !== '' ? '?' . $query : '');
}

/* ==========================================================================
 * 1. BUBBLE SORT - mengurutkan transaksi berdasarkan jumlah (kecil -> besar)
 * ========================================================================== */
function bubbleSortTransactions($data)
{
    $n = count($data);
    for ($i = 0; $i < $n - 1; $i++) {
        $tukar = false;
        for ($j = 0; $j < $n - $i - 1; $j++) {
            if ($data[$j]['jumlah'] > $data[$j + 1]['jumlah']) {
                $temp = $data[$j];
                $data[$j] = $data[$j + 1];
                $data[$j + 1] = $temp;
                $tukar = true;
            }
        }
        if (!$tukar) {
            break; // sudah terurut, hentikan lebih awal
        }
    }
    return $data;
}

/* ==========================================================================
 * 2. SELECTION SORT - mengurutkan berdasarkan keterangan (A -> Z)
 * ========================================================================== */
function selectionSortByKeterangan($data)
{
    $n = count($data);
    for ($i = 0; $i < $n - 1; $i++) {
        $minIdx = $i;
        for ($j = $i + 1; $j < $n; $j++) {
            if (strcasecmp($data[$j]['keterangan'], $data[$minIdx]['keterangan']) < 0) {
                $minIdx = $j;
            }
        }
        if ($minIdx !== $i) {
            $temp = $data[$i];
            $data[$i] = $data[$minIdx];
            $data[$minIdx] = $temp;
        }
    }
    return $data;
}

/* ==========================================================================
 * 3. BINARY SEARCH - cari nominal persis (data HARUS sudah urut by jumlah)
 * ========================================================================== */
function binarySearchTransaction($sortedData, $targetJumlah)
{
    $left = 0;
    $right = count($sortedData) - 1;

    while ($left <= $right) {
        $mid = intdiv($left + $right, 2);

        if ($sortedData[$mid]['jumlah'] === $targetJumlah) {
            // Kumpulkan semua data kembar di kiri dan kanan titik tengah
            $start = $mid;
            while ($start > 0 && $sortedData[$start - 1]['jumlah'] === $targetJumlah) {
                $start--;
            }
            $end = $mid;
            $total = count($sortedData);
            while ($end < $total - 1 && $sortedData[$end + 1]['jumlah'] === $targetJumlah) {
                $end++;
            }
            return array_slice($sortedData, $start, $end - $start + 1);
        } elseif ($sortedData[$mid]['jumlah'] < $targetJumlah) {
            $left = $mid + 1;
        } else {
            $right = $mid - 1;
        }
    }
    return [];
}

/* ==========================================================================
 * 4. LINEAR SEARCH - cari kata di keterangan
 * ========================================================================== */
function linearSearchByKeterangan($data, $keyword)
{
    $results = [];
    $keyword = strtolower(trim($keyword));
    if ($keyword === '') {
        return $results;
    }

    foreach ($data as $item) {
        if (strpos(strtolower($item['keterangan']), $keyword) !== false) {
            $results[] = $item;
        }
    }
    return $results;
}

/* ==========================================================================
 * 5. STACK (LIFO) - dipakai untuk riwayat transaksi resmi & fitur Undo
 * ========================================================================== */
function stackPush(&$stack, $item)
{
    $stack[] = $item; // taruh di tumpukan paling atas
}

function stackPop(&$stack)
{
    return array_pop($stack); // ambil dari tumpukan paling atas (null jika kosong)
}

function stackPeek($stack)
{
    return empty($stack) ? null : $stack[count($stack) - 1];
}

/* ==========================================================================
 * 6. QUEUE (FIFO) - antrean transaksi yang menunggu diproses
 * ========================================================================== */
function queueEnqueue(&$queue, $item)
{
    $queue[] = $item; // masuk dari belakang
}

function queueDequeue(&$queue)
{
    return array_shift($queue); // keluar dari depan (null jika kosong)
}

function queuePeek($queue)
{
    return empty($queue) ? null : $queue[0];
}

/* ==========================================================================
 * INISIALISASI SESSION
 * ========================================================================== */
if (!isset($_SESSION['transaksi']) || !is_array($_SESSION['transaksi'])) {
    $_SESSION['transaksi'] = []; // berperan sebagai STACK
}
if (!isset($_SESSION['queue_antrean']) || !is_array($_SESSION['queue_antrean'])) {
    $_SESSION['queue_antrean'] = []; // berperan sebagai QUEUE
}
if (empty($_SESSION['csrf'])) {
    $_SESSION['csrf'] = bin2hex(random_bytes(32));
}

/* ==========================================================================
 * PROSES FORM (POST) -> setelah selesai selalu redirect (pola PRG)
 * ========================================================================== */
if ($_SERVER['REQUEST_METHOD'] === 'POST') {
    $tokenKiriman = $_POST['csrf'] ?? '';
    if (!hash_equals($_SESSION['csrf'], (string) $tokenKiriman)) {
        setFlash('error', 'Sesi form tidak valid. Silakan coba lagi.');
        redirectSelf();
    }

    $action = $_POST['action'] ?? '';

    if ($action === 'tambah') {
        $jenis = $_POST['jenis'] ?? '';
        $keterangan = trim($_POST['keterangan'] ?? '');
        $jumlah = filter_var(
            $_POST['jumlah'] ?? '',
            FILTER_VALIDATE_INT,
            ['options' => ['min_range' => 1, 'max_range' => 1000000000000]]
        );

        if (!in_array($jenis, ['pemasukan', 'pengeluaran'], true)) {
            setFlash('error', 'Jenis transaksi tidak valid.');
        } elseif ($keterangan === '' || mb_strlen($keterangan) > 100) {
            setFlash('error', 'Keterangan wajib diisi (maksimal 100 karakter).');
        } elseif ($jumlah === false) {
            setFlash('error', 'Jumlah harus berupa angka bulat lebih dari 0.');
        } else {
            // Transaksi baru masuk ANTREAN dulu (enqueue)
            queueEnqueue($_SESSION['queue_antrean'], [
                'id'         => uniqid('trx_', true),
                'jenis'      => $jenis,
                'keterangan' => $keterangan,
                'jumlah'     => $jumlah,
            ]);
            setFlash('ok', 'Transaksi masuk antrean. Klik "Proses antrean" untuk mencatatnya.');
        }
    } elseif ($action === 'proses_queue') {
        // QUEUE dequeue (FIFO) -> hasilnya di-push ke STACK riwayat
        $item = queueDequeue($_SESSION['queue_antrean']);
        if ($item === null) {
            setFlash('error', 'Antrean kosong. Tambahkan transaksi terlebih dahulu.');
        } else {
            stackPush($_SESSION['transaksi'], $item);
            setFlash('ok', 'Antrean terdepan "' . $item['keterangan'] . '" sudah dicatat ke riwayat.');
        }
    } elseif ($action === 'undo_stack') {
        // STACK pop (LIFO) -> batalkan transaksi yang terakhir dicatat
        $item = stackPop($_SESSION['transaksi']);
        if ($item === null) {
            setFlash('error', 'Riwayat kosong, tidak ada yang bisa dibatalkan.');
        } else {
            setFlash('ok', 'Transaksi "' . $item['keterangan'] . '" dibatalkan.');
        }
    } elseif ($action === 'reset') {
        $_SESSION['transaksi'] = [];
        $_SESSION['queue_antrean'] = [];
        setFlash('ok', 'Semua data sudah dihapus.');
    }

    redirectSelf();
}

/* ==========================================================================
 * EKSEKUSI ALGORITMA UNTUK TAMPILAN (GET)
 * ========================================================================== */
$flash = $_SESSION['flash'] ?? null;
unset($_SESSION['flash']);

$transaksi = $_SESSION['transaksi'];
$antrean = $_SESSION['queue_antrean'];

// Binary search wajib memakai data yang terurut berdasarkan jumlah
$sortedByJumlah = bubbleSortTransactions($transaksi);

// Pilihan urutan tabel: jumlah (Bubble Sort) atau keterangan (Selection Sort)
$urut = $_GET['urut'] ?? 'jumlah';
if (!in_array($urut, ['jumlah', 'keterangan'], true)) {
    $urut = 'jumlah';
}
$tampil = ($urut === 'keterangan')
    ? selectionSortByKeterangan($transaksi)
    : $sortedByJumlah;

// Pencarian
$searchNominal = filter_var(
    $_GET['cari_jumlah'] ?? '',
    FILTER_VALIDATE_INT,
    ['options' => ['min_range' => 1]]
);
$searchKeyword = trim((string) ($_GET['cari_keyword'] ?? ''));

$binaryResult = ($searchNominal !== false)
    ? binarySearchTransaction($sortedByJumlah, $searchNominal)
    : null;
$linearResult = ($searchKeyword !== '')
    ? linearSearchByKeterangan($transaksi, $searchKeyword)
    : null;

// Kalkulasi keuangan (traversal)
$pemasukan = 0;
$pengeluaran = 0;
foreach ($transaksi as $t) {
    if ($t['jenis'] === 'pemasukan') {
        $pemasukan += $t['jumlah'];
    } else {
        $pengeluaran += $t['jumlah'];
    }
}
$saldo = $pemasukan - $pengeluaran;

$depanAntrean = queuePeek($antrean);
$puncakStack = stackPeek($transaksi);
?>
<!DOCTYPE html>
<html lang="id">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1">
    <title>Money Tracker - Algoritma & Struktur Data</title>
    <style>
        * { box-sizing: border-box; }
        body { margin: 0; padding: 24px; background: #f1f5f9; color: #1e293b; font-family: Arial, sans-serif; }
        .container { max-width: 900px; margin: auto; }
        h1 { color: #0f766e; }
        h2 { margin-top: 0; font-size: 19px; }
        .cards { display: grid; grid-template-columns: repeat(auto-fit, minmax(180px, 1fr)); gap: 14px; }
        .card, .panel { background: white; padding: 20px; border-radius: 14px; margin-bottom: 18px; box-shadow: 0 3px 12px #0000000a; }
        .card p { color: #64748b; margin: 0 0 8px; }
        .amount { font-size: 21px; font-weight: bold; }
        label { display: block; font-size: 14px; }
        input, select, button { width: 100%; padding: 12px; margin-top: 8px; margin-bottom: 12px; border: 1px solid #cbd5e1; border-radius: 8px; font-size: 15px; }
        button { background: #0f766e; color: white; border: none; cursor: pointer; font-weight: bold; }
        button:hover { background: #115e59; }
        button:focus-visible, input:focus-visible, select:focus-visible, a:focus-visible { outline: 3px solid #0ea5e9; outline-offset: 2px; }
        .btn-stack { background: #d97706; }
        .btn-stack:hover { background: #b45309; }
        .btn-queue { background: #7c3aed; }
        .btn-queue:hover { background: #6d28d9; }
        .btn-search { background: #0369a1; }
        .btn-search:hover { background: #075985; }
        .btn-reset { background: #fff; color: #dc2626; border: 1px solid #fca5a5; margin-bottom: 0; }
        .btn-reset:hover { background: #fef2f2; }
        table { width: 100%; border-collapse: collapse; }
        th, td { padding: 12px 6px; border-bottom: 1px solid #e2e8f0; text-align: left; }
        .masuk { color: #15803d; }
        .keluar { color: #dc2626; }
        .table-wrap { overflow-x: auto; }
        .grid-2 { display: grid; grid-template-columns: 1fr 1fr; gap: 15px; }
        .status { font-size: 13px; background: #f8fafc; padding: 10px; border-radius: 6px; line-height: 1.6; }
        .flash { padding: 12px 16px; border-radius: 10px; margin-bottom: 18px; }
        .flash.ok { background: #f0fdf4; border: 1px solid #bbf7d0; color: #166534; }
        .flash.error { background: #fef2f2; border: 1px solid #fecaca; color: #991b1b; }
        .sort-links a { display: inline-block; padding: 6px 12px; border-radius: 999px; border: 1px solid #cbd5e1; color: #334155; text-decoration: none; font-size: 14px; margin-right: 6px; }
        .sort-links a.aktif { background: #0f766e; border-color: #0f766e; color: white; }
        .hasil { margin-top: 15px; padding: 12px; border-radius: 8px; }
        .hasil ul { margin: 0; padding-left: 20px; }
        @media (max-width: 600px) { .grid-2 { grid-template-columns: 1fr; } body { padding: 14px; } }
    </style>
</head>
<body>
<div class="container">
    <h1>💰 Money Tracker</h1>
    <p>Aplikasi keuangan dengan antrean (Queue) dan pembatalan (Stack).</p>

    <?php if ($flash): ?>
        <div class="flash <?= $flash['tipe'] === 'ok' ? 'ok' : 'error' ?>" role="status">
            <?= e($flash['pesan']) ?>
        </div>
    <?php endif; ?>

    <!-- Ringkasan saldo -->
    <div class="cards">
        <div class="card">
            <p>Total saldo</p>
            <div class="amount"><?= rupiah($saldo) ?></div>
        </div>
        <div class="card">
            <p>Total pemasukan</p>
            <div class="amount masuk"><?= rupiah($pemasukan) ?></div>
        </div>
        <div class="card">
            <p>Total pengeluaran</p>
            <div class="amount keluar"><?= rupiah($pengeluaran) ?></div>
        </div>
    </div>

    <div class="grid-2">
        <!-- Form tambah transaksi (masuk antrean dulu) -->
        <div class="panel">
            <h2>Tambah transaksi (Enqueue)</h2>
            <form method="post" action="">
                <input type="hidden" name="csrf" value="<?= e($_SESSION['csrf']) ?>">
                <input type="hidden" name="action" value="tambah">

                <label for="jenis">Jenis transaksi</label>
                <select id="jenis" name="jenis" required>
                    <option value="pemasukan">Pemasukan</option>
                    <option value="pengeluaran">Pengeluaran</option>
                </select>

                <label for="keterangan">Keterangan</label>
                <input id="keterangan" name="keterangan" maxlength="100" placeholder="Contoh: Gaji, Makan" required>

                <label for="jumlah">Jumlah uang (Rp)</label>
                <input id="jumlah" type="number" name="jumlah" min="1" step="1" required>

                <button type="submit">Masukkan ke antrean</button>
            </form>
        </div>

        <!-- Kontrol Queue & Stack -->
        <div class="panel">
            <h2>Kontrol struktur data</h2>

            <form method="post" action="">
                <input type="hidden" name="csrf" value="<?= e($_SESSION['csrf']) ?>">
                <input type="hidden" name="action" value="proses_queue">
                <button type="submit" class="btn-queue" title="FIFO: proses antrean paling depan">
                    Proses antrean (Dequeue)
                </button>
            </form>

            <form method="post" action="">
                <input type="hidden" name="csrf" value="<?= e($_SESSION['csrf']) ?>">
                <input type="hidden" name="action" value="undo_stack">
                <button type="submit" class="btn-stack" title="LIFO: batalkan transaksi terakhir">
                    Batalkan transaksi terakhir (Pop)
                </button>
            </form>

            <div class="status">
                <div><b>Antrean (Queue):</b> <?= count($antrean) ?> transaksi</div>
                <?php if ($depanAntrean !== null): ?>
                    <div style="color:#7c3aed;">
                        Terdepan: <b><?= e($depanAntrean['keterangan']) ?></b> (<?= rupiah($depanAntrean['jumlah']) ?>)
                    </div>
                <?php endif; ?>
                <div><b>Riwayat (Stack):</b> <?= count($transaksi) ?> transaksi</div>
                <?php if ($puncakStack !== null): ?>
                    <div style="color:#d97706;">
                        Puncak: <b><?= e($puncakStack['keterangan']) ?></b> (<?= rupiah($puncakStack['jumlah']) ?>)
                    </div>
                <?php endif; ?>
            </div>
        </div>
    </div>

    <!-- Daftar antrean -->
    <div class="panel">
        <h2>Antrean menunggu diproses</h2>
        <div class="table-wrap">
            <table>
                <thead>
                    <tr><th>Urutan</th><th>Jenis</th><th>Keterangan</th><th>Jumlah</th></tr>
                </thead>
                <tbody>
                <?php if (empty($antrean)): ?>
                    <tr><td colspan="4">Antrean kosong. Tambahkan transaksi lewat form di atas.</td></tr>
                <?php else: ?>
                    <?php foreach ($antrean as $i => $q): ?>
                        <tr>
                            <td><?= $i + 1 ?></td>
                            <td><?= $q['jenis'] === 'pemasukan' ? 'Masuk' : 'Keluar' ?></td>
                            <td><?= e($q['keterangan']) ?></td>
                            <td class="<?= $q['jenis'] === 'pemasukan' ? 'masuk' : 'keluar' ?>"><?= rupiah($q['jumlah']) ?></td>
                        </tr>
                    <?php endforeach; ?>
                <?php endif; ?>
                </tbody>
            </table>
        </div>
    </div>

    <!-- Pencarian -->
    <div class="panel">
        <h2>Pencarian data (Binary &amp; Linear Search)</h2>
        <div class="grid-2">
            <form method="get" action="">
                <input type="hidden" name="urut" value="<?= e($urut) ?>">
                <input type="hidden" name="cari_keyword" value="<?= e($searchKeyword) ?>">
                <label for="cari_jumlah"><strong>Cari nominal (Binary Search)</strong></label>
                <input id="cari_jumlah" type="number" min="1" step="1" name="cari_jumlah"
                       placeholder="Nominal persis, contoh: 50000"
                       value="<?= $searchNominal !== false ? e($searchNominal) : '' ?>">
                <button type="submit">Cari nominal</button>
            </form>

            <form method="get" action="">
                <input type="hidden" name="urut" value="<?= e($urut) ?>">
                <input type="hidden" name="cari_jumlah" value="<?= $searchNominal !== false ? e($searchNominal) : '' ?>">
                <label for="cari_keyword"><strong>Cari keterangan (Linear Search)</strong></label>
                <input id="cari_keyword" type="text" name="cari_keyword"
                       placeholder="Kata di keterangan..."
                       value="<?= e($searchKeyword) ?>">
                <button type="submit" class="btn-search">Cari teks</button>
            </form>
        </div>

        <?php if ($binaryResult !== null): ?>
            <div class="hasil" style="background:#f0fdf4; border:1px solid #bbf7d0;">
                <p style="margin:0 0 8px; color:#166534;"><b>Hasil Binary Search (<?= rupiah($searchNominal) ?>):</b></p>
                <?php if (!empty($binaryResult)): ?>
                    <ul>
                        <?php foreach ($binaryResult as $res): ?>
                            <li><?= $res['jenis'] === 'pemasukan' ? 'Pemasukan' : 'Pengeluaran' ?> - <?= e($res['keterangan']) ?> (<?= rupiah($res['jumlah']) ?>)</li>
                        <?php endforeach; ?>
                    </ul>
                <?php else: ?>
                    <p style="margin:0; color:#dc2626;">Tidak ada transaksi dengan nominal itu di riwayat resmi.</p>
                <?php endif; ?>
            </div>
        <?php endif; ?>

        <?php if ($linearResult !== null): ?>
            <div class="hasil" style="background:#f0f9ff; border:1px solid #bae6fd;">
                <p style="margin:0 0 8px; color:#0369a1;"><b>Hasil Linear Search ("<?= e($searchKeyword) ?>"):</b></p>
                <?php if (!empty($linearResult)): ?>
                    <ul>
                        <?php foreach ($linearResult as $res): ?>
                            <li><?= $res['jenis'] === 'pemasukan' ? 'Pemasukan' : 'Pengeluaran' ?> - <?= e($res['keterangan']) ?> (<?= rupiah($res['jumlah']) ?>)</li>
                        <?php endforeach; ?>
                    </ul>
                <?php else: ?>
                    <p style="margin:0; color:#dc2626;">Tidak ada transaksi yang cocok dengan kata itu.</p>
                <?php endif; ?>
            </div>
        <?php endif; ?>
    </div>

    <!-- Tabel riwayat -->
    <div class="panel">
        <h2>Riwayat transaksi resmi</h2>
        <div class="sort-links" style="margin-bottom:12px;">
            <a href="<?= e(urlWith(['urut' => 'jumlah'])) ?>" class="<?= $urut === 'jumlah' ? 'aktif' : '' ?>">
                Urut jumlah (Bubble Sort)
            </a>
            <a href="<?= e(urlWith(['urut' => 'keterangan'])) ?>" class="<?= $urut === 'keterangan' ? 'aktif' : '' ?>">
                Urut keterangan (Selection Sort)
            </a>
        </div>
        <div class="table-wrap">
            <table>
                <thead>
                    <tr><th>Jenis</th><th>Keterangan</th><th>Jumlah</th></tr>
                </thead>
                <tbody>
                <?php if (empty($tampil)): ?>
                    <tr><td colspan="3">Belum ada transaksi resmi. Proses antrean di atas untuk mencatatnya.</td></tr>
                <?php else: ?>
                    <?php foreach ($tampil as $t): ?>
                        <tr>
                            <td><?= $t['jenis'] === 'pemasukan' ? 'Masuk' : 'Keluar' ?></td>
                            <td><?= e($t['keterangan']) ?></td>
                            <td class="<?= $t['jenis'] === 'pemasukan' ? 'masuk' : 'keluar' ?>"><?= rupiah($t['jumlah']) ?></td>
                        </tr>
                    <?php endforeach; ?>
                <?php endif; ?>
                </tbody>
            </table>
        </div>
    </div>

    <!-- Reset -->
    <form method="post" action="" onsubmit="return confirm('Hapus semua transaksi dan antrean?');">
        <input type="hidden" name="csrf" value="<?= e($_SESSION['csrf']) ?>">
        <input type="hidden" name="action" value="reset">
        <button type="submit" class="btn-reset">Hapus semua data</button>
    </form>
</div>
</body>
</html>
