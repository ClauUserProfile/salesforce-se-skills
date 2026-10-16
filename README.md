# Salesforce SE Skills — Claude Code

Skills para o time de Solution Engineering da Salesforce Brasil.
Cada skill é um comando `/` que executa diretamente no chat do Claude Code.

## Instalação

```bash
# Clone o repo
git clone https://github.com/claudiobecker/salesforce-se-skills.git

# Copie os skills para sua pasta local do Claude Code
cp salesforce-se-skills/*.md ~/.claude/skills/
```

Pronto. Abra o Claude Code e os comandos já estão disponíveis.

## Atualizar skills

```bash
cd salesforce-se-skills
git pull
cp *.md ~/.claude/skills/
```

---

## Skills disponíveis

### `/loyalty-setup [marca]`

Cria um ambiente completo de **Salesforce Loyalty Management** demo em uma org, do zero, diretamente no chat — sem steps manuais.

**O que cria:**
- Programa de fidelidade, moedas (acúmulo + cashback + resgate), tier group, tiers
- 10 membros fictícios com balances e tiers atribuídos
- Benefits e LoyaltyTierBenefits por tier
- Vouchers emitidos para membros elegíveis
- Quick Promotions
- Process Rules de acúmulo, tier upgrade e engajamento (via metadata deploy)
- 15 transações fictícias nos últimos 90 dias
- **Loyalty Member Portal** (Experience Cloud) com cores e logo do cliente

**Pré-requisitos na org:**
- Loyalty Management habilitado (Setup → Loyalty Management → Enable)
- Experience Cloud habilitado
- `sf` CLI autenticado na org (`sf org login web --alias <alias>`)
- `pip3 install simple_salesforce`

**Como usar:**

```
/loyalty-setup

Marca: Ipiranga
Org: mcn-audience
```

Para resultado mais personalizado:

```
/loyalty-setup

Marca: Natura
Setor: Cosméticos / Beleza
Org: minha-org
Logo: [soltar imagem aqui]

Caso de uso: Demo para CX da Natura mostrando consultoras
acumulando pontos por vendas e resgatando produtos do catálogo.
Tiers por volume de vendas mensal, cashback em crédito para
próximo pedido.

Cores: verde #00A859 primário, dourado #C8A951 ação
Programa: Natura Pontos
Moeda: Pontos (PT)
Tiers: Consultora / Bronze / Ouro / Diamante
```

**Marca pré-configurada:** Ipiranga — "Km de Vantagens" (amarelo + azul, 10 membros, 4 tiers, KMs por abastecimento).

---

## Contribuindo

Criou um skill novo? Abre um PR com:
1. O arquivo `.md` do skill
2. Uma linha no README descrevendo o que faz

---

## Autor

Claudio Becker — Solution Engineer, Salesforce Brasil
