# RCA-001: Falha no Sistema de Gravação Verint

## 📋 Informações do Incidente

| Campo | Detalhes |
|-------|----------|
| **ID** | INC-2024-1205 |
| **Data** | 28/04/2024 |
| **Horário** | 09:15 - 11:45 (2h30min) |
| **Severidade** | Alta |
| **Impacto** | ~3.500 gravações perdidas |
| **Cliente** | Instituição Bancária (anonimizado) |
| **Sistemas Afetados** | Verint Impact 360, Storage NetApp, Active Directory |

---

## 🔴 1. Descrição do Incidente

O sistema de gravação Verint Impact 360 parou de capturar chamadas de voz durante o período da manhã. O sistema continuava operacional (sem alarmes críticos), mas nenhuma nova gravação era armazenada.

### Sintomas Observados
- Dashboard Verint mostrava status "verde" (sem alertas)
- Nenhuma nova gravação aparecia nos últimos 30 minutos
- Espaço em disco aparentemente normal (45% usado)
- Logs do sistema sem erros aparentes

---

## ⏱️ 2. Timeline dos Eventos

| Horário | Evento |
|---------|--------|
| 09:15 | Qualidade reporta impossibilidade de acessar gravações recentes |
| 09:22 | Verificação inicial - últimas gravações às 08:47 |
| 09:30 | Abertura de incidente - sistema aparentemente funcional |
| 09:45 | Analista N3 identifica que serviço está rodando mas não grava |
| 10:10 | Descoberto problema de permissão no storage NetApp |
| 10:25 | Identificada mudança recente no Active Directory |
| 10:50 | Permissões corrigidas no storage |
| 11:15 | Sistema voltou a gravar normalmente |
| 11:45 | Validação completa - incidente resolvido |

---

## 🔍 3. Análise Técnica

### Investigação do Sistema Verint
```powershell
# Verificação dos serviços
Get-Service | Where-Object {$_.Name -like "*Verint*"}

# Resultado: Todos os serviços RUNNING ✅

# Verificação dos logs
Get-EventLog -LogName Application -Source "Verint*" -After (Get-Date).AddHours(-2)

# Resultado: Nenhum erro crítico encontrado
```

### Investigação do Storage
```bash
# Acesso ao NetApp via SSH
ssh admin@netapp-storage.empresa.local

# Verificação de quotas
quota report -volume recordings_vol

# Resultado: Quota OK (45% usado)

# Verificação de permissões CIFS
cifs share show -share-name Verint_Recordings -fields share-properties,dir-umask

# ⚠️ PROBLEMA IDENTIFICADO: Permissões alteradas
```

### Causa Identificada
```
Estrutura de Permissões:
├── Storage NetApp
│   └── Share: Verint_Recordings
│       ├── Antes: EMPRESA\SVC_Verint (Full Control) ✅
│       └── Depois: EMPRESA\SVC_Verint_NEW (Read Only) ❌
│
└── Active Directory
    └── Mudança realizada em 27/04/2024 23:45
        └── Migração de conta de serviço (mal executada)
```

---

## ✅ 4. Causa Raiz

**Causa Primária:**  
Mudança na conta de serviço do Active Directory sem atualização correspondente nas permissões do storage NetApp, resultando em perda de permissão de escrita.

**Causas Contribuintes:**
1. Processo de mudança (Change Management) não incluiu storage nas verificações
2. Monitoramento não detectava falha de escrita (apenas espaço em disco)
3. Falta de alertas quando nenhuma gravação é criada por > 15 minutos
4. Documentação de dependências entre sistemas desatualizada

---

## 🛠️ 5. Ações Corretivas (Implementadas)

### Imediato
```powershell
# 1. Correção de permissões no NetApp
cifs share access-control create -share-name Verint_Recordings `
    -user-or-group EMPRESA\SVC_Verint -permission Full_Control

# 2. Restart dos serviços Verint
Restart-Service -Name "Verint*" -Force

# 3. Validação
Test-Path \\netapp-storage\Verint_Recordings\test_write.txt
# Resultado: True ✅
```

### Curto Prazo
1. **Script de Monitoramento Criado:**
```powershell
# monitor_gravacoes.ps1
$ultimaGravacao = Get-ChildItem "\\storage\Verint_Recordings" | 
    Sort-Object LastWriteTime -Descending | 
    Select-Object -First 1

$minutos = ((Get-Date) - $ultimaGravacao.LastWriteTime).TotalMinutes

if ($minutos -gt 15) {
    Send-MailMessage -To "suporte@empresa.com" `
        -Subject "ALERTA: Sem gravações há $minutos minutos" `
        -Body "Sistema Verint pode estar com problemas de escrita" `
        -SmtpServer "smtp.empresa.com"
}
```

2. **Atualização da Documentação de Change:**
   - Incluído checklist de verificação de storage
   - Adicionado passo de validação de permissões
   - Documentadas todas as dependências do Verint

---

## 🔒 6. Ações Preventivas

| Ação | Responsável | Prazo | Status |
|------|-------------|-------|--------|
| Implementar monitoramento de "última gravação" | Infra | 3 dias | ✅ Concluído |
| Criar alerta se nenhuma gravação em 15 min | Monitoramento | 1 semana | ✅ Concluído |
| Atualizar processo de Change Management | Coordenação | 2 semanas | ✅ Concluído |
| Documentar matriz de dependências Verint | Analista N3 | 1 semana | ✅ Concluído |
| Implementar validação automática pós-mudança | DevOps | 1 mês | 🔄 Em andamento |

---

## 📊 7. Métricas de Impacto

### Gravações Perdidas
- **Total:** ~3.500 chamadas
- **Período:** 08:47 - 11:15 (2h28min)
- **Impacto:** Impossibilidade de auditoria de qualidade
- **Criticidade:** Alta (ambiente bancário - compliance)

### Tempo de Recuperação
- **MTTD** (Mean Time To Detect): 37 minutos
- **MTTR** (Mean Time To Repair): 2h30min
- **RTO** (Recovery Time Objective): 4 horas ✅
- **Meta:** Dentro do SLA

---

## 💡 8. Lições Aprendidas

### O que funcionou bem ✅
- Escalação rápida para N3
- Análise metodológica (serviço → aplicação → storage)
- Correção aplicada sem necessidade de reboot dos servidores
- Documentação detalhada do incidente

### O que pode melhorar 🔄
- Monitoramento deveria ter detectado automaticamente
- Processo de Change não incluía validação de storage
- Falta de teste de escrita após mudanças de AD
- Ausência de alertas proativos

### Recomendações Gerais
1. Todo Change em contas de serviço deve incluir teste de I/O
2. Implementar health check automatizado pós-mudança
3. Criar runbook específico para incidentes de gravação
4. Treinar equipe N2 em troubleshooting de storage

---

## 📎 Anexos

- `anexo-01-ad-change-log.txt` - Log de mudança no Active Directory
- `anexo-02-netapp-permissions-before-after.xlsx` - Comparativo de permissões
- `anexo-03-verint-logs.zip` - Logs completos do Verint
- `anexo-04-monitoring-script.ps1` - Script de monitoramento implementado

---

## ✍️ Informações do Documento

| Campo | Detalhes |
|-------|----------|
| **Autor** | Luciano Prado |
| **Cargo** | Analista de Suporte Sênior |
| **Data de Criação** | 29/04/2024 |
| **Última Atualização** | 05/05/2024 |
| **Revisores** | Coordenação de TI, Compliance |
| **Status** | Aprovado |

---

**Classificação:** Confidencial | **Retenção:** 7 anos (Compliance Bancário)
