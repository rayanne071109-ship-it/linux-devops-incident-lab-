# 🐧 Linux para DevOps — Investigação e Rollback de Incidente em Produção

> Case prático realizado durante a transição de carreira para TI/DevOps.  
> Baseado no curso **"Linux do Zero para DevOps"** da [Maria Lazara](https://www.youtube.com/@marialazaradev) | [Repositório original](https://github.com/marialazara/linux-essentials)

---

## 📋 Sobre este repositório

Este repositório documenta minha resolução do case prático proposto no curso, onde simulei o papel de um **DevOps Engineer on-call** investigando um incidente real de produção em um sistema legado.

O ambiente foi provisionado via Docker e toda a investigação foi conduzida exclusivamente pelo terminal Linux.

---

## 🎯 O Cenário

**Empresa fictícia:** MariaLazaraCloud (SaaS B2B de faturamento)  
**Data do incidente:** 27/09/2024 — 14h25 UTC  
**Serviço afetado:** `billing-api`  
**Sintoma:** API processando zero transações, dashboard em estado *degraded*

```
MariaLazaraCloud Billing API Server
Iniciando billing-api em background...
billing-api iniciado com PID 8
devops@prod-web-01:/srv/app$
```

---

## 🔍 Diagnóstico — Passo a Passo

### 1. Reconhecimento do sistema
```bash
uname -a
# Confirmar kernel, arquitetura e versão do SO antes de qualquer ação
```

### 2. Verificação de processos
```bash
ps aux
ps aux | grep billing
# billing-api rodando (PID 8, estado S = sleep) — processo vivo, mas travado
```

### 3. Leitura dos logs
```bash
find / -type d -name "billing" 2>/dev/null
cd /var/log/marialazaracloud/billing
tail app.log
```

**Output revelador:**
```
ERROR Data corruption detected in /data/billing/transactions
ERROR Cannot process transactions - data validation failed
```

### 4. Monitoramento em tempo real
```bash
tail -f app.log
# Erros a cada ~5 segundos → problema ativo, não histórico
# Ctrl+C para sair
```

### 5. Investigação da mudança recente (GMUD)
```bash
find / -name "GMUD-*" 2>/dev/null
cd /opt/docs
cat GMUD-CHG-2024-0927-001.txt
```

**Descoberta:** GMUD recente migrou dados de `/opt/seeds` → `/data/billing`. O rollback plan estava documentado.

### 6. Confirmação da configuração atual
```bash
find / -name "billing-config.yml" 2>/dev/null
cat /etc/marialazaracloud/billing-config.yml
# data_directory: "/data/billing" ← aponta para o diretório corrompido
```

### 7. Verificação da integridade dos dados
```bash
cat /data/billing/transactions
# ERROR_MIGRATION_INCOMPLETE_DATA_CORRUPTED_2024 ← causa raiz encontrada!

cat /opt/seeds/transactions
# TXN001,2024-09-27,cust_123,29.99,pending ← dados originais íntegros ✓
```

### 8. Análise do diff de configuração
```bash
diff billing-config.yml billing-config.yml.backup
# 4c4
# < data_directory: "/data/billing"
# ---
# > data_directory: "/opt/seeds"
```

---

## 🔄 Rollback — Execução

```bash
# 1. Backup de segurança antes de editar
cp billing-config.yml billing-config.yml.pre-rollback

# 2. Checar permissões (estava read-only)
ls -la billing-config.yml
# -r--r--r-- 1 devops devops 287 ...

# 3. Liberar escrita
chmod 644 billing-config.yml

# 4. Editar: trocar /data/billing → /opt/seeds
nano billing-config.yml

# 5. Restaurar permissões restritivas
chmod 444 billing-config.yml

# 6. Reiniciar o serviço
ps aux | grep billing
kill 8
/usr/local/bin/start-billing-api.sh
```

---

## ✅ Validação

```bash
cd /var/log/marialazaracloud/billing
tail -f app.log

# INFO Using data directory: /opt/seeds
# INFO Transaction validation completed successfully ← resolvido!
```

**Métricas antes/depois:**
```bash
grep "Data corruption detected" app.log | wc -l     # 15 erros antes
grep "Transaction validation completed" app.log | wc -l  # 8 sucessos após rollback
```

---

## 📚 Comandos utilizados

| Comando | Finalidade |
|---|---|
| `uname -a` | Informações do sistema operacional |
| `ps aux` | Listar processos em execução |
| `grep` + `\|` pipe | Filtrar saída de comandos |
| `find` | Localizar arquivos e diretórios |
| `tail` / `tail -f` | Ler / monitorar logs em tempo real |
| `cat` | Exibir conteúdo de arquivos |
| `diff` | Comparar diferenças entre arquivos |
| `chmod` | Gerenciar permissões de arquivos |
| `cp` | Copiar arquivos (backups) |
| `nano` | Editar arquivos de configuração |
| `kill` | Encerrar processos pelo PID |
| `wc -l` | Contar linhas (métricas de logs) |

---

## 🧠 Lições aprendidas

- **Valide dados após migrações** — a GMUD executou sem checar integridade real dos arquivos copiados
- **Documente rollback plans** — o plano estava na GMUD e foi o que guiou a correção
- **Logs são a caixa preta do sistema** — `tail -f` foi o primeiro indício da causa raiz
- **Troubleshooting é narrativo** — entender *por que* cada comando antes de rodar é mais valioso que decorar sintaxes

---

## 🚀 Como reproduzir

```bash
# Subir o ambiente do laboratório
docker pull marialazaradev/linux-essentials:latest
docker run -it --name devops-investigation marialazaradev/linux-essentials:latest
```

---

## 📎 Créditos

- **Curso:** [Linux do Zero para DevOps](https://www.youtube.com/playlist?list=PLOCRt8ucq6xMxYup9rfmtiSwlVU02usi4)
- **Autora:** [Maria Lazara (@marialazaradev)](https://github.com/marialazara)
- **Repositório base:** [marialazara/linux-essentials](https://github.com/marialazara/linux-essentials)

---

*Documentação escrita como parte da minha transição de carreira para TI/DevOps. 🐧*
