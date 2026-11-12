# Relatório de Incidente — SSH Brute Force

## Resumo

Durante a janela de teste do laboratório, foram executados dois ataques de força bruta via SSH contra a máquina `alvo-ubuntu` (192.168.56.104), originados da máquina `kali-atacante` (192.168.56.105), utilizando a ferramenta **Hydra**. Ambos os ataques foram detectados pelo Wazuh através da regra nativa **40112** (nível 12), que correlaciona múltiplas falhas de autenticação seguidas de um login bem-sucedido no mesmo host/usuário.

| Campo | Valor |
|---|---|
| Data do incidente | 21/10/2026 |
| Host afetado | alvo-ubuntu (192.168.56.104) |
| Origem do ataque | kali-atacante (192.168.56.105) |
| Usuário-alvo | `labuser` |
| Ferramenta de ataque | Hydra v9.7 |
| Ferramenta de detecção | Wazuh 4.14.9 (regra 40112) |
| Severidade (rule.level) | 12 |
| Status | Contido em ambiente de laboratório isolado — nenhum sistema real foi comprometido |

## Linha do tempo

### Evento 1 — Primeiro ataque (senha correta no início da wordlist, ~3 segundos)

> Neste teste a senha correta estava logo no início da wordlist. O Hydra executa várias conexões em paralelo (até 16 tarefas), então algumas tentativas incorretas ocorreram junto com o sucesso, o suficiente para a regra 40112 (`rule.frequency` 2) disparar. O número exato de falhas não foi registrado.

| Campo | Valor |
|---|---|
| Hydra (início e fim) | Horário exato não registrado; execução de cerca de 3 segundos, encerrada junto com o login bem-sucedido (estimativa a partir do log do SSH) |
| Login bem-sucedido (full_log, UTC → local UTC-3) | 18:23:37 → 15:23:37 |
| Alerta Wazuh gerado | 15:23:38 |
| srcip | 192.168.56.105 |
| srcport | 50276 |
| dstuser | labuser |
| rule.id | 40112 |
| rule.level | 12 |
| rule.firedtimes | 1 |
| full_log | `Oct 21 18:23:37 labuser sshd[18872]: Accepted password for labuser from 192.168.56.105 port 50276 ssh2` |

> Nota: o timestamp dentro do `full_log` está em UTC (18:23), 3h à frente do horário local de Brasília (15:23) — consistente com o fuso configurado na VM alvo.

### Evento 2 — Segundo ataque (senha correta perto do fim da wordlist, 102 senhas)

| Campo | Valor |
|---|---|
| Hydra iniciado | 15:37:21 |
| Hydra finalizado | 15:37:46 |
| Login bem-sucedido (full_log, UTC → local) | 18:37:43 → 15:37:43 |
| Alerta Wazuh gerado | 15:37:44 |
| srcip | 192.168.56.105 |
| srcport | 37482 |
| dstuser | labuser |
| rule.id | 40112 |
| rule.level | 12 |
| rule.firedtimes | 2 |
| full_log | `Oct 21 18:37:43 labuser sshd[19090]: Accepted password for labuser from 192.168.56.105 port 37482 ssh2` |

## Indicadores de Comprometimento (IOCs)

- **IP de origem:** 192.168.56.105
- **Usuário visado:** labuser
- **Portas de origem usadas:** 50276, 37482 (variam a cada conexão, como esperado em ataques automatizados)
- **Ferramenta identificada:** Hydra (padrão de múltiplas tentativas em curto intervalo de tempo, típico de brute-force automatizado)

## Mapeamento MITRE ATT&CK

| Tática | Técnica |
|---|---|
| Credential Access | T1110 — Brute Force |
| Initial Access / Persistence / Privilege Escalation / Defense Evasion | T1078 — Valid Accounts |

## Mapeamento de Compliance (gerado automaticamente pelo Wazuh)

| Framework | Controles |
|---|---|
| GDPR | IV_35.7.d, IV_32.2 |
| GPG13 | 7.1, 7.8 |
| HIPAA | 164.312.b |
| NIST 800-53 | AU.14, AC.7, SI.4 |
| PCI DSS | 10.2.4, 10.2.5, 11.4 |
| TSC | CC6.1, CC6.8, CC7.2, CC7.3 |

## Análise

O ataque seguiu o padrão clássico de **brute force via SSH** (testes sucessivos de senhas de uma wordlist contra um único usuário): múltiplas tentativas de autenticação com diferentes combinações de usuário/senha em um curto intervalo de tempo, partindo de um único host de origem. No segundo evento, a execução do Hydra (102 combinações login/senha, 1 usuário) durou cerca de 25 segundos (15:37:21 a 15:37:46) e encontrou 1 senha válida, o que gerou o login bem-sucedido às 15:37:43. A saída do Hydra está em `evidence/08-hydra-success-redacted.png` (senha ocultada).

O Wazuh não gerou um alerta para cada tentativa individual de senha incorreta — em vez disso, aplicou a lógica de correlação da regra 40112, que identifica o padrão "múltiplas falhas seguidas de um sucesso" no mesmo contexto (mesmo usuário/host) e consolida isso em um único alerta de alta severidade. Esse comportamento reduz ruído em ambientes de produção, mantendo o foco do analista nos eventos que de fato indicam comprometimento.

## Resposta executada

Por se tratar de ambiente de laboratório isolado, a resposta foi limitada à **análise e documentação** do incidente (sem necessidade de contenção real). As ações recomendadas para um cenário de produção estão descritas em `playbook-response.md`.

## Lições aprendidas

- Senhas fracas (ex.: presentes em wordlists públicas) continuam sendo o vetor de entrada mais simples e mais comum em ataques de força bruta.
- A regra nativa do Wazuh foi suficiente para detectar o ataque sem necessidade de tuning adicional, mas o login bem-sucedido ocorreu às 15:37:43 e o alerta foi gerado às 15:37:44, cerca de 1 segundo depois, o que mostra detecção quase em tempo real (o Hydra só encerrou às 15:37:46).
- Um SOC real precisaria de uma resposta automatizada (bloqueio de IP) para reduzir a janela de exposição após a detecção — ver próximos passos no `README.md`.
