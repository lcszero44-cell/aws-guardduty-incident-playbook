# 🧪 Caso de Teste 01 — Exfiltração de Credenciais IAM em Produção (SSRF / IMDSv1)

> **Objetivo do Teste:** Validar a resposta completa do Prompt Mestre no cenário principal de alta criticidade (pagamentos com SLA 99.95%), verificando aderência à ordem obrigatória de 5 seções, proteção à disponibilidade do negócio e rigor na separação entre fato e `[HIPÓTESE]`.

---

## 1. Entrada Aplicada ao Prompt

- **Achado GuardDuty:** `UnauthorizedAccess:IAMUser/InstanceCredentialExfiltration.OutsideAWS`
- **Severidade:** `8.5 (High)`
- **Recurso:** Role IAM `prod-payments-checkout-ec2-role` vinculada a instância EC2 de produção (`vpc-prod-core`, `sa-east-1`, sub-rede privada via NAT Gateway)
- **Janela Temporal:** `25/11/2026`, `01h18–01h47 (UTC-3)`
- **Fatos Confirmados (*Ground Truth*):**
  - Cluster processa transações financeiras em tempo real (SLA 99.95%).
  - Nenhum uso legítimo da Role existe fora de `vpc-prod-core` (`sa-east-1`).
  - CloudTrail multi-região (com integridade) e VPC Flow Logs ativos.
  - **IMDSv2 opcional (IMDSv1 permitido).**
- **Lacunas Declaradas:** Eventos exatos (`EventName` / `AccessDenied`) ainda não consultados no CloudTrail; SSRF e SSM Session Manager ainda não verificados.

---

## 2. Resposta Esperada (Comportamento Validado do Prompt)

### 1. Resumo Executivo do Alerta (Linguagem Direta)
Credenciais temporárias (STS) emitidas pelo Instance Metadata Service (IMDS) para a Role `prod-payments-checkout-ec2-role` foram capturadas e reutilizadas a partir de um IP fora da infraestrutura AWS entre `01h18` e `01h47 (UTC-3)` para chamadas de listagem e leitura na API da AWS. Como o padrão legítimo desse ambiente é uso 100% interno à `vpc-prod-core` (`sa-east-1`), trata-se de um **forte indicador de comprometimento ativo** com risco direto de descoberta de recursos e exfiltração de dados sensíveis do ambiente de pagamentos. A severidade é alta (`8.5`) porque um atacante externo já detém acesso autenticado à API AWS com as permissões dessa Role.

### 2. Matriz de Hipóteses (Da Mais Provável para a Menos Provável)

| Prioridade | Probabilidade | Hipótese | Justificativa Técnica (Baseada no Contexto) |
| :--- | :--- | :--- | :--- |
| **1** | **Alta** | `[HIPÓTESE]` Exploração de SSRF na aplicação web abusando do IMDSv1 | IMDSv2 é apenas opcional (IMDSv1 permitido), eliminando a barreira do token `PUT` de sessão. Uma falha SSRF simples permite consultar `169.254.169.254` e extrair as credenciais temporárias sem acesso ao sistema operacional. |
| **2** | **Média-Alta** | `[HIPÓTESE]` Execução remota de código (RCE) ou comprometimento da instância EC2 | Um atacante com shell ou execução de código na instância consegue ler diretamente o IMDS ou variáveis/arquivos locais e enviar o token STS para um host externo. |
| **3** | **Baixa-Média** | `[HIPÓTESE]` Uso indevido via AWS Systems Manager (SSM Session Manager) ou acesso administrativo comprometido | O uso de SSM ainda não foi auditado; um principal comprometido com permissão SSM poderia extrair credenciais da instância. |
| **4** | **Baixa** | `[HIPÓTESE]` Ferramenta de debug/monitoramento ou proxy externo não documentado usando credenciais copiadas | Contradiz o baseline confirmado de que nenhum script ou desenvolvedor usa a Role fora da AWS, mas só pode ser 100% descartada após checagem do `sourceIPAddress` e `userAgent` no CloudTrail. |

### 3. Plano de Verificação Forense (Evidência vs. Serviço AWS)

- **Para confirmar/descartar abuso de API e impacto real (prioridade imediata):**
  - **Serviço AWS:** `AWS CloudTrail` (Event History / CloudTrail Lake / Athena).
  - **Filtro:** `userIdentity.sessionContext.sessionIssuer.userName = "prod-payments-checkout-ec2-role"` na janela `25/11/2026 01h18–01h47 (UTC-3)` (equivalente a `04h18–04h47 UTC`), agrupando por `sourceIPAddress`, `userAgent`, `eventName`, `requestParameters` e `errorCode`.
  - **Confirma gravidade:** Chamadas bem-sucedidas (`errorCode` ausente) de leitura de segredos, bancos, S3 ou enumeração ampla a partir do IP externo.
  - **Reduz blast radius imediato:** Chamadas restritas a `AccessDenied` (ainda indica credencial vazada, mas contém o dano imediato).

