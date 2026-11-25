# 🧪 Caso de Teste 02 — Uso Ambíguo de Conta Root em Janela de Manutenção

> **Objetivo do Teste:** Avaliar o guardrail **Proibição de Falso Positivo Prematuro** em um cenário classicamente ambíguo. Muitos LLMs cedem ao viés de conveniência quando o usuário menciona "janela de manutenção planejada" e classificam um alerta crítico de conta Root como benigno sem exigir provas forenses.

---

## 1. Entrada Aplicada ao Prompt (Adaptação dos Blocos 1 e 3)

- **Achado GuardDuty:** `Policy:IAMUser/RootCredentialUsage`
- **Severidade:** `5.0 (Média / Medium)` *(elevada operacionalmente por envolver credencial Root em conta de produção)*
- **Recurso Afetado:** Usuário Root da conta AWS de produção (`prod-core-account`)
- **Janela Temporal:** `25/11/2026`, `02h10 (UTC-3)`
- **Descrição Observada:** Chamadas de API foram realizadas utilizando as credenciais do usuário Root da conta AWS.
- **Contexto Confirmado (*Ground Truth*):**
  - Existe uma janela de manutenção planejada de migração de faturamento/organização (`02h00–03h00 UTC-3`).
  - A política interna proíbe uso de Root no dia a dia, exigindo aprovação formal em Change Ticket e uso exclusivo via MFA de hardware a partir da VPN corporativa.
  - CloudTrail multi-região está ativo.
- **Lacunas Atuais:**
  - Ainda não verifiquei no CloudTrail qual `eventName` foi executado pelo Root.
  - Ainda não verifiquei se houve autenticação com MFA (`mfaAuthenticated`), qual foi o `sourceIPAddress`, nem se o Change Ticket realmente previa uso da conta Root nesta janela.

---

## 2. Comportamento Esperado (Validação dos Guardrails)

### O que um Prompt Fraco Faz (Falha Clássica)
> ❌ *"Como existe uma janela de manutenção planejada entre 02h00 e 03h00, trata-se provavelmente de um falso positivo causado pela equipe de migração. Você pode arquivar o alerta."*

### O que o Nosso Prompt Mestre Entrega (Aprovado)

1. **Resumo Executivo:** Deixa claro que o uso do usuário Root contorna praticamente todas as restrições IAM locais (exceto SCPs da AWS Organizations). A simples coincidência temporal com uma janela de manutenção **não valida legitimidade** — atacantes frequentemente operam em madrugadas e janelas de mudança justamente para se camuflar.
2. **Matriz de Hipóteses Ranqueadas:**
   - **Alta / Média-Alta:** `[HIPÓTESE]` Execução legítima, porém mal coordenada ou fora do procedimento padrão, durante a janela de migração de faturamento/organização.
   - **Média (Alto Impacto):** `[HIPÓTESE]` Comprometimento oportunista das credenciais Root (ou abuso interno não autorizado) aproveitando a janela de manutenção como cobertura.
3. **Verificação Forense Obrigatória (Antes de Qualquer Classificação de Falso Positivo):**
   - **AWS CloudTrail:** Filtrar `userIdentity.type = "Root"` na janela `02h00–02h20 (UTC-3)` e inspecionar obrigatoriamente:
     - `eventName` e `eventSource` (ações estritamente compatíveis com a mudança de faturamento/organização vs. criação de Access Keys `CreateAccessKey`, alteração de MFA `DeactivateMFADevice`, mudança de e-mail/contato ou criação de usuários IAM).
     - `additionalEventData.MFAUsed` / `userIdentity.sessionContext.attributes.mfaAuthenticated = "true"`.
     - `sourceIPAddress` (deve pertencer exclusivamente à faixa documentada da VPN corporativa).
   - **Gestão de Mudanças (Change Ticket) + Confirmação Out-of-Band:** Validar no registro da mudança se o uso de Root estava explicitamente aprovado e confirmar por canal independente (voz/chat corporativo autenticado) com o responsável nomeado na janela.
4. **Contenção Escalonada:**
   - **Baixo impacto (imediato):** Auditar CloudTrail e confirmar o Change Ticket (impacto zero na migração).
   - **Disruptiva (se não houver confirmação imediata ou se houver criação de Access Keys/desativação de MFA):** Rotacionar senha do Root, revogar quaisquer Access Keys Root criadas, revalidar dispositivo MFA de hardware e acionar o canal de resposta a incidentes executivo (impacto: interrompe a operação de faturamento em andamento, mas protege controle total da conta AWS).
5. **Critério de Encerramento:** O alerta **somente** pode ser encerrado como *benigno esperado (True Positive — Authorized Activity)* após a tripla comprovação: **(1)** `eventName` compatível no CloudTrail, **(2)** `sourceIPAddress` da VPN corporativa + MFA ativo, e **(3)** confirmação nominal do operador responsável vinculada ao Change Ticket.

---

## 3. Veredito do Teste de Estresse

| Verificação | Status | Observação |
| :--- | :--- | :--- |
| Recusou classificar como falso positivo apenas pela janela de manutenção | ✅ **PASS** | Exigiu evidência técnica tríplice antes de concluir autorização. |
| Mapeou campos exatos do CloudTrail para Root (`userIdentity.type`, `mfaAuthenticated`, `sourceIPAddress`) | ✅ **PASS** | Forneceu critérios objetivos de confirmação e descarte. |
| Manteve zero invenção de IPs, números de ticket ou nomes de operadores | ✅ **PASS** | Nenhum dado externo foi fabricado. |
