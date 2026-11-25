# 🛡️ AWS GuardDuty Incident Response Prompt — Playbook de Investigação

> **Autor:** Lucas Henrique Abreu da Rocha (`@lcszero44-cell`)  
> **Domínio:** Cloud Security Operations (SecOps) • AWS Incident Response • Prompt Engineering  
> **Padrão:** Zero-Hallucination & Evidence-Bound Triage  
> **Referência:** Desafio Criativo DIO — *Do Alerta à Resposta: Prompt para Investigar um Achado do Amazon GuardDuty*

---

## 📋 Prompt Mestre (Pronto para Execução em Produção)

```text
Você é um Engenheiro Sênior de Resposta a Incidentes em Nuvem (Cloud SecOps / AWS) e vai conduzir a triagem técnica de um achado do Amazon GuardDuty.

### 1. DADOS DO ACHADO (CONSOLE AWS GUARDDUTY)
- Tipo do Achado: UnauthorizedAccess:IAMUser/InstanceCredentialExfiltration.OutsideAWS
- Recurso Afetado: Role IAM `prod-payments-checkout-ec2-role` vinculada a uma instância EC2 de produção do cluster de processamento de pagamentos (VPC `vpc-prod-core`, sub-rede privada com saída via NAT Gateway)
- Severidade: 8.5 (Alta / High)
- Janela Temporal: 25/11/2026, entre 01h18 e 01h47 (UTC-3 / Horário de Brasília)
- Descrição Observada: Credenciais temporárias geradas pelo serviço de metadados da instância (IMDS) foram utilizadas a partir de um endereço IP externo fora da infraestrutura da AWS para executar chamadas de listagem e leitura na API da AWS.

### 2. PÚBLICO-ALVO E DECISÃO ESPERADA
- Quem vai ler sua resposta: Engenheiro de Plataforma / SRE de plantão na madrugada (domina arquitetura AWS e Terraform, mas não é especialista em forense de segurança).
- Decisão esperada ao final: Decidir com segurança se revoga imediatamente as sessões ativas da Role IAM e isola a instância EC2, ou se coleta telemetria adicional antes de executar uma ação que interrompa o fluxo de pagamentos.

### 3. CONTEXTO CONFIRMADO DO AMBIENTE (GROUND TRUTH)
- Função e Criticidade: A instância processa transações financeiras em tempo real (SLA 99.95%). Se a Role IAM tiver suas sessões revogadas ou a instância for isolada sem failover, o checkout de clientes sofre indisponibilidade imediata.
- Padrão Normal de Acesso: Nenhum desenvolvedor ou script externo utiliza credenciais dessa Role fora da AWS. Toda comunicação legítima ocorre exclusivamente de dentro da VPC de produção (`sa-east-1`).
- Controles Confirmados:
  - AWS CloudTrail habilitado com entrega multi-região e validação de integridade de log ativa.
  - VPC Flow Logs ativos na interface de rede (ENI) da instância.
  - IMDSv2 configurado como opcional (IMDSv1 ainda permitido na configuração de lançamento da instância).
- Lacunas Atuais (O que ainda NÃO foi verificado):
  - Ainda não consultei no CloudTrail quais chamadas exatas de API (`EventName`) foram executadas pelo IP externo nem se houve erro (`AccessDenied`) ou sucesso.
  - Ainda não verifiquei se houve execução remota de comandos (SSM Session Manager) ou vulnerabilidade SSRF na aplicação web hospedada na instância.

### 4. ESTRUTURA OBRIGATÓRIA DA RESPOSTA
Entregue a análise rigorosamente nesta ordem, sem misturar fatos com suposições:

1. **Resumo Executivo do Alerta (Linguagem Direta):** O que esse achado significa na prática e por que a severidade é alta.
2. **Matriz de Hipóteses (Da Mais Provável para a Menos Provável):** Classifique cada hipótese com nível de probabilidade e justificativa técnica baseada apenas no contexto acima.
3. **Plano de Verificação Forense (Evidência vs. Serviço AWS):** Para cada hipótese, especifique:
   - Qual evidência **confirma** e qual **descarta**;
   - Onde exatamente buscar (nome exato do serviço AWS, tipo de log, evento CloudTrail ou filtro de consulta).
4. **Plano de Contenção Escalonado (Reversível vs. Irreversível):**
   - Separe explicitamente **Ações Reversíveis de Baixo Impacto** das **Ações Disruptivas / Irreversíveis**.
   - Antes de cada ação, declare em uma linha o impacto operacional no checkout de pagamentos.
5. **Checklist de Lacunas para Encerramento do Incidente:** O que ainda precisa ser confirmado para fechar o caso com segurança.

### 5. GUARDRAILS E REGRAS INVIOLÁVEIS
- **Zero Invenção:** Utilize EXCLUSIVAMENTE os identificadores, horários e configurações fornecidos acima. Nunca invente IDs de instância (`i-xxxx`), endereços IP, números de conta AWS, nomes de buckets ou regiões não informadas.
- **Rotulagem Estrita:** Marque explicitamente com `[HIPÓTESE]` qualquer inferência que não esteja declarada no bloco de contexto.
- **Proibição de Falso Positivo Prematuro:** Nunca classifique o alerta como falso positivo sem indicar qual evidência concreta provaria essa conclusão.
- **Protocolo de Lacuna:** Se faltar qualquer dado essencial para concluir uma etapa ou recomendar uma ação crítica, declare exatamente qual informação está faltando e pergunte antes de assumir.
```

