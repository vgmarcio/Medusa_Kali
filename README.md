# 🔐 Simulando um Ataque de Brute Force com Medusa e Kali Linux

![Kali Linux](https://img.shields.io/badge/Kali_Linux-557C94?style=for-the-badge&logo=kali-linux&logoColor=white)
![Medusa](https://img.shields.io/badge/Medusa-2.3-8B0000?style=for-the-badge)
![Metasploitable](https://img.shields.io/badge/Metasploitable_2-1.2-2E3440?style=for-the-badge)

> **Aviso:** todo o conteúdo deste repositório foi produzido em **laboratório próprio, isolado e controlado**, com máquinas virtuais de teste (Metasploitable 2 / DVWA). O objetivo é **aprender a detectar e mitigar** ataques de força bruta. Não utilize estas técnicas em sistemas que você não possui ou não tem autorização escrita para testar — isso é crime (Lei 12.737/2012, art. 154-A do Código Penal).

---

## 📌 Sobre o desafio

Desafio de projeto do bootcamp de Cibersegurança: simular um **ataque de força bruta de senhas** utilizando a ferramenta **Medusa** no **Kali Linux** contra ambientes vulneráveis por projeto, documentando cada etapa, os resultados obtidos e as **medidas de prevenção**.

O Medusa é um *brute-forcer* modular e paralelo: ele tenta autenticar em vários serviços (FTP, SSH, SMB, HTTP, bancos de dados) combinando listas de usuários e de senhas, com várias threads simultâneas.

## 🎯 Objetivos

- [x] Montar um laboratório isolado com atacante (Kali) e alvo vulnerável (Metasploitable 2 + DVWA)
- [x] Enumerar os serviços expostos do alvo com Nmap
- [x] Executar ataques de força bruta com Medusa nos serviços FTP, SMB e formulário web
- [x] Registrar as evidências (prints) de cada etapa
- [x] Propor as mitigações correspondentes para cada serviço

## 🧪 Ambiente do laboratório

| Papel | Sistema | IP | Observação |
|-------|---------|----|------------|
| Atacante | Kali Linux (VM) | `192.168.56.102` | Medusa, Nmap, wordlists |
| Alvo | Metasploitable 2 (VM) | `192.168.56.101` | Serviços vulneráveis por padrão |

- **Rede:** Host-Only (VirtualBox/VMware) — sem exposição à internet
- **Wordlist:** `/usr/share/wordlists/rockyou.txt` (ou uma lista reduzida criada para o teste)
- **Usuário de teste:** `msfadmin` / `admin` (contas padrão do Metasploitable)

> ⚠️ Os IPs acima são os do meu laboratório. Se você reproduzir, ajuste para os seus.

## 🛠️ Ferramentas utilizadas

| Ferramenta | Função |
|------------|--------|
| **Kali Linux** | Distribuição do ambiente atacante |
| **Nmap** | Enumeração de portas e serviços do alvo |
| **Medusa 2.3** | Ataque de força bruta paralelo e modular |
| **Metasploitable 2** | Alvo intencionalmente vulnerável |
| **DVWA** | Aplicação web vulnerável (formulário de login) |

## 🗺️ Topologia

![Topologia do laboratório](images/01-topologia.png)

## 🔍 Etapa 1 — Reconhecimento com Nmap

Antes do ataque, é preciso saber **o que está aberto**:

```bash
nmap -sV -p- 192.168.56.101
```

Resultado resumido (serviços relevantes):

| Porta | Serviço | Versão |
|-------|---------|--------|
| 21/tcp | FTP | vsftpd 2.3.4 |
| 80/tcp | HTTP | Apache httpd 2.2.8 (DVWA) |
| 139/445/tcp | SMB | Samba 3.x |

`-sV` identifica a versão do serviço, o que permite escolher o módulo correto do Medusa e entender o vetor de ataque.

![Enumeração com Nmap](images/02-nmap.png)

## 💥 Etapa 2 — Ataque de força bruta em FTP

```bash
medusa -h 192.168.56.101 -U usuarios.txt -P /usr/share/wordlists/rockyou.txt -M ftp -t 4 -f -v 6
```

**Parâmetros usados:**

| Flag | Significado |
|------|-------------|
| `-h` | host alvo |
| `-U` | arquivo com a lista de usuários |
| `-P` | arquivo com a lista de senhas |
| `-M ftp` | módulo do serviço a atacar |
| `-t 4` | 4 threads simultâneas |
| `-f` | para no primeiro sucesso |
| `-v 6` | nível máximo de detalhe na saída |

O `-e ns` ainda testa dois casos especiais: **`n`** = conta sem senha e **`s`** = senha igual ao nome de usuário.

![Ataque FTP com Medusa](images/03-medusa-ftp.png)

## 💥 Etapa 3 — Ataque contra SMB

```bash
medusa -h 192.168.56.101 -U usuarios.txt -P senhas.txt -M smbnt -t 2 -f
```

O módulo `smbnt` é usado para o serviço SMB (porta 445). Como o SMB é mais sensível a tentativas repetidas, reduzi as threads para não derrubar o serviço no meio do teste.

![Ataque SMB com Medusa](images/04-medusa-smb.png)

## 💥 Etapa 4 — Formulário de login web (DVWA)

Para aplicações web, o módulo é o `web-form`. Os parâmetros vão em pares `chave:valor` no `-m`:

```bash
medusa -h 192.168.56.101 -u admin -P senhas.txt -M web-form \
  -m FORM:"/dvwa/login.php" \
  -m DENY-SIGNAL:"Login failed" \
  -m FORM-DATA:"post?username=&password=&Login=Login"
```

- `FORM` → caminho da página de login
- `FORM-DATA` → método (`post` ou `get`) e os campos do formulário; **o campo de usuário vem primeiro, depois o de senha**
- `DENY-SIGNAL` → trecho que aparece na resposta quando o login **falha** (o que não tiver esse trecho é considerado acerto)

> 💡 Cada aplicação é diferente: rode `medusa -M web-form -q` para ver todos os parâmetros que o módulo aceita e ajuste ao formulário que estiver testando.

![Ataque ao formulário web do DVWA](images/05-dvwa.png)

## 📊 Resultados

| Serviço | Módulo | Resultado |
|---------|--------|-----------|
| FTP | `ftp` | Credenciais fracas descobertas |
| SMB | `smbnt` | Autenticação obtida com conta padrão |
| HTTP (DVWA) | `web-form` | Login obtido por dicionário |

**Conclusão:** em todos os três cenários o fator quebrado foi o **fator humano** — senhas fracas, contas padrão não alteradas e ausência de bloqueio por tentativas. O poder de fogo bruto do Medusa só encontra espaço porque essas três defesas faltam.

## 🛡️ Como se proteger (mitigações)

| Vetor | Mitigação |
|-------|-----------|
| FTP | Desabilitar FTP em texto claro; usar SFTP/FTPS; bloquear após N falhas |
| SMB | Desativar SMBv1; senhas fortes e únicas; restringir portas 139/445 por firewall |
| Formulário web | Rate limiting, CAPTCHA, MFA, bloqueio progressivo por IP/conta |
| Geral | Política de senhas, MFA, monitoramento de logs de autenticação, detecção de *password spraying* |

## 🧠 Aprendizados

1. **Reconhecimento vem antes do ataque.** Saber porta, serviço e versão define módulo e estratégia.
2. **Força bruta é ruidosa.** Gera muitos eventos de log — é exatamente por isso que monitorar autenticação funciona.
3. **A wordlist importa mais que a ferramenta.** `rockyou.txt` só é eficaz contra senhas previsíveis; senhas longas e aleatórias tornam o ataque inviável em tempo hábil.
4. **Toda técnica ofensiva tem um par defensivo.** Documentar a mitigação é parte da entrega, não um extra.

## 📚 Referências

- [Medusa — Kali Linux Tools](https://www.kali.org/tools/medusa/)
- [Medusa man page](https://manpages.ubuntu.com/manpages/jammy/man1/medusa.1.html)
- [Metasploitable 2 — Documentation](https://docs.rapid7.com/metasploit/metasploitable-2/)
- [OWASP — Blocking Brute Force Attacks](https://owasp.org/www-community/controls/Blocking_Brute_Force_Attacks)

---

<p align="center">
Desafio de projeto — Cibersegurança · documentação produzida para fins educacionais.
</p>
