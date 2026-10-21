# Mini-SOC Lab: SSH Brute Force Detection with Wazuh

Laboratório prático simulando um ambiente de SOC (Security Operations Center) em miniatura, com detecção, análise e resposta a um ataque de força bruta via SSH.

## Objetivo

Demonstrar, de ponta a ponta, o ciclo de vida de um incidente de segurança:
**Ataque → Detecção (SIEM) → Análise → Resposta → Documentação**

Este projeto foi construído como parte de um portfólio em Security Operations Center (SOC).

## Arquitetura

```
┌─────────────────────┐         ┌──────────────────────┐
│   kali-atacante      │  SSH    │    alvo-ubuntu       │
│   192.168.56.105     │────────▶│   192.168.56.104     │
│   (Kali Linux)       │ brute   │   (Ubuntu 22.04)     │
│   Ferramenta: Hydra  │ force   │   Wazuh Agent 4.14.9 │
└─────────────────────┘         └───────────┬──────────┘
                                             │ logs/eventos
                                             ▼
                                 ┌──────────────────────┐
                                 │   wazuh-server        │
                                 │   192.168.56.102      │
                                 │   Wazuh 4.14.9         │
                                 │   (Indexer+Manager+   │
                                 │    Dashboard)          │
                                 └──────────────────────┘
```

**Rede:** VirtualBox, modo Host-only (`192.168.56.0/24`), isolada da rede externa. Adaptador NAT adicional em cada VM apenas para atualizações de pacotes.

## Ambiente

| Componente | Especificação |
|---|---|
| Hipervisor | Oracle VirtualBox |
| wazuh-server | Ubuntu Server 22.04 LTS, 4 vCPUs, 8GB RAM, Wazuh 4.14.9 (all-in-one) |
| alvo-ubuntu | Ubuntu Server 22.04 LTS, 2 vCPUs, 2GB RAM, Wazuh Agent 4.14.9 |
| kali-atacante | Kali Linux (VM oficial), 2 vCPUs, 4GB RAM, Hydra v9.7 |

## O que foi feito

1. **Setup da infraestrutura**: 3 VMs isoladas em rede host-only, simulando um ambiente corporativo mínimo.
2. **Instalação do SIEM**: Wazuh all-in-one (Indexer + Manager + Dashboard) no servidor central.
3. **Deploy do agente**: instalação e registro do Wazuh Agent na máquina alvo, monitorando logs de autenticação (`auth.log` / journald).
4. **Simulação de ataque**: ataque de força bruta via SSH contra o usuário `labuser`, usando Hydra com uma wordlist de 102 senhas.
5. **Detecção**: o Wazuh identificou o padrão de múltiplas falhas de autenticação seguidas de um login bem-sucedido, disparando a regra nativa **40112** (nível 12 - severidade alta).
6. **Análise**: investigação dos eventos no dashboard (Threat Hunting), extração de IOCs (IP de origem, usuário, timestamps).
7. **Mapeamento MITRE ATT&CK**: o próprio Wazuh classificou o incidente em **T1110 (Brute Force)** e **T1078 (Valid Accounts)**.
8. **Documentação**: relatório de incidente e playbook de resposta (ver arquivos neste repositório).

## Principais achados

- O Wazuh **não gera um alerta por tentativa de senha incorreta** — ele correlaciona o padrão temporal (N falhas seguidas de 1 sucesso) e gera um único alerta consolidado. Isso evita ruído excessivo em ambientes de produção.
- A regra 40112 já vem mapeada nativamente para múltiplos frameworks de compliance: GDPR, HIPAA, PCI DSS, NIST 800-53 e TSC — útil para contextos onde o SOC precisa justificar controles de auditoria.
- Um ataque de 102 tentativas em cerca de 25 segundos foi detectado e classificado corretamente sem necessidade de regras customizadas.

## Arquivos deste repositório

- `README.md` — este arquivo
- `incident-report.md` — relatório completo do incidente, com linha do tempo e evidências
- `playbook-response.md` — playbook de resposta a incidentes de brute force SSH
- `evidence/` — screenshots do dashboard Wazuh, detalhes dos dois eventos (regra 40112) e saída do Hydra (senha ocultada)

## Como reproduzir

1. Provisionar 3 VMs (VirtualBox/VMware) conforme especificado acima, em rede host-only isolada.
2. Instalar o Wazuh all-in-one no servidor (script oficial: [documentation.wazuh.com](https://documentation.wazuh.com/current/quickstart.html)).
3. Instalar o Wazuh Agent na máquina alvo e registrá-lo via dashboard (**Agents > Deploy new agent**).
4. No Kali, instalar/confirmar Hydra (`hydra -h`) e preparar uma wordlist.
5. Executar: `hydra -L usuarios.txt -P senhas.txt ssh://<IP-ALVO>`
6. Observar os alertas em **Threat Hunting > Events** no dashboard do Wazuh.

## Próximos passos (extensões futuras)

- [ ] Adicionar máquina Windows ao ambiente, monitorando Event IDs 4624/4625
- [ ] Integrar Nmap ao cenário (reconhecimento antes do ataque)
- [ ] Criar regra de detecção customizada (limiar diferente, alertas por e-mail)
- [ ] Implementar resposta automatizada (active response do Wazuh para bloqueio de IP)
- [ ] Análise de vulnerabilidades com OpenVAS/Nessus Essentials

## Autoria e uso de IA

Laboratório montado e executado por mim. Tive apoio de IA para dúvidas de configuração das VMs (alocação de recursos, armazenamento) e para organizar a documentação, que revisei depois.

---

**Disclaimer:** Este laboratório foi executado inteiramente em ambiente isolado e controlado, de propriedade do autor, para fins exclusivamente educacionais. Nenhum sistema de terceiros foi acessado ou testado.