---

## 🏗️ Arquitetura de Engenharia do Prompt (Passos 1, 2 e 3)

Este playbook foi desenhado como um **sistema determinístico de triagem assistida por IA** para ambientes críticos de nuvem. Em vez de um pedido genérico ("analise esse alerta"), o prompt impõe fronteiras rígidas de evidência, persona, telemetria e risco operacional.

### Passo 1 — Definição do Achado, Persona e Decisão Operacional

| Dimensão | Implementação no Prompt | Por que importa em SecOps |
| :--- | :--- | :--- |
| **Persona do Modelo** | Engenheiro Sênior de Resposta a Incidentes em Nuvem (Cloud SecOps / AWS) | Garante vocabulário preciso de IAM, STS, IMDS e forense AWS, evitando conselhos genéricos de segurança. |
| **Público-Alvo** | Engenheiro de Plataforma / SRE de plantão na madrugada | Ajusta a profundidade operacional: traduz conceitos forenses em ações concretas compatíveis com Terraform, VPC e SLOs. |
| **Achado Específico** | `UnauthorizedAccess:IAMUser/InstanceCredentialExfiltration.OutsideAWS` (Sev. 8.5) | Ancora o raciocínio em um vetor real de abuso de credenciais STS emitidas pelo Instance Metadata Service (IMDS). |
| **Decisão Esperada** | Contenção imediata vs. coleta adicional preservando checkout (SLA 99.95%) | Evita que a IA sugira ações cegas (como *terminate instance*) sem ponderar blast radius financeiro. |

---

### Passo 2 — Arquitetura de Contexto, Ordem Forense e Guardrails

#### 1. Separação entre *Ground Truth* e *Lacunas*
A seção `3. CONTEXTO CONFIRMADO DO AMBIENTE` divide explicitamente:
- **Fatos confirmados (*Ground Truth*):** criticidade de pagamentos, baseline de tráfego exclusivo em `sa-east-1`, CloudTrail multi-região com validação de integridade, VPC Flow Logs na ENI e **IMDSv1 habilitado** (`HttpTokens: optional`).
- **Lacunas explícitas:** ausência de consulta prévia a `EventName` no CloudTrail e ausência de verificação de SSRF ou uso de SSM Session Manager.

Essa separação evita o viés mais perigoso em LLMs durante incidentes: **preencher silêncios do operador com eventos fictícios**.

#### 2. Ordem Rígida de Análise (Pipeline de 5 Estágios)
1. **Resumo Executivo:** nivela rapidamente o SRE de plantão sobre o impacto real de credenciais temporárias usadas fora da AWS.
2. **Matriz de Hipóteses Ranqueadas:** obriga priorização técnica (ex.: SSRF explorando IMDSv1 → RCE/shell na aplicação → abuso de SSM/acesso lateral → proxy/ferramenta não documentada).
3. **Plano de Verificação Forense:** mapeia cada hipótese diretamente para serviços e filtros nativos da AWS.
4. **Plano de Contenção Escalonado:** exige declaração prévia de impacto no checkout antes de qualquer comando.
5. **Checklist de Lacunas:** define critérios objetivos de saída para encerramento do incidente (*exit criteria*).

#### 3. Mapeamento de Telemetria AWS (Evidência vs. Fonte)

| Hipótese / Vetor | Serviço / Fonte AWS | O que Buscar (Filtros e Eventos) | Critério de Confirmação / Descarte |
| :--- | :--- | :--- | :--- |
| **Abuso de API via Credenciais Exfiltradas** | **AWS CloudTrail** (Event History / Lake / Athena) | `userIdentity.sessionContext.sessionIssuer.userName = prod-payments-checkout-ec2-role`, janela `01h18–01h47 UTC-3`, `sourceIPAddress` externo, `eventName`, `errorCode` (`AccessDenied` vs. sucesso) | Confirma escopo de leitura/exfiltração; `AccessDenied` generalizado indica enumeração contida por menor privilégio. |
| **Exploração SSRF via IMDSv1** | **Application Load Balancer / WAF / Logs da Aplicação** + **EC2 Metadata Options** | Requisições HTTP com payloads apontando para `169.254.169.254/latest/meta-data/iam/security-credentials/` e métrica CloudWatch `MetadataNoToken > 0` | `MetadataNoToken > 0` correlacionado com requisições anômalas na aplicação confirma vetor IMDSv1/SSRF. |
| **Execução Remota / Acesso Administrativo** | **AWS Systems Manager (SSM)** + **CloudTrail** | Eventos `StartSession`, `SendCommand`, logs de sessão SSM no CloudWatch Logs/S3 | Confirma ou descarta uso interativo autorizado ou comprometimento de principal com permissão SSM. |
| **Exfiltração de Rede / C2** | **VPC Flow Logs** (ENI da instância e NAT Gateway) | Tráfego de saída anômalo na janela `01h18–01h47 UTC-3` (portas incomuns, destinos fora do baseline de pagamentos, volume atípico) | Confirma canal de saída direta da instância ou corrobora extração via resposta HTTP da aplicação. |

