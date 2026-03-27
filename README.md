/controle-portaria
│
├── banco.sql
├── config.php
├── conexao.php
├── auth.php
├── menu.php
├── login.php
├── logout.php
├── dashboard.php
├── index.php
├── salvar.php
├── buscar_pessoa.php
├── saida.php
├── moradores.php
├── encomendas.php
├── dar_baixa.php
├── whatsapp.php

CREATE DATABASE portaria;
USE portaria;

CREATE TABLE usuarios (
    id INT AUTO_INCREMENT PRIMARY KEY,
    usuario VARCHAR(50),
    senha VARCHAR(255)
);

INSERT INTO usuarios (usuario, senha)
VALUES ('admin', '$2y$10$exemploHashAqui'); -- usar password_hash

CREATE TABLE moradores (
    id INT AUTO_INCREMENT PRIMARY KEY,
    nome VARCHAR(100),
    apartamento VARCHAR(20),
    telefone VARCHAR(20)
);

CREATE TABLE pessoas (
    id INT AUTO_INCREMENT PRIMARY KEY,
    nome VARCHAR(100),
    documento VARCHAR(50) UNIQUE,
    telefone VARCHAR(20),
    tipo ENUM('visitante','prestador'),
    empresa VARCHAR(100)
);

CREATE TABLE acessos (
    id INT AUTO_INCREMENT PRIMARY KEY,
    pessoa_id INT,
    morador_id INT,
    tipo_visita ENUM('visitante','prestador'),
    data_entrada DATETIME,
    data_saida DATETIME,
    status ENUM('dentro','saiu') DEFAULT 'dentro'
);

CREATE TABLE encomendas (
    id INT AUTO_INCREMENT PRIMARY KEY,
    morador_id INT,
    telefone VARCHAR(20),
    descricao VARCHAR(255),
    data_recebimento DATETIME DEFAULT CURRENT_TIMESTAMP,
    status VARCHAR(20) DEFAULT 'pendente'
);

CREATE TABLE logs_whatsapp (
    id INT AUTO_INCREMENT PRIMARY KEY,
    telefone VARCHAR(20),
    mensagem TEXT,
    resposta TEXT,
    erro TEXT,
    data_envio DATETIME DEFAULT CURRENT_TIMESTAMP
);

<?php
return [
    "zapi_instance" => "SEU_INSTANCE_ID",
    "zapi_token" => "SEU_TOKEN"
];

<?php
$conn = new mysqli("localhost","root","","portaria");
if($conn->connect_error){
    die("Erro: ".$conn->connect_error);
}
session_start();
?>

<?php
if(!isset($_SESSION['usuario'])){
    header("Location: login.php");
    exit();
}
?>

<?php
function enviarWhatsApp($telefone, $mensagem){

    include 'conexao.php';
    $config = include('config.php');

    $tel = "55".preg_replace('/[^0-9]/','',$telefone);

    $url = "https://api.z-api.io/instances/".$config['zapi_instance']."/token/".$config['zapi_token']."/send-text";

    $data = [
        "phone" => $tel,
        "message" => $mensagem
    ];

    $ch = curl_init($url);
    curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
    curl_setopt($ch, CURLOPT_POSTFIELDS, json_encode($data));
    curl_setopt($ch, CURLOPT_HTTPHEADER, ['Content-Type: application/json']);

    $response = curl_exec($ch);
    $erro = curl_error($ch);

    curl_close($ch);

    // log
    $stmt = $conn->prepare("INSERT INTO logs_whatsapp(telefone,mensagem,resposta,erro) VALUES(?,?,?,?)");
    $stmt->bind_param("ssss",$tel,$mensagem,$response,$erro);
    $stmt->execute();
}

<?php include 'conexao.php'; ?>

<form method="POST">
<input name="usuario" placeholder="Usuário"><br>
<input type="password" name="senha" placeholder="Senha"><br>
<button>Entrar</button>
</form>

<?php
if($_POST){
$u=$_POST['usuario'];

$stmt=$conn->prepare("SELECT * FROM usuarios WHERE usuario=?");
$stmt->bind_param("s",$u);
$stmt->execute();
$res=$stmt->get_result();

if($res->num_rows>0){
$user=$res->fetch_assoc();

if(password_verify($_POST['senha'],$user['senha'])){
$_SESSION['usuario']=$u;
header("Location: dashboard.php");
}else{
echo "Senha inválida";
}
}else{
echo "Usuário não encontrado";
}
}
?>

<?php
include 'conexao.php';
include 'whatsapp.php';

$nome=$_POST['nome'];
$doc=$_POST['documento'];
$tipo=$_POST['tipo'];
$empresa=$_POST['empresa'];
$morador=$_POST['morador_id'];

// verifica duplicidade
$stmt=$conn->prepare("SELECT id FROM pessoas WHERE documento=?");
$stmt->bind_param("s",$doc);
$stmt->execute();
$res=$stmt->get_result();

if($res->num_rows>0){
$id=$res->fetch_assoc()['id'];
}else{
$stmt=$conn->prepare("INSERT INTO pessoas(nome,documento,tipo,empresa) VALUES(?,?,?,?)");
$stmt->bind_param("ssss",$nome,$doc,$tipo,$empresa);
$stmt->execute();
$id=$stmt->insert_id;
}

// registra acesso
$stmt=$conn->prepare("INSERT INTO acessos(pessoa_id,morador_id,tipo_visita,data_entrada) VALUES(?,?,?,NOW())");
$stmt->bind_param("iis",$id,$morador,$tipo);
$stmt->execute();

// busca morador
$stmt=$conn->prepare("SELECT nome,telefone FROM moradores WHERE id=?");
$stmt->bind_param("i",$morador);
$stmt->execute();
$m=$stmt->get_result()->fetch_assoc();

// envia whatsapp
if($m['telefone']){
$msg="Olá ".$m['nome'].", o visitante ".$nome." chegou.";
enviarWhatsApp($m['telefone'],$msg);
}

header("Location: dashboard.php");

<?php
include 'conexao.php';

$id=$_GET['id'];

$stmt=$conn->prepare("UPDATE encomendas SET status='retirada' WHERE id=?");
$stmt->bind_param("i",$id);
$stmt->execute();

header("Location: encomendas.php");
?>

<?php include 'menu.php'; include 'conexao.php'; ?>

<h2>Encomendas</h2>

<form method="POST">
<select name="morador_id">
<?php
$res=$conn->query("SELECT * FROM moradores");
while($m=$res->fetch_assoc()){
echo "<option value='{$m['id']}' data-tel='{$m['telefone']}'>{$m['nome']}</option>";
}
?>
</select>

<input name="telefone" id="telefone">
<input name="descricao">
<button>Salvar</button>
</form>

<hr>

<?php
if($_POST){
$stmt=$conn->prepare("INSERT INTO encomendas(morador_id,telefone,descricao) VALUES(?,?,?)");
$stmt->bind_param("iss",$_POST['morador_id'],$_POST['telefone'],$_POST['descricao']);
$stmt->execute();
}

$res=$conn->query("SELECT e.*,m.nome FROM encomendas e JOIN moradores m ON m.id=e.morador_id");

while($r=$res->fetch_assoc()){

echo "<div style='border:1px solid #ccc;padding:10px;margin:5px'>";

echo "<b>".$r['nome']."</b> - ".$r['descricao']."<br>";

if($r['status']=='pendente'){
echo "<a href='dar_baixa.php?id=".$r['id']."' onclick='return confirm(\"Confirmar retirada?\")'>
<button>Dar Baixa</button></a>";
}else{
echo "✔ Retirada";
}

echo "</div>";
}
?>
