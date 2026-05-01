# 🔐 SSH Hardening & Monitoring Lab

**Ambiente:** VirtualBox | Ubuntu Server 24.04.4 LTS + Kali Linux  
**Objetivo:** Simular um ataque de força bruta SSH, analisar os logs como um analista SOC e aplicar hardening progressivo para mitigar o ataque.

---

## 🧱 Ambiente

| Máquina | Sistema | IP | Função |
|---|---|---|---|
| Ubuntu Server | Ubuntu 24.04.4 LTS | 192.168.1.100 | Alvo / Servidor |
| Kali Linux | Kali 2025.4 | 192.168.1.13 | Atacante / Cliente |

Rede: Bridge Adapter (ambiente temporário de treino)

---

## 📋 Fases do Projeto

### Fase 1 — Configuração do Ambiente
- Instalação do Ubuntu Server 24.04.4
- Configuração de IP fixo via Netplan (`/etc/netplan/50-cloud-init.yaml`)
- Verificação e ativação do serviço OpenSSH
- Conexão SSH bem-sucedida do Kali para o servidor

**⚠️ Lição aprendida:** O Ubuntu 24.04 usa ativação por socket (`ssh.socket`) por padrão. O `ssh.service` aparece como `inactive` mas funciona normalmente quando ativado via socket.

---

### Fase 2 — Simulação de Ataque de Força Bruta
- Criação de usuário `vitima` com senha fraca (`123456`)
- Monitoramento de logs em tempo real com `tail -f /var/log/auth.log`
- Ataque com Hydra usando a wordlist `rockyou.txt`

**Comando utilizado:**
```bash
hydra -l vitima -P /usr/share/wordlists/rockyou.txt 192.168.1.100 ssh -t 4 -V
```

**Resultado:** Senha `123456` encontrada na 4ª tentativa em menos de 1 minuto.

**IOCs identificados nos logs:**
- IP do atacante repetido: `192.168.1.13`
- Múltiplas linhas `Failed password` em sequência
- Portas de origem variando (comportamento de automação)
- Sequência: falhas → `Accepted password` → `session opened`

---

### Fase 3 — Hardening Progressivo

#### Medida 1 — Troca de Porta Padrão (22 → 2222)
Editado `/etc/ssh/sshd_config` e desativado o `ssh.socket` para que o `ssh.service` assuma controle total da porta.

```bash
sudo systemctl disable --now ssh.socket
sudo systemctl enable --now ssh.service
```

**⚠️ Lição aprendida:** No Ubuntu 24.04, o `ssh.socket` mantém a porta 22 ativa mesmo após alterar o `sshd_config`. É necessário desativá-lo explicitamente.

#### Medida 2 — Desabilitar Login do Root
Configurado `PermitRootLogin no` no sshd_config.

**⚠️ Lição aprendida:** O arquivo `/etc/ssh/sshd_config.d/50-cloud-init.conf` continha `PermitRootLogin yes` e sobrescrevia silenciosamente o `sshd_config`. Sempre verificar o diretório `sshd_config.d/` após qualquer alteração.

#### Medida 3 — Autenticação por Chave (Desabilitar Senha)
```bash
# Geração do par de chaves no Kali
ssh-keygen -t ed25519 -C "kali-lab"

# Cópia da chave pública para o servidor
ssh-copy-id -i ~/.ssh/id_ed25519.pub -p 2222 lab@192.168.1.100
```

Após confirmar o login por chave, `PasswordAuthentication no` foi definido em `/etc/ssh/sshd_config.d/50-cloud-init.conf`.

**Resultado:** Hydra falha imediatamente — o servidor não aceita autenticação por senha independente da wordlist utilizada.

#### Medida 4 — Fail2ban
Configurado `/etc/fail2ban/jail.local` com regras para SSH:

```ini
[DEFAULT]
banaction = nftables
banaction_allports = nftables[type=allports]
backend = systemd

[sshd]
enabled = true
port    = 2222
maxretry = 3
bantime = 600
findtime = 60
```

**Resultado:** Após 3 tentativas falhas, o IP `192.168.1.13` foi automaticamente banado por 10 minutos.

**⚠️ Lição aprendida:** Ao copiar o `jail.conf` para `jail.local`, o arquivo gerado é extenso e propenso a erros de edição (seções duplicadas, linhas comentadas acidentalmente). A abordagem mais segura é criar um `jail.local` mínimo do zero contendo apenas as configurações necessárias.

---

### Fase 4 — SSH Tunneling

#### Túnel Local
Acesso ao nginx interno do servidor (porta 80) através da porta 8080 do Kali:
```bash
ssh -p 2222 -L 8080:localhost:80 lab@192.168.1.100
curl http://localhost:8080  # Retorna o HTML do nginx
```
**Caso de uso ofensivo:** Atacante com acesso SSH acessa serviços internos não expostos externamente (banco de dados, painéis admin).

#### Túnel Remoto
Servidor acessa serviço rodando no Kali através de túnel reverso:
```bash
# No Kali — servidor web simples
python3 -m http.server 9000

# Criação do túnel reverso
ssh -p 2222 -R 7070:localhost:9000 lab@192.168.1.100

# No servidor Ubuntu
curl http://localhost:7070  # Retorna conteúdo do Kali
```
**Caso de uso ofensivo:** Mecanismo de reverse shell / canal C2 — tráfego parece SSH legítimo saindo da rede interna.

#### Dynamic SOCKS5 Proxy
Todo o tráfego do Kali roteado pelo servidor:
```bash
ssh -p 2222 -D 1080 lab@192.168.1.100
curl --socks5 localhost:1080 http://ifconfig.me
# Retornou o IP público da rede do servidor
```
**Caso de uso ofensivo:** Anonimização de tráfego e pivoting — requisições suspeitas aparecem nos logs com o IP do servidor comprometido, não do atacante real.

---

## 🛡️ Resumo de Hardening Aplicado

| Medida | Configuração | Impacto |
|---|---|---|
| Porta customizada | `Port 2222` | Elimina scanners automáticos na 22 |
| Root bloqueado | `PermitRootLogin no` | Remove o alvo mais valioso |
| Autenticação por chave | `PasswordAuthentication no` | Força bruta se torna inútil |
| Fail2ban | `maxretry 3`, `bantime 600` | Ban automático de IPs suspeitos |

---

## 🔑 Comandos de Referência Rápida

```bash
# Status dos serviços
sudo systemctl status ssh.service
sudo systemctl status fail2ban

# Monitoramento de logs
sudo tail -f /var/log/auth.log
sudo tail -f /var/log/fail2ban.log
sudo journalctl -u ssh -f

# Fail2ban
sudo fail2ban-client status sshd
sudo fail2ban-client set sshd unbanip <IP>

# Verificar porta ativa
ss -tlnp | grep 2222
```

---

## 📁 Evidências

As evidências de cada etapa estão na pasta `/evidencias-lab-ubuntu-ssh/`.

---

## 🧰 Ferramentas Utilizadas

- **OpenSSH** — servidor e cliente SSH
- **Hydra** — brute force de credenciais
- **fail2ban** — detecção e bloqueio automático
- **Netplan** — configuração de rede no Ubuntu
- **rockyou.txt** — wordlist de senhas

---

*Lab realizado como parte da construção de portfólio para vaga de Junior SOC Analyst / Blue Team.*
