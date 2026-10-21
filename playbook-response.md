# Playbook de Resposta a Incidentes — SSH Brute Force

Playbook de referência para resposta a alertas gerados pela regra **40112** do Wazuh (SSH brute force com sucesso), baseado no incidente documentado em `incident-report.md`.

## 1. Detecção

**Gatilho:** alerta no Wazuh com `rule.id: 40112`, `rule.level: 12`, indicando múltiplas falhas de autenticação SSH seguidas de um login bem-sucedido no mesmo host/usuário.

**Onde observar:**
- Dashboard Wazuh → **Threat Hunting** → filtrar por `rule.id: 40112`
- Campos relevantes no evento: `data.srcip`, `data.dstuser`, `data.srcport`, `full_log`, `rule.firedtimes`

**Ação imediata:** analista confirma que o alerta é real (não falso positivo) verificando se o usuário em questão de fato realizou o login naquele horário.

## 2. Análise

Perguntas a responder antes de agir:

- [ ] O IP de origem (`data.srcip`) é conhecido/esperado (ex.: VPN corporativa) ou desconhecido/externo?
- [ ] O usuário-alvo (`data.dstuser`) confirma ter feito login no horário do alerta?
- [ ] Quantas tentativas ocorreram (`rule.firedtimes`) e em qual intervalo de tempo? Um volume alto em poucos segundos é característico de automação (ex.: Hydra).
- [ ] Há outros alertas associados ao mesmo IP de origem em outros hosts (indicando varredura lateral)?
- [ ] A conta comprometida tem privilégios elevados (sudo, acesso a dados sensíveis)?

**No incidente de referência:** IP de origem 192.168.56.105, usuário `labuser`, 102 tentativas em ~25 segundos, sem confirmação do usuário real — tratado como comprometimento confirmado.

## 3. Contenção

Ações para interromper o acesso do atacante, da mais rápida para a mais abrangente:

1. **Bloquear o IP de origem** na máquina afetada:
   ```
   sudo ufw deny from 192.168.56.105 to any port 22
   ```
   ou, via iptables:
   ```
   sudo iptables -A INPUT -s 192.168.56.105 -p tcp --dport 22 -j DROP
   ```
2. **Desabilitar ou trocar a senha da conta comprometida** imediatamente:
   ```
   sudo passwd labuser
   ```
3. **Encerrar sessões ativas** do usuário, caso existam:
   ```
   sudo pkill -KILL -u labuser
   ```
4. Em ambiente de produção: acionar bloqueio no firewall perimetral/próximo salto (não só no host), para impedir novas tentativas de outros vetores.

## 4. Erradicação

- Revisar se a conta comprometida foi usada para criar outros usuários, chaves SSH (`~/.ssh/authorized_keys`) ou tarefas agendadas (`crontab -l`, `/etc/cron.*`) não autorizadas.
- Verificar histórico de comandos (`~/.bash_history`) do usuário no período do incidente.
- Confirmar que nenhum pacote/binário malicioso foi instalado.

## 5. Recuperação

- Restaurar acesso normal apenas após: senha forte redefinida, regra de bloqueio de IP mantida ou IP confirmado como não mais ativo, e MFA habilitado na conta (se disponível).
- Reabilitar o usuário e monitorar de perto por 24–48h (alertas adicionais do Wazuh para o mesmo host/usuário).

## 6. Lições aprendidas / ações preventivas

- [ ] **Política de senha forte** — exigir senhas que não constem em wordlists públicas (mínimo de complexidade e tamanho).
- [ ] **Autenticação por chave SSH** em vez de senha, desabilitando `PasswordAuthentication` no `sshd_config`.
- [ ] **Fail2ban ou active response do Wazuh** para bloqueio automático de IP após N tentativas falhas, reduzindo a janela de exposição.
- [ ] **MFA** para contas com acesso remoto.
- [ ] **Alerta em tempo real** (e-mail/Slack) configurado para qualquer disparo da regra 40112, não apenas visualização no dashboard.
- [ ] Revisar se a porta SSH precisa estar exposta à rede mais ampla ou se pode ficar restrita a uma VPN/bastion host.

## Referência rápida — comandos de resposta

| Ação | Comando |
|---|---|
| Bloquear IP (ufw) | `sudo ufw deny from <IP> to any port 22` |
| Bloquear IP (iptables) | `sudo iptables -A INPUT -s <IP> -p tcp --dport 22 -j DROP` |
| Trocar senha do usuário | `sudo passwd <usuario>` |
| Encerrar sessões do usuário | `sudo pkill -KILL -u <usuario>` |
| Ver tentativas de login (journald) | `sudo journalctl -u ssh --no-pager \| grep -E "Failed password\|Accepted password"` |
| Ver chaves SSH autorizadas | `cat ~<usuario>/.ssh/authorized_keys` |