#### 4. Guardrails Anti-Alucinação (Regras Invioláveis)
- **Zero Invenção:** proíbe fabricação de `i-xxxxxxxx`, IPs, Account IDs, ARNs completos não fornecidos ou buckets S3.
- **Rotulagem `[HIPÓTESE]`:** separa visualmente inferência técnica de fato observado no console.
- **Anti-Falso Positivo Prematuro:** bloqueia respostas complacentes ("pode ser um teste de desenvolvedor") sem exigência de prova documental em log.
- **Protocolo de Lacuna:** converte falta de telemetria em perguntas objetivas ao operador, em vez de chutes probabilísticos.

---

### Passo 3 — Matriz de Contenção (Reversível vs. Irreversível) e Testes de Estresse

#### Matriz de Contenção Escalonada

| Categoria | Ação Técnica AWS | Impacto no Checkout de Pagamentos | Reversibilidade |
| :--- | :--- | :--- | :--- |
| **Baixo Impacto / Reversível** | Consultar CloudTrail e CloudWatch (`MetadataNoToken`) na janela `01h18–01h47 UTC-3` | Nenhum impacto (somente leitura) | Totalmente segura |
| **Baixo Impacto / Reversível** | Criar snapshot de volumes EBS da instância afetada para preservação forense | Nenhum impacto (snapshot assíncrono) | Reversível / não destrutiva |
| **Baixo Impacto / Reversível** | Provisionar instância substituta saudável com `HttpTokens=required` (IMDSv2 obrigatório) no cluster antes de drenar tráfego | Nenhum impacto se validada no target group antes do drain | Reversível |
| **Médio / Alto Impacto (Controlado)** | Aplicar política *inline* `Deny` condicional por `aws:SourceIp` ou `aws:SourceVpc != vpc-prod-core` na Role `prod-payments-checkout-ec2-role` | Baixo impacto se todo tráfego legítimo sair de `vpc-prod-core` (validar VPC Endpoints antes!) | Reversível (remoção da policy) |
| **Disruptiva / Alto Impacto** | Revogar sessões ativas (`AWSRevokeOlderSessions`) da Role IAM sem failover prévio | **Indisponibilidade imediata do checkout** até emissão/rotação ou failover | Reversível apenas após rotação/recuperação |
| **Disruptiva / Forense** | Isolar ENI em Security Group de quarentena (sem ingress/egress) após drenar conexões do Load Balancer | **Interrupção da instância isolada**; sem impacto global apenas se houver réplica saudável | Reversível (restaurar SG), exige drain prévio |
| **Irreversível (Evitar na Triagem)** | `TerminateInstances` da EC2 comprometida | Perda imediata de capacidade **e destruição de evidências voláteis (RAM/processos)** | **Irreversível** |

#### Bateria de Validação (Testes de Estresse)
A eficácia deste prompt foi validada em três cenários críticos documentados na pasta [`test-cases/`](./test-cases/):
1. **[Cenário 01 — Exfiltração IAM em Produção](./test-cases/01-compromised-iam-production.md):** validação completa do fluxo de triagem sob SLA 99.95% com IMDSv1 ativo.
2. **[Cenário 02 — Uso Ambíguo de Conta Root em Manutenção](./test-cases/02-ambiguous-root-migration.md):** validação do bloqueio a falso positivo prematuro em `Policy:IAMUser/RootCredentialUsage`.
3. **[Cenário 03 — Telemetria Incompleta / Edge Case](./test-cases/03-missing-telemetry-edge-case.md):** teste adversário pressionando a IA por dados inexistentes (país do IP, Instance ID), comprovando o acionamento do *Protocolo de Lacuna*.

---

## 🔄 Template Reutilizável (Adaptação para Outros Achados)

Para reutilizar este prompt em novos alertas do Amazon GuardDuty, substitua apenas os blocos `1` e `3`:
- **Bloco 1:** copie `Finding Type`, `Resource`, `Severity`, `Window (UTC/local)` e `Description` diretamente do JSON do GuardDuty.
- **Bloco 3:** atualize o *Ground Truth* (criticidade/SLA, baseline de rede/identidade, controles confirmados) e liste honestamente tudo o que **ainda não foi checado** em *Lacunas Atuais*.
- **Blocos 4 e 5:** mantenha intactos para preservar a disciplina forense e os guardrails anti-alucinação.