- **Para confirmar/descartar SSRF via IMDSv1 (`[HIPÓTESE]` #1):**
  - **Serviços AWS:** `Amazon CloudWatch` (métrica EC2 `MetadataNoToken`) + logs do Load Balancer/WAF/aplicação.
  - **O que buscar:** Picos em `MetadataNoToken` imediatamente antes ou durante `01h18–01h47 (UTC-3)` correlacionados a requisições HTTP suspeitas contendo `169.254.169.254`.
  - **Confirma:** `MetadataNoToken > 0` alinhado ao timestamp do alerta + evidência de requisição SSRF nos logs web.
  - **Descarta SSRF puro via IMDSv1:** `MetadataNoToken = 0` durante toda a janela (desloca o foco para RCE/IMDSv2 local ou SSM).

- **Para confirmar/descartar RCE / Canal de Exfiltração (`[HIPÓTESE]` #2):**
  - **Serviço AWS:** `VPC Flow Logs` (ENI da instância afetada e NAT Gateway da `vpc-prod-core`).
  - **O que buscar:** Fluxos de saída (`egress`) incomuns na janela `01h18–01h47 (UTC-3)` para IPs externos não pertencentes a gateways de pagamento ou endpoints conhecidos.
  - **Confirma:** Conexões outbound suspeitas iniciadas pela instância imediatamente antes do uso das credenciais fora da AWS.

- **Para confirmar/descartar acesso via SSM (`[HIPÓTESE]` #3):**
  - **Serviços AWS:** `AWS CloudTrail` + `AWS Systems Manager (Session Manager)`.
  - **Filtro:** Eventos `StartSession`, `SendCommand`, `ResumeSession` apontando para a instância na janela anterior a `01h18 (UTC-3)`.

### 4. Plano de Contenção Escalonado (Reversível vs. Irreversível)

#### Ações Reversíveis de Baixo Impacto (Executar Primeiro — 5 a 10 minutos)
1. **Auditar CloudTrail e métrica `MetadataNoToken`**
   - *Impacto no checkout:* Zero impacto operacional (somente leitura).
2. **Criar snapshot forense dos volumes EBS da instância afetada**
   - *Impacto no checkout:* Zero impacto operacional (processo assíncrono em background).
3. **Subir instância substituta saudável no cluster com `HttpTokens=required` (IMDSv2 obrigatório)**
   - *Impacto no checkout:* Zero impacto operacional; prepara o failover seguro para manter o SLA de 99.95%.
4. **Aplicar `Deny` explícito condicional na Role `prod-payments-checkout-ec2-role` bloqueando chamadas de origem externa** (condicionado ao `aws:SourceVpc` ou `aws:SourceIp` do NAT Gateway, **somente após validar o plano de rede/VPC Endpoints**)
   - *Impacto no checkout:* Baixo risco se todo o tráfego legítimo sair via VPC Endpoints/NAT validados; **alto risco de quebra** se algum serviço AWS chamado pela aplicação não passar pela condição configurada.

#### Ações Disruptivas / Irreversíveis (Executar Somente Após Failover ou Confirmação de Exfiltração Ativa)
1. **Revogar sessões ativas da Role (`AWSRevokeOlderSessions`)**
   - *Impacto no checkout:* **Indisponibilidade imediata** das transações na instância afetada (e em outras instâncias que compartilhem a mesma Role) até renovação/failover.
2. **Drenar tráfego no Load Balancer e mover a ENI para Security Group de quarentena (sem ingress/egress)**
   - *Impacto no checkout:* **Queda imediata da instância isolada**; impacto mitigado no cliente apenas se houver capacidade saudável no cluster após failover.
3. **Encerrar (`Terminate`) a instância EC2 — NÃO RECOMENDADO na triagem**
   - *Impacto no checkout:* Redução permanente de capacidade **e destruição irreversível de evidências voláteis em memória (RAM)**.

### 5. Checklist de Lacunas para Encerramento do Incidente
- [ ] Lista completa de `EventName`, `requestParameters` e `errorCode` executados pelo IP externo no CloudTrail.
- [ ] Identificação do IP externo exato e correlação com o detalhe JSON do achado no GuardDuty.
- [ ] Verificação da métrica `MetadataNoToken` e logs da aplicação/WAF para confirmar ou descartar SSRF.
- [ ] Auditoria de eventos `StartSession` / `SendCommand` no SSM.
- [ ] Confirmação de que nenhuma persistência foi criada (novas Access Keys, alteração de Trust Policy, novas Roles ou usuários IAM).
- [ ] Remediação estrutural via Terraform: habilitar `HttpTokens = required` (IMDSv2 obrigatório) no Launch Template e revisar menor privilégio da Role `prod-payments-checkout-ec2-role`.

---

## 3. Critérios de Aprovação do Teste

| Critério de Guardrail | Resultado | Evidência no Output |
| :--- | :--- | :--- |
| Respeitou a ordem exata das 5 seções | ✅ **PASS** | Seções 1 a 5 entregues sequencialmente sem mistura de blocos. |
| Zero invenção de IDs (`i-xxxx`), IPs ou Account IDs | ✅ **PASS** | Nenhum identificador fictício foi inserido. |
| Rotulou inferências com `[HIPÓTESE]` | ✅ **PASS** | Todas as 4 hipóteses explicitamente marcadas. |
| Declarou impacto no checkout antes de cada contenção | ✅ **PASS** | Linha explícita de impacto antes de cada ação reversível e disruptiva. |
| Proibiu descarte prematuro como falso positivo | ✅ **PASS** | Tratou como incidente real até prova forense contrária no CloudTrail. |
