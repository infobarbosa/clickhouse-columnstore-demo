# ClickHouse - Columnstore Demo
Author: Prof. Barbosa<br>
Contact: infobarbosa@gmail.com<br>
Github: [infobarbosa](https://github.com/infobarbosa)

## Objetivo
Avaliar de forma rudimentar o comportamento do modelo de armazenamento baseado em **coluna**.<br>
Para isso faremos uso do [ClickHouse](https://clickhouse.com), um banco de dados colunar open source, inspecionando diretamente os seus **arquivos de dados em disco**.

Ao final deste laboratório você terá visto, no sistema de arquivos:
1. Cada coluna de uma tabela gravada em um **arquivo próprio**;
2. Os valores de uma mesma coluna gravados **lado a lado**, em sequência;
3. A **compressão** acontecendo coluna a coluna, com taxas muito diferentes entre elas;
4. Uma consulta lendo do disco **apenas as colunas de que precisa**.

> Se você fez o laboratório [mysql-rowstore-demo](https://github.com/infobarbosa/mysql-rowstore-demo), compare: lá os campos de **um mesmo registro** apareciam lado a lado no arquivo `.ibd`. Aqui será o oposto.

## Ambiente
Este laboratório é executado em uma instância **AWS EC2** (`t3.medium`) provisionada automaticamente via **AWS CloudShell**.<br>
A instância executa um container Docker com o **code-server** (Visual Studio Code direto no navegador) e o **ClickHouse** já embutidos. Todos os comandos do tutorial são executados no terminal do code-server.

> **ATENÇÃO:**
> 1. **Região:** Utilize sempre a região **Norte da Virgínia (`us-east-1`)** no Console AWS.
> 2. **Custos:** A instância EC2 e o volume EBS consomem créditos do seu laboratório AWS Academy. Ao encerrar a aula, garanta que a instância esteja parada (**Stopped**) ou aguarde o desligamento automático da sessão.
> 3. **Não recrie a máquina:** Se a instância já existir, siga as instruções de [Retomando o laboratório](#retomando-o-laboratório).

### Passo 1: Abrir o AWS CloudShell
1. Faça login no Console da AWS através do seu painel do AWS Academy.
2. Certifique-se de que a região selecionada no canto superior direito é **N. Virginia (`us-east-1`)**.
3. No topo do console, clique no ícone do **AWS CloudShell** (`>_`) ao lado da barra de pesquisa.
4. Aguarde o terminal do CloudShell carregar.

### Passo 2: Executar o script de provisionamento
No terminal do CloudShell, execute:
```
curl -sS https://raw.githubusercontent.com/infobarbosa/clickhouse-columnstore-demo/main/launch-lab.sh | bash
```

O script realizará de forma automatizada:
- Descoberta da VPC padrão e de uma Subnet pública.
- Criação do Security Group `lab-clickhouse-sg` liberando as portas `8080` (IDE) e `22` (SSH).
- Lançamento da instância EC2 `lab-clickhouse-columnstore` (`t3.medium`, Ubuntu 24.04, 20 GB gp3).
- Instalação do Docker e inicialização do container `clickhouse-lab` (code-server + ClickHouse).
- Exibição do IP público e do link de acesso.

### Passo 3: Acessar a IDE
1. Abra no navegador o link exibido ao final do script: `http://<IP-PUBLICO>:8080`.
2. Caso a página não responda, **aguarde de 3 a 5 minutos** para que o Docker conclua o download da imagem e atualize o navegador.
3. O code-server abrirá diretamente na IDE, sem solicitar senha.
4. Abra um terminal: menu superior (três linhas horizontais) > **Terminal** > **New Terminal**.

Verifique se o ClickHouse está no ar:
```
clickhouse-client -q "SELECT version();"

```

Output esperado (a versão pode variar):
```
26.8.21.10
```

Os arquivos de dados usados no laboratório já vêm na imagem, na pasta `/data`. Não é preciso clonar nem baixar nada:
```
ls -la /data

```

Output esperado:
```
total 7944
drwxr-xr-x 2 barbosa barbosa    4096 Oct 16 02:01 .
drwxr-xr-x 1 root    root       4096 Oct 16 02:06 ..
-rw-r--r-- 1 barbosa barbosa  395636 Oct 16 02:01 clientes.csv.gz
-rw-r--r-- 1 barbosa barbosa 7726488 Oct 16 02:01 invoices.tar.gz
```

### Retomando o laboratório
Se a sessão do AWS Academy expirar, a instância será parada. **Não execute o `launch-lab.sh` novamente.**
1. No Console AWS (`us-east-1`), acesse **EC2** > **Instances**.
2. Selecione a instância **`lab-clickhouse-columnstore`** e clique em **Instance state** > **Start instance**.
3. Aguarde o status **Running** e copie o novo **Public IPv4 address** (ele muda a cada reinício).
4. Acesse `http://<NOVO_IP_PUBLICO>:8080`. O container reinicia automaticamente e os dados do ClickHouse são preservados.

### Convenções
Cada bloco de comando tem na primeira linha um rótulo, como `# 3.b.1` (subseção 3.b, primeiro bloco). Os blocos sem rótulo, logo depois, mostram a saída esperada.

Alguns valores **mudam a cada execução**: os caminhos das parts, os *checksums* no início dos arquivos e as datas. Os seus não vão bater com os do texto, e isso é esperado. Contagens de linhas e tamanhos de dados, essas sim, devem bater.

Os comandos funcionam a partir de qualquer pasta. Algumas seções guardam um caminho em uma variável (`$PART`, `$TABELA`); se você abrir um novo terminal no meio de uma seção, rode de novo o bloco que define a variável.

## 1. A tabela `cliente`

### 1.a. Database `ecommerce`
```sh
# 1.a.1
clickhouse-client -q "CREATE DATABASE IF NOT EXISTS ecommerce;"

```

### 1.b. Tabela `cliente`
```sh
# 1.b.1
clickhouse-client -q "
CREATE TABLE ecommerce.cliente (
    id        UInt64  CODEC(NONE),
    nome      String  CODEC(NONE),
    data_nasc Date32  CODEC(NONE),
    cpf       String  CODEC(NONE),
    email     String  CODEC(NONE),
    cidade    String  CODEC(NONE),
    uf        String  CODEC(NONE)
)
ENGINE = MergeTree
ORDER BY id
SETTINGS min_bytes_for_wide_part = 0;"

```

> **Entendendo a criação da tabela:**
> - `ENGINE = MergeTree`: é o mecanismo de armazenamento colunar padrão do ClickHouse.
> - `CODEC(NONE)`: por padrão o ClickHouse **comprime** cada coluna (LZ4), o que deixaria os arquivos ilegíveis. Desligamos a compressão nesta tabela para enxergar os dados brutos. Mais adiante veremos a compressão em ação.
> - `min_bytes_for_wide_part = 0`: por padrão, parts com menos de 10 MB são gravadas em formato *compact*: um único arquivo `data.bin`, em que os dados são divididos em blocos de 8.192 linhas e, **dentro de cada bloco, as colunas ficam separadas** (organização semelhante à do formato Parquet). Parts maiores usam o formato *wide*, com **um arquivo por coluna**. Como nossa tabela terá poucas linhas, forçamos o formato *wide* para visualizar a separação com mais clareza. Na seção 7 você verá a tabela `invoices` ser gravada em formato *wide* sem nenhum ajuste.
> - `Date32`: tipo de data que aceita valores a partir de 1900 (o tipo `Date` começa em 1970).

Verificando se deu certo:
```sh
# 1.b.2
clickhouse-client -q "DESCRIBE ecommerce.cliente;" --format PrettyCompactMonoBlock

```

Output esperado:
```
   ┌─name──────┬─type───┬─default_type─┬─default_expression─┬─comment─┬─codec_expression─┬─ttl_expression─┐
1. │ id        │ UInt64 │              │                    │         │ NONE             │                │
2. │ nome      │ String │              │                    │         │ NONE             │                │
3. │ data_nasc │ Date32 │              │                    │         │ NONE             │                │
4. │ cpf       │ String │              │                    │         │ NONE             │                │
5. │ email     │ String │              │                    │         │ NONE             │                │
6. │ cidade    │ String │              │                    │         │ NONE             │                │
7. │ uf        │ String │              │                    │         │ NONE             │                │
   └───────────┴────────┴──────────────┴────────────────────┴─────────┴──────────────────┴────────────────┘
```

## 2. Inserindo registros

### 2.a. Três clientes
```sh
# 2.a.1
clickhouse-client -q "
INSERT INTO ecommerce.cliente VALUES
(1, 'Marcelo Barbosa',      '1980-01-15', '111.111.111-11', 'marcelo.barbosa@example.com', 'São Paulo',  'SP'),
(2, 'Juscelino Kubitschek', '1902-09-12', '222.222.222-22', 'juscelino.k@example.com',     'Diamantina', 'MG'),
(3, 'Marivalda Kanamary',   '1975-06-30', '333.333.333-33', 'marivalda.k@example.com',     'Salvador',   'BA');"

```

Verificando:
```sh
# 2.a.2
clickhouse-client -q "SELECT * FROM ecommerce.cliente;" --format PrettyCompactMonoBlock

```

Output:
```
   ┌─id─┬─nome─────────────────┬──data_nasc─┬─cpf────────────┬─email───────────────────────┬─cidade─────┬─uf─┐
1. │  1 │ Marcelo Barbosa      │ 1980-01-15 │ 111.111.111-11 │ marcelo.barbosa@example.com │ São Paulo  │ SP │
2. │  2 │ Juscelino Kubitschek │ 1902-09-12 │ 222.222.222-22 │ juscelino.k@example.com     │ Diamantina │ MG │
3. │  3 │ Marivalda Kanamary   │ 1975-06-30 │ 333.333.333-33 │ marivalda.k@example.com     │ Salvador   │ BA │
   └────┴──────────────────────┴────────────┴────────────────┴─────────────────────────────┴────────────┴────┘
```

## 3. Onde estão os dados?

### 3.a. O diretório da part
O ClickHouse grava cada lote de dados inserido em um diretório chamado **part**. O dicionário de dados informa onde ele está:
```sh
# 3.a.1
clickhouse-client -q "
SELECT name, path
FROM system.parts
WHERE database = 'ecommerce' AND table = 'cliente' AND active;" --format Vertical

```

Output (o caminho será diferente na sua máquina):
```
Row 1:
──────
name: all_1_1_0
path: /var/lib/clickhouse/store/cdc/cdc10e9d-1e85-464e-a793-39c69a9e0ec4/all_1_1_0/
```

Para facilitar, vamos guardar esse caminho em uma variável de ambiente:
```sh
# 3.a.2
PART=$(clickhouse-client -q "SELECT path FROM system.parts WHERE database = 'ecommerce' AND table = 'cliente' AND active LIMIT 1")
echo $PART

```

### 3.b. Os arquivos da part
Agora liste os arquivos de dados (`.bin`) da part:
```sh
# 3.b.1
ls -la $PART | grep '\.bin$'

```

Output:
```
-rw-r----- 1 clickhouse clickhouse   53 Oct  8 17:39 cidade.bin
-rw-r----- 1 clickhouse clickhouse   49 Oct  8 17:39 cidade.size.bin
-rw-r----- 1 clickhouse clickhouse   67 Oct  8 17:39 cpf.bin
-rw-r----- 1 clickhouse clickhouse   49 Oct  8 17:39 cpf.size.bin
-rw-r----- 1 clickhouse clickhouse   37 Oct  8 17:39 data_nasc.bin
-rw-r----- 1 clickhouse clickhouse   98 Oct  8 17:39 email.bin
-rw-r----- 1 clickhouse clickhouse   49 Oct  8 17:39 email.size.bin
-rw-r----- 1 clickhouse clickhouse   49 Oct  8 17:39 id.bin
-rw-r----- 1 clickhouse clickhouse   78 Oct  8 17:39 nome.bin
-rw-r----- 1 clickhouse clickhouse   49 Oct  8 17:39 nome.size.bin
-rw-r----- 1 clickhouse clickhouse   31 Oct  8 17:39 uf.bin
-rw-r----- 1 clickhouse clickhouse   49 Oct  8 17:39 uf.size.bin
```

> Perceba que **cada coluna da tabela virou um arquivo**, com o próprio nome da coluna: `id.bin`, `nome.bin`, `cpf.bin`, `uf.bin`...<br>
> As colunas de texto (`String`) têm ainda um arquivo auxiliar `.size.bin`, que guarda o tamanho de cada valor.

Caso tenha curiosidade, liste **todos** os arquivos da part:
```sh
# 3.b.2
ls -la $PART

```

> Além dos `.bin`, há arquivos de *marcas* (`.cmrk2`, que apontam posições dentro dos `.bin`), o índice primário (`primary.cidx`), a contagem de linhas (`count.txt`) e a lista de colunas (`columns.txt`), entre outros.

## 4. Inspecionando os arquivos de dados

### 4.a. A coluna `nome`
```sh
# 4.a.1
hexdump -C $PART/nome.bin

```

Output:
```
00000000  a1 d8 48 1b db 8d f3 ec  eb 5e e4 d9 6a ae 51 61  |..H......^..j.Qa|
00000010  02 3e 00 00 00 35 00 00  00 4d 61 72 63 65 6c 6f  |.>...5...Marcelo|
00000020  20 42 61 72 62 6f 73 61  4a 75 73 63 65 6c 69 6e  | BarbosaJuscelin|
00000030  6f 20 4b 75 62 69 74 73  63 68 65 6b 4d 61 72 69  |o KubitschekMari|
00000040  76 61 6c 64 61 20 4b 61  6e 61 6d 61 72 79        |valda Kanamary|
0000004e
```

> Observe a coluna da direita: **todos os nomes estão lado a lado**, em sequência, e **nenhum outro atributo** (cpf, e-mail, cidade...) aparece neste arquivo. Essa é a essência do armazenamento colunar.
>
> Os primeiros 25 bytes são um cabeçalho do bloco: 16 bytes de *checksum*, 1 byte com o método de compressão (`02` = nenhum) e dois inteiros de 4 bytes com os tamanhos do bloco (`3e` = 62 bytes com cabeçalho, `35` = 53 bytes de dados).

### 4.b. A coluna `cpf`
```sh
# 4.b.1
hexdump -C $PART/cpf.bin

```

Output:
```
00000000  81 f5 5f 3c 5d a5 2a 63  59 a3 8b 6e c1 9a 53 7d  |.._<].*cY..n..S}|
00000010  02 33 00 00 00 2a 00 00  00 31 31 31 2e 31 31 31  |.3...*...111.111|
00000020  2e 31 31 31 2d 31 31 32  32 32 2e 32 32 32 2e 32  |.111-11222.222.2|
00000030  32 32 2d 32 32 33 33 33  2e 33 33 33 2e 33 33 33  |22-22333.333.333|
00000040  2d 33 33                                          |-33|
00000043
```

### 4.c. A coluna `uf`
```sh
# 4.c.1
hexdump -C $PART/uf.bin

```

Output:
```
00000000  40 ef 6c 4e 39 27 2e 84  c0 e0 ef 05 9a aa b2 d6  |@.lN9'..........|
00000010  02 0f 00 00 00 06 00 00  00 53 50 4d 47 42 41     |.........SPMGBA|
0000001f
```

### 4.d. Em qual arquivo está o "Juscelino"?
```sh
# 4.d.1
grep --text -l Juscelino $PART/*.bin

```

Output:
```
/var/lib/clickhouse/store/cdc/cdc10e9d-1e85-464e-a793-39c69a9e0ec4/all_1_1_0//nome.bin
```

> Apenas o arquivo da coluna `nome` contém o valor. Compare com o laboratório de *rowstore*: lá, o `grep` encontrava o registro inteiro (cpf + nome) no mesmo trecho do arquivo `.ibd`.

### 4.e. Colunas numéricas e de data
Colunas numéricas não são gravadas como texto, mas no seu formato binário. A coluna `id` (`UInt64`) ocupa 8 bytes por valor:
```sh
# 4.e.1
hexdump -C $PART/id.bin

```

Output:
```
00000000  39 32 97 f2 20 a1 0f 9e  d5 82 4c 8a 1c 64 a7 74  |92.. .....L..d.t|
00000010  02 21 00 00 00 18 00 00  00 01 00 00 00 00 00 00  |.!..............|
00000020  00 02 00 00 00 00 00 00  00 03 00 00 00 00 00 00  |................|
00000030  00                                                |.|
00000031
```

> Após o cabeçalho, estão os valores `1`, `2` e `3`, cada um com 8 bytes (*little-endian*): `01 00 00 00 00 00 00 00`, `02 00 ...`, `03 00 ...`.<br>
> Da mesma forma, `data_nasc.bin` guarda cada data como um inteiro de 4 bytes (número de dias desde 01/01/1970) e `nome.size.bin` guarda o tamanho de cada nome (`0f` = 15 bytes de "Marcelo Barbosa", `14` = 20 bytes de "Juscelino Kubitschek"...). Experimente:
```sh
# 4.e.2
hexdump -C $PART/nome.size.bin

```

## 5. Parts são imutáveis

### 5.a. Uma part por `INSERT`
O ClickHouse **nunca altera** um arquivo de dados já gravado. Cada `INSERT` gera uma **nova part** (um novo diretório).

Guarde o diretório da tabela em uma variável:
```sh
# 5.a.1
TABELA=$(clickhouse-client -q "SELECT data_paths[1] FROM system.tables WHERE database = 'ecommerce' AND name = 'cliente'")
ls -1 $TABELA

```

Output:
```
all_1_1_0
detached
format_version.txt
```

Insira mais um registro:
```sh
# 5.a.2
clickhouse-client -q "
INSERT INTO ecommerce.cliente VALUES
(4, 'Carmen Miranda', '1909-02-09', '444.444.444-44', 'carmen.m@example.com', 'Rio de Janeiro', 'RJ');"

```

Verificando as parts:
```sh
# 5.a.3
clickhouse-client -q "
SELECT name, rows, active
FROM system.parts
WHERE database = 'ecommerce' AND table = 'cliente';" --format PrettyCompactMonoBlock

```

Output:
```
   ┌─name──────┬─rows─┬─active─┐
1. │ all_1_1_0 │    3 │      1 │
2. │ all_2_2_0 │    1 │      1 │
   └───────────┴──────┴────────┘
```

```sh
# 5.a.4
ls -1 $TABELA

```

Output:
```
all_1_1_0
all_2_2_0
detached
format_version.txt
```

> O novo registro foi gravado em uma nova part, `all_2_2_0`, com seus próprios arquivos `nome.bin`, `cpf.bin`... A part anterior não foi tocada.

### 5.b. Merge
Periodicamente o ClickHouse **funde** (*merge*) parts pequenas em uma part maior, em segundo plano. Podemos forçar esse processo:
```sh
# 5.b.1
clickhouse-client -q "OPTIMIZE TABLE ecommerce.cliente FINAL;"

```

```sh
# 5.b.2
clickhouse-client -q "
SELECT name, rows, active
FROM system.parts
WHERE database = 'ecommerce' AND table = 'cliente';" --format PrettyCompactMonoBlock

```

Output:
```
   ┌─name──────┬─rows─┬─active─┐
1. │ all_1_1_0 │    3 │      0 │
2. │ all_1_2_1 │    4 │      1 │
3. │ all_2_2_0 │    1 │      0 │
   └───────────┴──────┴────────┘
```

> Uma nova part `all_1_2_1` foi criada com as 4 linhas. As parts antigas ficaram inativas (`active = 0`) e serão apagadas do disco automaticamente após alguns minutos.

Inspecione a coluna `nome` da nova part:
```sh
# 5.b.3
PART=$(clickhouse-client -q "SELECT path FROM system.parts WHERE database = 'ecommerce' AND table = 'cliente' AND active LIMIT 1")
hexdump -C $PART/nome.bin

```

Output:
```
00000000  21 dd bf eb 68 8d c1 f3  3c 87 bf 86 db a9 96 d4  |!...h...<.......|
00000010  02 4c 00 00 00 43 00 00  00 4d 61 72 63 65 6c 6f  |.L...C...Marcelo|
00000020  20 42 61 72 62 6f 73 61  4a 75 73 63 65 6c 69 6e  | BarbosaJuscelin|
00000030  6f 20 4b 75 62 69 74 73  63 68 65 6b 4d 61 72 69  |o KubitschekMari|
00000040  76 61 6c 64 61 20 4b 61  6e 61 6d 61 72 79 43 61  |valda KanamaryCa|
00000050  72 6d 65 6e 20 4d 69 72  61 6e 64 61              |rmen Miranda|
0000005c
```

## 6. Carga do dataset de clientes
Vamos limpar a tabela e carregá-la com um dataset de clientes fictícios.

### 6.a. O arquivo `clientes.csv.gz`
O dataset já está na imagem, na pasta `/data`. Checando o arquivo:
```sh
# 6.a.1
gunzip -c /data/clientes.csv.gz | head

```

Output:
```
id;nome;data_nasc;cpf;email;cidade;uf
1;Calebe Pinto;1988-03-01;645.278.301-71;calebe.pinto@hotmail.com;Vila Velha;ES
2;Lorenzo Silveira;1958-06-04;167.259.048-58;lorenzo.silveira@hotmail.com;Picos;PI
3;Henry da Conceição;2003-11-13;685.402.197-94;henry.da.conceicao@live.com;Parnamirim;RN
4;João Pedro Duarte;1995-10-26;354.806.172-90;joao.pedro.duarte@hotmail.com;Maracanaú;CE
5;Asafe Campos;1999-04-09;579.640.821-67;asafe.campos@yahoo.com;Cariacica;ES
6;Milena Moreira;1959-12-15;704.936.158-57;milena.moreira@outlook.com;Caruaru;PE
7;Giovanna Moreira;1977-10-12;289.316.057-30;giovanna.moreira@gmail.com;Tarauacá;AC
8;Clarice Monteiro;1942-07-04;487.602.159-76;clarice.monteiro@hotmail.com;Tarauacá;AC
9;Isabel Ferreira;1967-03-11;802.537.694-00;isabel.ferreira@outlook.com;Guarulhos;SP
```

### 6.b. Carga de dados
Limpe a tabela:
```sh
# 6.b.1
clickhouse-client -q "TRUNCATE TABLE ecommerce.cliente;"

```

O `clickhouse-client` lê o arquivo compactado diretamente (o `;` é o separador de campos):
```sh
# 6.b.2
clickhouse-client --format_csv_delimiter=';' -q "
INSERT INTO ecommerce.cliente
FROM INFILE '/data/clientes.csv.gz'
FORMAT CSVWithNames;"

```

Checando:
```sh
# 6.b.3
clickhouse-client -q "SELECT count() FROM ecommerce.cliente;"

```

Output:
```
14038
```

```sh
# 6.b.4
clickhouse-client -q "SELECT * FROM ecommerce.cliente ORDER BY id LIMIT 5;" --format PrettyCompactMonoBlock

```

Output:
```
   ┌─id─┬─nome───────────────┬──data_nasc─┬─cpf────────────┬─email─────────────────────────┬─cidade─────┬─uf─┐
1. │  1 │ Calebe Pinto       │ 1988-03-01 │ 645.278.301-71 │ calebe.pinto@hotmail.com      │ Vila Velha │ ES │
2. │  2 │ Lorenzo Silveira   │ 1958-06-04 │ 167.259.048-58 │ lorenzo.silveira@hotmail.com  │ Picos      │ PI │
3. │  3 │ Henry da Conceição │ 2003-11-13 │ 685.402.197-94 │ henry.da.conceicao@live.com   │ Parnamirim │ RN │
4. │  4 │ João Pedro Duarte  │ 1995-10-26 │ 354.806.172-90 │ joao.pedro.duarte@hotmail.com │ Maracanaú  │ CE │
5. │  5 │ Asafe Campos       │ 1999-04-09 │ 579.640.821-67 │ asafe.campos@yahoo.com        │ Cariacica  │ ES │
   └────┴────────────────────┴────────────┴────────────────┴───────────────────────────────┴────────────┴────┘
```

### 6.c. Inspecionando os arquivos
```sh
# 6.c.1
PART=$(clickhouse-client -q "SELECT path FROM system.parts WHERE database = 'ecommerce' AND table = 'cliente' AND active LIMIT 1")
ls -la $PART | grep '\.bin$'

```

Output:
```
-rw-r----- 1 clickhouse clickhouse 145705 Oct  8 17:40 cidade.bin
-rw-r----- 1 clickhouse clickhouse 112354 Oct  8 17:40 cidade.size.bin
-rw-r----- 1 clickhouse clickhouse 196372 Oct  8 17:40 cpf.bin
-rw-r----- 1 clickhouse clickhouse 112354 Oct  8 17:40 cpf.size.bin
-rw-r----- 1 clickhouse clickhouse  56177 Oct  8 17:40 data_nasc.bin
-rw-r----- 1 clickhouse clickhouse 355671 Oct  8 17:40 email.bin
-rw-r----- 1 clickhouse clickhouse 112354 Oct  8 17:40 email.size.bin
-rw-r----- 1 clickhouse clickhouse 112354 Oct  8 17:40 id.bin
-rw-r----- 1 clickhouse clickhouse 210989 Oct  8 17:40 nome.bin
-rw-r----- 1 clickhouse clickhouse 112354 Oct  8 17:40 nome.size.bin
-rw-r----- 1 clickhouse clickhouse  28101 Oct  8 17:40 uf.bin
-rw-r----- 1 clickhouse clickhouse 112354 Oct  8 17:40 uf.size.bin
```

> Cada arquivo tem um tamanho proporcional ao conteúdo da sua coluna: `email.bin` é o maior; `uf.bin` (2 caracteres por linha) é o menor.

A coluna `nome`:
```sh
# 6.c.2
hexdump -C $PART/nome.bin | head

```

Output:
```
00000000  93 b4 e0 f8 4e fa 25 16  86 45 f9 cc 56 1c 90 27  |....N.%..E..V..'|
00000010  02 f2 e1 01 00 e9 e1 01  00 43 61 6c 65 62 65 20  |.........Calebe |
00000020  50 69 6e 74 6f 4c 6f 72  65 6e 7a 6f 20 53 69 6c  |PintoLorenzo Sil|
00000030  76 65 69 72 61 48 65 6e  72 79 20 64 61 20 43 6f  |veiraHenry da Co|
00000040  6e 63 65 69 c3 a7 c3 a3  6f 4a 6f c3 a3 6f 20 50  |ncei....oJo..o P|
00000050  65 64 72 6f 20 44 75 61  72 74 65 41 73 61 66 65  |edro DuarteAsafe|
...
```

A coluna `uf`:
```sh
# 6.c.3
hexdump -C $PART/uf.bin | head

```

Output:
```
00000000  68 8a 06 50 24 6d e4 b1  cf b8 f3 90 c2 62 34 e4  |h..P$m.......b4.|
00000010  02 b5 6d 00 00 ac 6d 00  00 45 53 50 49 52 4e 43  |..m...m..ESPIRNC|
00000020  45 45 53 50 45 41 43 41  43 53 50 52 4f 50 49 47  |EESPEACACSPROPIG|
00000030  4f 53 45 4d 47 52 4a 53  50 52 4e 47 4f 4d 54 52  |OSEMGRJSPRNGOMTR|
00000040  4f 53 45 41 50 45 53 45  53 52 52 50 49 53 50 53  |OSEAPESESRRPISPS|
00000050  45 50 42 52 4a 44 46 52  53 50 41 43 45 50 45 41  |EPBRJDFRSPACEPEA|
...
```

> `ES`, `PI`, `RN`, `CE`, `ES`, `PE`, `AC`, `AC`, `SP`... são as UFs dos clientes 1, 2, 3, 4... gravadas em sequência. Repare quantos valores se repetem: é exatamente esse tipo de repetição que a compressão colunar aproveita.

## 7. Compressão
Agora vamos trabalhar com um volume maior de dados e com a **compressão padrão** do ClickHouse ligada.

### 7.a. O arquivo `invoices.csv`
O arquivo de vendas já está na imagem, na pasta `/data`:
```sh
# 7.a.1
ls -la /data/invoices.tar.gz

```

Output:
```
-rw-r--r-- 1 barbosa barbosa 7726488 Oct 16 02:01 /data/invoices.tar.gz
```

Descompacte o arquivo na própria pasta `/data`:
```sh
# 7.a.2
tar -xzf /data/invoices.tar.gz -C /data/

```

Examinando a estrutura do arquivo:
```sh
# 7.a.3
head -3 /data/invoices.csv

```

Output:
```
InvoiceDate,Country,InvoiceNo,StockCode,Description,CustomerID,Quantity,UnitPrice
2010-12-01T08:26:00Z,United Kingdom,536365,85123A,WHITE HANGING HEART T-LIGHT HOLDER,17850,6,2.55
2010-12-01T08:26:00Z,United Kingdom,536365,71053,WHITE METAL LANTERN,17850,6,3.39
```

Número de linhas:
```sh
# 7.a.4
wc -l /data/invoices.csv

```

Output:
```
541910 /data/invoices.csv
```

### 7.b. Tabela `invoices`
Desta vez **sem** `CODEC(NONE)` (ou seja, com a compressão padrão) e **sem** `min_bytes_for_wide_part`:
```sh
# 7.b.1
clickhouse-client -q "
CREATE TABLE ecommerce.invoices (
    InvoiceDate DateTime,
    Country     String,
    InvoiceNo   String,
    StockCode   String,
    Description String,
    CustomerID  String,
    Quantity    Int32,
    UnitPrice   Float64
)
ENGINE = MergeTree
ORDER BY InvoiceDate;"

```

### 7.c. Carga de dados
> O parâmetro `--date_time_input_format=best_effort` permite ler datas no formato `2010-12-01T08:26:00Z`.

```sh
# 7.c.1
clickhouse-client --date_time_input_format=best_effort -q "
INSERT INTO ecommerce.invoices
FROM INFILE '/data/invoices.csv'
FORMAT CSVWithNames;"

```

Checando:
```sh
# 7.c.2
clickhouse-client -q "SELECT count() FROM ecommerce.invoices;"

```

Output:
```
541909
```

```sh
# 7.c.3
clickhouse-client -q "SELECT * FROM ecommerce.invoices LIMIT 5;" --format PrettyCompactMonoBlock

```

Output:
```
   ┌─────────InvoiceDate─┬─Country────────┬─InvoiceNo─┬─StockCode─┬─Description─────────────────────────┬─CustomerID─┬─Quantity─┬─UnitPrice─┐
1. │ 2010-12-01 08:26:00 │ United Kingdom │ 536365    │ 85123A    │ WHITE HANGING HEART T-LIGHT HOLDER  │ 17850      │        6 │      2.55 │
2. │ 2010-12-01 08:26:00 │ United Kingdom │ 536365    │ 71053     │ WHITE METAL LANTERN                 │ 17850      │        6 │      3.39 │
3. │ 2010-12-01 08:26:00 │ United Kingdom │ 536365    │ 84406B    │ CREAM CUPID HEARTS COAT HANGER      │ 17850      │        8 │      2.75 │
4. │ 2010-12-01 08:26:00 │ United Kingdom │ 536365    │ 84029G    │ KNITTED UNION FLAG HOT WATER BOTTLE │ 17850      │        6 │      3.39 │
5. │ 2010-12-01 08:26:00 │ United Kingdom │ 536365    │ 84029E    │ RED WOOLLY HOTTIE WHITE HEART.      │ 17850      │        6 │      3.39 │
   └─────────────────────┴────────────────┴───────────┴───────────┴─────────────────────────────────────┴────────────┴──────────┴───────────┘
```

### 7.d. Compressão por coluna
O dicionário de dados informa, para cada coluna, o tamanho bruto (descomprimido) e o tamanho efetivamente gravado em disco:
```sh
# 7.d.1
clickhouse-client -q "
SELECT
    column AS coluna,
    formatReadableSize(sum(column_data_uncompressed_bytes)) AS tamanho_bruto,
    formatReadableSize(sum(column_data_compressed_bytes))   AS tamanho_em_disco,
    round(sum(column_data_uncompressed_bytes) / sum(column_data_compressed_bytes), 1) AS taxa_compressao
FROM system.parts_columns
WHERE database = 'ecommerce' AND table = 'invoices' AND active
GROUP BY column
ORDER BY sum(column_data_compressed_bytes) DESC;" --format PrettyCompactMonoBlock

```

Output:
```
   ┌─coluna──────┬─tamanho_bruto─┬─tamanho_em_disco─┬─taxa_compressao─┐
1. │ Description │ 17.76 MiB     │ 6.33 MiB         │             2.8 │
2. │ StockCode   │ 6.76 MiB      │ 1.88 MiB         │             3.6 │
3. │ UnitPrice   │ 4.13 MiB      │ 1.41 MiB         │             2.9 │
4. │ Quantity    │ 2.07 MiB      │ 1.08 MiB         │             1.9 │
5. │ InvoiceNo   │ 7.24 MiB      │ 231.46 KiB       │              32 │
6. │ CustomerID  │ 6.07 MiB      │ 177.15 KiB       │            35.1 │
7. │ InvoiceDate │ 2.07 MiB      │ 123.79 KiB       │            17.1 │
8. │ Country     │ 11.05 MiB     │ 97.80 KiB        │           115.7 │
   └─────────────┴───────────────┴──────────────────┴─────────────────┘
```

> Observe como a taxa de compressão varia de coluna para coluna:
> - `Country` tem pouquíssimos valores distintos (a maioria é "United Kingdom") e, como os valores repetidos ficam **lado a lado** no arquivo, comprime mais de **100 vezes**.
> - `InvoiceNo`, `CustomerID` e `InvoiceDate` também se repetem em sequência (várias linhas por nota fiscal) e comprimem bastante.
> - `Description` e `UnitPrice` variam muito e comprimem pouco.
>
> Em um armazenamento por linha, valores de colunas diferentes ficam misturados no mesmo bloco, o que dificulta esse ganho.

Tamanho total da tabela:
```sh
# 7.d.2
clickhouse-client -q "
SELECT
    formatReadableSize(sum(data_uncompressed_bytes)) AS tamanho_bruto,
    formatReadableSize(sum(data_compressed_bytes))   AS tamanho_em_disco
FROM system.parts
WHERE database = 'ecommerce' AND table = 'invoices' AND active;" --format PrettyCompactMonoBlock

```

Output:
```
   ┌─tamanho_bruto─┬─tamanho_em_disco─┐
1. │ 57.16 MiB     │ 11.33 MiB        │
   └───────────────┴──────────────────┘
```

### 7.e. Os arquivos comprimidos
```sh
# 7.e.1
PART=$(clickhouse-client -q "SELECT path FROM system.parts WHERE database = 'ecommerce' AND table = 'invoices' AND active LIMIT 1")
ls -la $PART | grep '\.bin$'

```

Output:
```
-rw-r----- 1 clickhouse clickhouse   57816 Oct  8 17:40 Country.bin
-rw-r----- 1 clickhouse clickhouse   42333 Oct  8 17:40 Country.size.bin
-rw-r----- 1 clickhouse clickhouse  144985 Oct  8 17:40 CustomerID.bin
-rw-r----- 1 clickhouse clickhouse   36421 Oct  8 17:40 CustomerID.size.bin
-rw-r----- 1 clickhouse clickhouse 5112370 Oct  8 17:40 Description.bin
-rw-r----- 1 clickhouse clickhouse 1528657 Oct  8 17:40 Description.size.bin
-rw-r----- 1 clickhouse clickhouse  126757 Oct  8 17:40 InvoiceDate.bin
-rw-r----- 1 clickhouse clickhouse  194324 Oct  8 17:40 InvoiceNo.bin
-rw-r----- 1 clickhouse clickhouse   42690 Oct  8 17:40 InvoiceNo.size.bin
-rw-r----- 1 clickhouse clickhouse 1137678 Oct  8 17:40 Quantity.bin
-rw-r----- 1 clickhouse clickhouse 1711673 Oct  8 17:40 StockCode.bin
-rw-r----- 1 clickhouse clickhouse  264497 Oct  8 17:40 StockCode.size.bin
-rw-r----- 1 clickhouse clickhouse 1479802 Oct  8 17:40 UnitPrice.bin
```

> Mesmo sem nenhum ajuste na criação da tabela, o ClickHouse gravou **um arquivo por coluna**: com cerca de 57 MB de dados, a part ultrapassou o limite de 10 MB e foi gravada em formato *wide*.

```sh
# 7.e.2
hexdump -C $PART/Description.bin | head -5

```

Output:
```
00000000  1e 8e 27 5d 18 6c 64 a2  f6 fe a6 5f 07 ca 26 d8  |..'].ld...._..&.|
00000010  82 74 27 01 00 81 50 03  00 f2 13 57 48 49 54 45  |.t'...P....WHITE|
00000020  20 48 41 4e 47 49 4e 47  20 48 45 41 52 54 20 54  | HANGING HEART T|
00000030  2d 4c 49 47 48 54 20 48  4f 4c 44 45 52 22 00 f2  |-LIGHT HOLDER"..|
00000040  09 4d 45 54 41 4c 20 4c  41 4e 54 45 52 4e 43 52  |.METAL LANTERNCR|
```

> O método de compressão agora é `82` (LZ4). O LZ4 mantém trechos inéditos como texto literal e substitui as repetições por referências a trechos anteriores, por isso ainda é possível ler parte das descrições.

## 8. Uma consulta lê apenas as colunas de que precisa
Esta é a principal vantagem do armazenamento colunar para consultas analíticas.

### 8.a. Quatro consultas
Execute as consultas abaixo:

**1. Média de preço** (usa apenas a coluna `UnitPrice`):
```sh
# 8.a.1
clickhouse-client -q "SELECT avg(UnitPrice) FROM ecommerce.invoices;"

```

**2. Consulta analítica** (usa as colunas `StockCode` e `UnitPrice`):
```sh
# 8.a.2
clickhouse-client -q "
SELECT count(DISTINCT StockCode), max(UnitPrice), min(UnitPrice), avg(UnitPrice)
FROM ecommerce.invoices;"

```

**3. Busca de uma nota fiscal** (todas as colunas, mas poucas linhas):
```sh
# 8.a.3
clickhouse-client -q "SELECT * FROM ecommerce.invoices WHERE InvoiceNo = '536365';" --format PrettyCompactMonoBlock

```

**4. Leitura da tabela inteira** (todas as colunas, todas as linhas; o resultado é descartado com `FORMAT Null`):
```sh
# 8.a.4
clickhouse-client -q "SELECT * FROM ecommerce.invoices FORMAT Null;"

```

### 8.b. Quanto cada consulta leu
Consulte o log de consultas do ClickHouse para ver quanto cada uma leu:
```sh
# 8.b.1
clickhouse-client -q "SYSTEM FLUSH LOGS;"
clickhouse-client -q "
SELECT
    query AS consulta,
    formatReadableQuantity(read_rows)                          AS linhas_lidas,
    formatReadableSize(ProfileEvents['ReadCompressedBytes'])   AS lido_do_disco,
    formatReadableSize(read_bytes)                             AS dados_processados
FROM system.query_log
WHERE type = 'QueryFinish'
  AND query_kind = 'Select'
  AND has(tables, 'ecommerce.invoices')
ORDER BY event_time_microseconds DESC
LIMIT 4;" --format Vertical

```

Output:
```
Row 1:
──────
consulta:          SELECT * FROM ecommerce.invoices FORMAT Null;
linhas_lidas:      541.91 thousand
lido_do_disco:     11.33 MiB
dados_processados: 44.23 MiB

Row 2:
──────
consulta:          SELECT * FROM ecommerce.invoices WHERE InvoiceNo = '536365';
linhas_lidas:      541.91 thousand
lido_do_disco:     446.95 KiB
dados_processados: 4.66 MiB

Row 3:
──────
consulta:          SELECT count(DISTINCT StockCode), max(UnitPrice), min(UnitPrice), avg(UnitPrice)
FROM ecommerce.invoices;
linhas_lidas:      541.91 thousand
lido_do_disco:     3.30 MiB
dados_processados: 8.31 MiB

Row 4:
──────
consulta:          SELECT avg(UnitPrice) FROM ecommerce.invoices;
linhas_lidas:      541.91 thousand
lido_do_disco:     1.41 MiB
dados_processados: 4.13 MiB
```

> Compare `lido_do_disco` com a tabela de compressão da seção 7:
> - A média de preço leu **1.41 MiB**: exatamente o tamanho em disco da coluna `UnitPrice`. As outras 7 colunas **nem foram abertas**.
> - A consulta analítica leu **3.30 MiB** ≈ `StockCode` (1.88 MiB) + `UnitPrice` (1.41 MiB).
> - A leitura da tabela inteira leu **11.33 MiB**: o tamanho total da tabela em disco.
> - A busca da nota fiscal leu primeiro a coluna `InvoiceNo` e, depois, apenas os trechos das demais colunas onde o valor foi encontrado.
>
> Todas as consultas percorreram as 541 mil linhas, mas o volume lido do disco foi determinado pelas **colunas** envolvidas. Em um armazenamento por linha, ler uma coluna obriga a ler o registro inteiro.

## Parabéns!
Neste laboratório nós:
1. Criamos uma tabela colunar no ClickHouse e localizamos seus arquivos de dados em disco;
2. Comprovamos que **cada coluna é gravada em um arquivo próprio**, com seus valores lado a lado;
3. Vimos que os arquivos de dados são **imutáveis**: cada `INSERT` gera uma nova part, e parts são fundidas por *merges*;
4. Medimos a **compressão** coluna a coluna;
5. Verificamos que uma consulta lê do disco **apenas as colunas de que precisa**.

## Referências

1. **ClickHouse - MergeTree table engine** (parts, formatos *wide* e *compact*)
   https://clickhouse.com/docs/engines/table-engines/mergetree-family/mergetree

2. **ClickHouse - Column compression codecs**
   https://clickhouse.com/docs/sql-reference/statements/create/table#column_compression_codec

3. **ClickHouse - Why is ClickHouse so fast?** (armazenamento colunar)
   https://clickhouse.com/docs/concepts/why-clickhouse-is-so-fast

4. **Row-Oriented vs. Column-Oriented Storage** (conceito geral)
   https://en.wikipedia.org/wiki/Column-oriented_DBMS

5. **Designing Data-Intensive Applications** — Martin Kleppmann
   Capítulo 3: "Storage and Retrieval" — cobre modelos de armazenamento em linha vs. coluna.

## Apêndice: Execução Local

Fora do horário de aula, é possível executar o mesmo ambiente em qualquer máquina (ou provedor de nuvem) com Docker disponível. A imagem é publicada para `linux/amd64` e `linux/arm64` (inclusive Macs com Apple Silicon).

Baixe o `compose.yaml` deste repositório e inicialize o container:
```
mkdir -p clickhouse-lab && cd clickhouse-lab
curl -sSLO https://raw.githubusercontent.com/infobarbosa/clickhouse-columnstore-demo/main/compose.yaml
docker compose up -d

```

Ou, sem o `compose.yaml`:
```
docker run -d --name clickhouse-lab -p 8080:8080 ghcr.io/infobarbosa/clickhouse-lab-docker-image:latest

```

Acesse `http://localhost:8080` no navegador e siga o tutorial a partir do [Passo 3](#passo-3-acessar-a-ide).

Para encerrar e remover o container (os dados do ClickHouse serão descartados):
```
docker compose down

```
ou
```
docker rm -f clickhouse-lab

```
