---
name: loyalty-setup
description: Creates and deploys a complete Salesforce Loyalty Management demo environment directly into a target org — including data (program, tiers, currencies, members, benefits, vouchers, process rules) AND a branded Loyalty Member Portal (Experience Cloud) with the client's colors and logo.
triggers:
  - /loyalty-setup
---

# Loyalty Management Demo Setup Skill

## What This Skill Does

Generates and EXECUTES a complete Loyalty Management demo setup directly into a Salesforce org. No manual steps — records are created in the org during the chat session.

## Invocation

`/loyalty-setup [brand]`

If brand is omitted, ask the user which brand to use.

---

## Execution Flow

### Step 1 — List available orgs

Run:
```bash
sf org list --json
```

Show the user a clean list of available orgs (alias + instanceUrl + type). Ask which org to use for the demo setup.

### Step 2 — Verify org connection

Run:
```bash
sf org display --target-org {alias} --json
```

Extract `instanceUrl` and `accessToken`. If the org is expired or not connected, tell the user to run `sf org login web --alias {alias}` and retry.

### Step 3 — Check dependencies

```bash
python3 -c "import simple_salesforce" 2>/dev/null || pip3 install simple_salesforce --quiet
```

### Step 4 — Execute setup in phases

Run these phases in sequence, capturing IDs for use in subsequent phases:

**Phase 1 — Core structure** (LoyaltyProgram, Currencies, TierGroup, Tiers, Contacts, Members)  
**Phase 2 — Transactions & Processes** (LoyaltyProgramProcess via API, Process Rules via metadata deploy)  
**Phase 3 — Promotions & Vouchers** (Promotions, VoucherDefinitions, Vouchers)  
**Phase 4 — Benefits** (BenefitTypes, Benefits, LoyaltyTierBenefits, MemberCurrency)  
**Phase 5 — Loyalty Member Portal** (create Experience Cloud site, upload logo, apply brand colors)

### Step 5 — Confirm and summarize

After execution, show a summary table:
```
=== Loyalty Setup Complete: {Brand} ===

✓ LoyaltyProgram            1    → {ProgramName}
✓ LoyaltyProgramCurrency    3    → KM (Qualifying), Cashback R$, KM Resgate
✓ LoyaltyTierGroup          1    → Categorias KM
✓ LoyaltyTier               4    → Bronze, Prata, Ouro, Diamante
✓ Contact                  10
✓ LoyaltyProgramMember     10
✓ LoyaltyMemberCurrency    10
✓ LoyaltyMemberTier        10
✓ LoyaltyProgramProcess     3    → Accrual, TierProcessing, CreditPoints
✓ LoyaltyProgramProcessRule 7    → via metadata deploy
✓ Promotion                 3    → Quick promotions
✓ VoucherDefinition         3
✓ Voucher                  11
✓ BenefitType               3
✓ Benefit                  14
✓ LoyaltyTierBenefit       14
✓ TransactionJournal       15
✓ Loyalty Member Portal     1    → {instanceUrl}/s/{urlPathPrefix}

Org: {instanceUrl}
```

---

## Phase 5 — Loyalty Member Portal

### 5a — Create the Experience Cloud site

```bash
sf community create \
  --name "{ProgramName}" \
  --template-name "Loyalty Member Portal" \
  --url-path-prefix "{urlPathPrefix}" \
  --target-org {alias}
```

Wait ~30 seconds for the site to provision, then get the ExperienceBundle API name:

```bash
sf org list metadata --metadata-type ExperienceBundle --target-org {alias} --json
```

The bundle name will be `{SiteName_without_spaces}1` (e.g. `Km_de_Vantagens1`).

### 5b — Retrieve the ExperienceBundle

```bash
mkdir -p /tmp/loyalty_portal_{brand}
cd /tmp/loyalty_portal_{brand}
cat > sfdx-project.json << 'EOF'
{"packageDirectories":[{"path":"force-app","default":true}],"namespace":"","sourceApiVersion":"68.0"}
EOF
cat > package.xml << 'EOF'
<?xml version="1.0" encoding="UTF-8"?>
<Package xmlns="http://soap.sforce.com/2006/04/metadata">
    <types><members>{BundleName}</members><name>ExperienceBundle</name></types>
    <version>68.0</version>
</Package>
EOF
sf project retrieve start --manifest package.xml --target-org {alias}
```

### 5c — Upload brand logo (if user provides an image file)

```python
import base64, requests

with open(logo_path, 'rb') as f:
    logo_b64 = base64.b64encode(f.read()).decode('utf-8')

resp = requests.post(f"{instance_url}/services/data/v68.0/sobjects/ContentVersion/",
    headers={'Authorization': f'Bearer {access_token}', 'Content-Type': 'application/json'},
    json={
        "Title": f"{brand}_Logo",
        "PathOnClient": f"{brand}_Logo.jpg",
        "VersionData": logo_b64,
        "IsMajorVersion": True
    })
cv_id = resp.json()['id']
doc_resp = requests.get(f"{instance_url}/services/data/v68.0/sobjects/ContentVersion/{cv_id}?fields=ContentDocumentId",
    headers={'Authorization': f'Bearer {access_token}'})
doc_id = doc_resp.json()['ContentDocumentId']
logo_url = f"/sfc/servlet.shepherd/document/download/{doc_id}"
```

If no logo file is provided, set `logo_url = ""` and skip.

### 5d — Apply brand colors via Python

Modify the two retrieved JSON files with the brand palette, then deploy:

```python
import json

# Load brandingSet
path_bs = f'/tmp/loyalty_portal_{brand}/force-app/main/default/experiences/{bundle_name}/brandingSets/loyaltyMemberPortal.json'
with open(path_bs) as f:
    bs = json.load(f)

bs['values'].update({
    "ActionColor"   : brand['action_color'],    # e.g. "rgb(27, 117, 187)"
    "LinkColor"     : brand['action_color'],
    "LoginBackgroundColor": brand['primary_color'],  # e.g. "rgb(255, 182, 0)"
    "CompanyLogo"   : logo_url,
    # ... (full palette — see brand templates below)
})
with open(path_bs, 'w') as f:
    json.dump(bs, f, indent=2)

# Load theme — update header background, hero text/colors, CTA button
path_theme = f'/tmp/loyalty_portal_{brand}/force-app/main/default/experiences/{bundle_name}/themes/loyaltyMemberPortal.json'
with open(path_theme) as f:
    theme = json.load(f)

# Update headerBgColor on all layouts
for view in theme['views']:
    if 'headerBgColor' in view.get('componentAttributes', {}):
        view['componentAttributes']['headerBgColor'] = f"rgba({brand['primary_rgb']}, 0.97)"
    for region in view.get('regions', []):
        for comp in region.get('components', []):
            ca = comp.get('componentAttributes', {})
            # Header
            if comp['componentName'] == 'forceCommunity:themeHeaderCompact':
                ca['background'] = f"background: {brand['primary_color']}"
                ca['linkColor'] = brand['action_color']
                ca['linkHoverColor'] = brand['action_color_dark']
            # Hero
            if comp['componentName'] == 'forceCommunity:themeHeroStarter':
                ca['bgImage'] = f"background: {brand['primary_color']} url(/sfsites/assets/themes/loyalty-home-banner.jpg) top center /cover"
                ca['titleText'] = brand['program_name']
                ca['captionText'] = brand['hero_caption']
                ca['subtitleText'] = brand['hero_subtitle']
                ca['titleTextColor'] = brand['action_color']
                ca['ctaBtnOneBgColor'] = brand['action_color']
                ca['ctaBtnOneText'] = brand['hero_cta_text']
                ca['ctaBtnOneTextColor'] = "rgb(255, 255, 255)"
                ca['ctaShowBtnOne'] = True
                ca['showSubtitle'] = True

theme['customCSS'] = brand['custom_css']
with open(path_theme, 'w') as f:
    json.dump(theme, f, indent=2)
```

### 5e — Deploy

```bash
cd /tmp/loyalty_portal_{brand}
cat > deploy_package.xml << 'EOF'
<?xml version="1.0" encoding="UTF-8"?>
<Package xmlns="http://soap.sforce.com/2006/04/metadata">
    <types><members>{BundleName}</members><name>ExperienceBundle</name></types>
    <version>68.0</version>
</Package>
EOF
sf project deploy start --manifest deploy_package.xml --target-org {alias}
```

---

## Rollback

If invoked as `/loyalty-setup rollback {brand} --org {alias}`:

1. Read `/tmp/loyalty_ids_{brand}.json`
2. Generate and execute deletion script (reverse order: child before parent)
3. Confirm each object type was deleted

---

## CRITICAL: Correct Field Names and API Patterns

These are verified against API v68.0. Wrong field names are the most common failure.

### Authentication
```python
from simple_salesforce import Salesforce
import subprocess, json

r = subprocess.run(['sf', 'org', 'display', '--target-org', ALIAS, '--json'],
                   capture_output=True, text=True)
info = json.loads(r.stdout)['result']
sf = Salesforce(instance_url=info['instanceUrl'],
                session_id=info['accessToken'],
                version='68.0')   # ← REQUIRED: always specify v68.0
```

### LoyaltyProgram
```python
sf.LoyaltyProgram.create({
    'Name': 'Km de Vantagens',
    'Status': 'Active',
    'Description': '...',
})
# ProgramType is NOT a valid field via REST API
```

### LoyaltyProgramCurrency
```python
sf.LoyaltyProgramCurrency.create({
    'Name': 'KM',
    'CurrencyCode': 'KM',
    'Status': 'Active',
    'LoyaltyProgramId': program_id,
    'UnitOfMeasure': 'Points',
    'CurrencyType': 'Qualifying',
    # NO Description field
    # NO ExpiryModel on Qualifying currencies
})
# Non-qualifying (e.g. Redeemable) can have ExpiryModel, ExpiryPeriodFrequency, ExpiryPeriodUnit
```

### LoyaltyTierGroup
```python
sf.LoyaltyTierGroup.create({
    'Name': 'Categorias KM',
    'LoyaltyProgramId': program_id,
    'TierModel': 'Anniversary',        # NOT PeriodType
    'QpResetPeriod': 'Years',          # NOT TierPeriodUnit
    'IsPrimary': True,
    'Category': 'TIER_GROUP',
    # DO NOT include QpResetDate when using Anniversary model
    # DO NOT include TierPeriodQuantity
})
```

### LoyaltyTier
```python
sf.LoyaltyTier.create({
    'Name': 'Bronze',
    'LoyaltyTierGroupId': tier_group_id,
    'MinimumEligibleBalance': 0,       # NOT LowerBound
    'MaximumEligibleBalance': 500,     # NOT UpperBound
    'SequenceNumber': 1,               # NOT TierSequence
})
```

### LoyaltyProgramMember
```python
sf.LoyaltyProgramMember.create({
    'ProgramId': program_id,           # NOT LoyaltyProgramId
    'ContactId': contact_id,
    'MemberType': 'Individual',        # 'Corporate' requires AccountId
    'MembershipNumber': 'KM100001',
    'MemberStatus': 'Active',
    'EnrollmentDate': '2024-01-15',
    'IsMarketingOptIn': True,
})
```

### LoyaltyMemberCurrency
```python
sf.LoyaltyMemberCurrency.create({
    'Name': 'KM – Carlos Silva',
    'LoyaltyMemberId': member_id,
    'LoyaltyProgramCurrencyId': currency_id,
    # DO NOT set PointsBalance — it is read-only (system-calculated from transactions)
})
```

### LoyaltyMemberTier
```python
sf.LoyaltyMemberTier.create({
    'Name': 'Carlos – Bronze',
    'LoyaltyMemberId': member_id,
    'LoyaltyTierId': tier_id,
    'MembershipStartDate': '2024-01-15',
    # NO Status field via API
})
```

### LoyaltyProgramProcess
```python
# TransactionJournal type needs JournalTypeId
sf.LoyaltyProgramProcess.create({
    'Name': 'Acumulação KM',
    'LoyaltyProgramId': program_id,
    'ProcessType': 'TransactionJournal',
    'Status': 'Active',
    'JournalTypeId': journal_type_id,  # Required for TransactionJournal type
})

# TierProcessing type needs LoyaltyTierGroupId
sf.LoyaltyProgramProcess.create({
    'Name': 'Processamento de Tier',
    'LoyaltyProgramId': program_id,
    'ProcessType': 'TierProcessing',
    'Status': 'Active',
    'LoyaltyTierGroupId': tier_group_id,  # Required for TierProcessing
})

# CreditPointsToMembers (no extra required field)
sf.LoyaltyProgramProcess.create({
    'Name': 'Crédito de KMs Engajamento',
    'LoyaltyProgramId': program_id,
    'ProcessType': 'CreditPointsToMembers',
    'Status': 'Active',
})
```

### LoyaltyProgramJournalType
```python
sf.LoyaltyProgramJournalType.create({
    'Name': 'Abastecimento Combustível',
    'LoyaltyProgramId': program_id,
    'Status': 'Active',
})
```

### TransactionJournal
```python
sf.TransactionJournal.create({
    'LoyaltyProgramId': program_id,
    'LoyaltyProgramMemberId': member_id,
    'JournalTypeId': journal_type_id,
    'TransactionDate': '2025-08-15T10:30:00',
    'TransactionAmount': 150.0,
    'PointsToProcess': 150.0,
    'Status': 'Processed',
    # DO NOT set CurrencyIsoCode in single-currency orgs (causes error)
    # DO NOT set Points directly
})
```

### Quick Promotions (Promotion with points)
```python
from datetime import date
sf.Promotion.create({
    'Name': 'Bônus Abastecimento Duplo',
    'Status': 'Active',
    'StartDate': date.today().isoformat(),
    'EndDate': (date.today() + timedelta(days=30)).isoformat(),
    'LoyaltyProgramId': program_id,
    'TransactionJournalType': 'Accrual',
    'PointFactor': 2.0,
    'Description': 'Acumule KMs em dobro no abastecimento durante 30 dias',
    # NOT TotalRewardPoints (invalid for non-voucher promotions)
})
```

### VoucherDefinition
```python
sf.VoucherDefinition.create({
    'Name': 'Voucher Desconto Combustível R$10',
    'Description': 'Desconto de R$10 no abastecimento',
    'DiscountPercent': 10.0,
    'EffectiveDays': 90,
    'VoucherType': 'DiscountPercentage',
    'Status': 'Active',
    # PartnerAccountId is optional
})
```

### Voucher (issued to member)
```python
import random, string

def rand_code(prefix='KMV', length=8):
    return prefix + ''.join(random.choices(string.digits, k=length))

sf.Voucher.create({
    'VoucherDefinitionId': vd_id,
    'LoyaltyProgramMemberId': member_id,
    'ContactId': contact_id,
    'EffectiveDate': date.today().isoformat(),
    'ExpirationDate': (date.today() + timedelta(days=90)).isoformat(),
    'Status': 'Issued',
    'VoucherCode': rand_code('KMV'),
    'Reason': 'Benefício tier Ouro – trimestral',
})
```

### BenefitType — CRITICAL: must be linked to LoyaltyProgram
```python
# The BenefitType.LoyaltyProgramId is what makes LoyaltyTierBenefit work.
# The validation "Specify a tier and benefit that belong to the same loyalty program"
# checks BenefitType.LoyaltyProgramId == LoyaltyTier's LoyaltyProgram.
# Using a BenefitType without LoyaltyProgramId (or with a different program) will ALWAYS fail.

UOM_IDS = {
    'Percent':    '0hEHn0000008pPgMAI',   # For percentage discounts
    'Fee Waiver': '0hEHn0000008pPZMAY',   # For service benefits
}

sf.BenefitType.create({
    'Name': 'Desconto Km de Vantagens',
    'LoyaltyProgramId': loyalty_program_id,  # REQUIRED for LoyaltyTierBenefit
    'ProcessType': 'Loyalty',
    'UnitofMeasureId': UOM_IDS['Percent'],    # REQUIRED field
})
```

### Benefit
```python
sf.Benefit.create({
    'Name': 'Desconto 5% Loja ampm',
    'BenefitTypeId': benefit_type_id,   # Must use BenefitType with correct LoyaltyProgramId
    'BenefitStatus': 'Active',
    'IsActive': True,
    'Value': 5.0,
    'Description': 'Desconto de 5% em todos os produtos da loja ampm',
    # ProgramId (Program object, 11W prefix) is optional and NOT used for LoyaltyTierBenefit validation
})
```

### LoyaltyTierBenefit
```python
# Only works if Benefit.BenefitType.LoyaltyProgramId == LoyaltyTier's LoyaltyProgram
sf.LoyaltyTierBenefit.create({
    'Name': 'Bronze – Desconto 5% Loja ampm',
    'LoyaltyTierId': tier_id,
    'BenefitId': benefit_id,
})
```

---

## LoyaltyProgramSetup — Metadata Deploy for Process Rules

**Process Rules and Parameters CANNOT be created via REST API or Apex Anonymous.**  
They MUST be deployed via `LoyaltyProgramSetup` metadata XML.

### SFDX Project setup
```bash
mkdir -p /tmp/loyalty_sfdx/force-app/main/default/loyaltyProgramSetups
cd /tmp/loyalty_sfdx
cat > sfdx-project.json << 'EOF'
{
  "packageDirectories": [{ "path": "force-app", "default": true }],
  "namespace": "",
  "sfdcLoginUrl": "https://login.salesforce.com",
  "sourceApiVersion": "68.0"
}
EOF
```

### Retrieve existing config (optional, for reference)
```bash
cat > /tmp/loyalty_sfdx/package.xml << 'EOF'
<?xml version="1.0" encoding="UTF-8"?>
<Package xmlns="http://soap.sforce.com/2006/04/metadata">
    <types>
        <members>Km_de_Vantagens</members>
        <name>LoyaltyProgramSetup</name>
    </types>
    <version>68.0</version>
</Package>
EOF
sf project retrieve start --manifest /tmp/loyalty_sfdx/package.xml --target-org {alias} --output-dir /tmp/loyalty_sfdx/retrieved
```

### XML Structure
```xml
<?xml version="1.0" encoding="UTF-8"?>
<LoyaltyProgramSetup xmlns="http://soap.sforce.com/2006/04/metadata">
    <label>Km de Vantagens</label>
    <loyaltyProgram>Km_de_Vantagens</loyaltyProgram>
    <programProcesses>
        <processName>Acumulacao_KM</processName>
        <processType>TransactionJournal</processType>
        <status>Active</status>
        <journalTypeName>Abastecimento_Combustivel</journalTypeName>
        <programProcessRules>
            <ruleName>Acumulacao_por_Abastecimento</ruleName>
            <ruleLabel>Acumulação por Abastecimento – 100 KM</ruleLabel>
            <ruleStatus>Active</ruleStatus>
            <executionOrder>1</executionOrder>
            <ruleActions>
                <actionName>CreditPoints</actionName>
                <actionType>CreditPoints</actionType>
                <actionParameters>
                    <parameterName>TransactionAmount</parameterName>
                    <parameterValue>100</parameterValue>
                </actionParameters>
            </ruleActions>
        </programProcessRules>
        <!-- Add more rules here -->
    </programProcesses>
</LoyaltyProgramSetup>
```

### Deploy
```bash
sf project deploy start \
  --source-dir /tmp/loyalty_sfdx/force-app/main/default/loyaltyProgramSetups \
  --target-org {alias}
```

---

## Brand Templates

### Ipiranga — "Km de Vantagens"

#### Program Structure
- **Program Name**: Km de Vantagens
- **Currency**: KM (Quilômetros), type=Qualifying
- **TierGroup**: Categorias KM, TierModel=Anniversary, QpResetPeriod=Years
- **Tiers**: Bronze (0–500), Prata (501–2000), Ouro (2001–5000), Diamante (5001+)

#### Members (10 fictitious contacts)
```json
[
  { "FirstName": "Carlos",      "LastName": "Silva",        "Email": "carlos.silva@demoemail.com",       "MemberNumber": "KM100000", "Tier": "Ouro",     "Balance": 3420 },
  { "FirstName": "Ana",         "LastName": "Beatriz Santos","Email": "ana.santos@demoemail.com",         "MemberNumber": "KM100001", "Tier": "Prata",    "Balance": 1150 },
  { "FirstName": "João",        "LastName": "Ribeiro",      "Email": "joao.ribeiro@demoemail.com",       "MemberNumber": "KM100002", "Tier": "Diamante", "Balance": 28900 },
  { "FirstName": "João",        "LastName": "Ferreira",     "Email": "joao.ferreira@demoemail.com",      "MemberNumber": "KM100003", "Tier": "Diamante", "Balance": 12340 },
  { "FirstName": "Márcia",      "LastName": "Oliveira",     "Email": "marcia.oliveira@demoemail.com",    "MemberNumber": "KM100004", "Tier": "Bronze",   "Balance": 280 },
  { "FirstName": "Pedro",       "LastName": "Henrique Lima","Email": "pedro.lima@demoemail.com",         "MemberNumber": "KM100005", "Tier": "Prata",    "Balance": 890 },
  { "FirstName": "Roberto",     "LastName": "Gomes",        "Email": "roberto.gomes@demoemail.com",      "MemberNumber": "KM100006", "Tier": "Ouro",     "Balance": 4750 },
  { "FirstName": "Fernanda",    "LastName": "Costa",        "Email": "fernanda.costa@demoemail.com",     "MemberNumber": "KM100007", "Tier": "Bronze",   "Balance": 120 },
  { "FirstName": "Ricardo",     "LastName": "Almeida",      "Email": "ricardo.almeida@demoemail.com",    "MemberNumber": "KM100008", "Tier": "Ouro",     "Balance": 2980 },
  { "FirstName": "Marina",      "LastName": "Rocha",        "Email": "marina.rocha@demoemail.com",       "MemberNumber": "KM100009", "Tier": "Prata",    "Balance": 1670 }
]
```

#### Quick Promotions (3)
- **Bônus Abastecimento Duplo** — PointFactor=2.0, TransactionJournalType=Accrual, 30 days
- **Bônus ampm Fim de Semana** — PointFactor=1.5, TransactionJournalType=Accrual, 14 days
- **Resgate com Desconto Extra** — PointFactor=1.2, TransactionJournalType=Redemption, 60 days

#### VoucherDefinitions (3)
- **Voucher Desconto Combustível R$10** — VoucherType=DiscountPercentage, DiscountPercent=10, EffectiveDays=90
- **Voucher Desconto Combustível R$25** — VoucherType=DiscountPercentage, DiscountPercent=25, EffectiveDays=180
- **Voucher ampm 20% Desconto** — VoucherType=DiscountPercentage, DiscountPercent=20, EffectiveDays=90

#### Benefits per Tier (14 total)
**Bronze (2)**: Desconto 5% Loja ampm (Value=5), Acesso a Promoções Básicas  
**Prata (3)**: Desconto 10% Loja ampm (Value=10), Fila Prioritária Pagamento, KMs Extra no Aniversário (Value=500)  
**Ouro (4)**: Desconto 15% Loja ampm (Value=15), Lavagem Gratuita Mensal, Acesso Lounge Ipiranga, KMs Expiram em 18 Meses  
**Diamante (5)**: Desconto 20% Loja ampm (Value=20), KMs Não Expiram, Gerente de Conta Dedicado, Lavagem Gratuita Semanal, Upgrade de Combustível Gratuito

#### Currencies (3)
- **KM** — CurrencyType=Qualifying, IsPrimary=True (moeda principal de acúmulo)
- **Cashback R$** — CurrencyType=NonQualifying, CostperUnit=0.01 (100 KMs = R$1,00), ExpiryModel=Activity, ExpiryPeriodFrequency=12, ExpiryPeriodUnit=Months
- **KM Resgate** — CurrencyType=NonQualifying, ExpiryModel=Activity (para resgates)

#### Process Rules (via LoyaltyProgramSetup metadata)
**TransactionJournal process (Accrual)**:
1. Acumulação por Abastecimento – 100 KM por transação
2. Bônus Abastecimento R$200+ – 50 KM extra quando TransactionAmount ≥ 200
3. Acumulação Loja ampm – 25 KM

**TierProcessing process**:
1. Mudança de Tier por Saldo KM — GetMemberTier/GetMemberPointBalance/ChangeMemberTier actions
   - Bronze: balance < 501 | Prata: 501–2000 | Ouro: 2001–5000 | Diamante: 5001+

**CreditPointsToMembers process**:
1. Bônus Instalação App – 200 KM | 2. Bônus Completar Perfil – 100 KM | 3. Bônus Aniversário – 500 KM

#### Portal Brand Config (Ipiranga)
```python
IPIRANGA_BRAND = {
    'program_name'      : 'Km de Vantagens',
    'url_path_prefix'   : 'kmdevantagens',
    'primary_color'     : 'rgb(255, 182, 0)',       # #FFB600 — amarelo
    'primary_rgb'       : '255, 182, 0',
    'action_color'      : 'rgb(27, 117, 187)',       # #1B75BB — azul
    'action_color_dark' : 'rgb(18, 88, 145)',        # hover
    'action_color_darker': 'rgb(12, 60, 100)',
    'action_color_trans': 'rgba(27, 117, 187, 0.9)',
    'hero_caption'      : 'Programa de Fidelidade',
    'hero_subtitle'     : 'Acumule KMs e troque por vantagens exclusivas',
    'hero_cta_text'     : 'Ver Meus KMs',
    'login_bg_color'    : 'rgb(255, 182, 0)',
    'custom_css': """
/* Ipiranga — Km de Vantagens: Yellow #FFB600 + Blue #1B75BB */
.siteforceHeader, .comm-header, nav.comm-nav-menu {
  background-color: rgb(255, 182, 0) !important; }
.siteforceHeader a, .comm-header a, nav.comm-nav-menu a {
  color: rgb(27, 117, 187) !important; font-weight: 600; }
.slds-button--brand, .slds-button_brand {
  background-color: rgb(27, 117, 187) !important;
  border-color: rgb(27, 117, 187) !important; color: #fff !important; }
.slds-card__header { border-top: 3px solid rgb(255, 182, 0); }
a { color: rgb(27, 117, 187); } a:hover { color: rgb(18, 88, 145); }
"""
}
```

---

### Generic Template (non-pre-configured brands)

When a new brand is requested, ask:
1. Program name + URL path prefix (no spaces/accents)
2. Currency name + code + cashback rate (e.g. 100 pts = R$1)
3. Tier names (2–5 tiers) + balance thresholds
4. Industry (retail, fuel, financial, airline, telecom)
5. How many members (default: 10)
6. Brand colors: primary (header/background) + action (buttons/links)
7. Logo file path (optional — user can drop file in chat)

Then adapt the Ipiranga template structure with the new brand data. For the portal, use the provided colors in place of `IPIRANGA_BRAND`.

---

## Error Handling

- If `SalesforceMalformedRequest`: show the field causing the error. Most common issue is wrong field names (see Critical section above)
- If record with same Name already exists: query and reuse the existing ID (idempotent)
- If org token is expired: show `sf org login web --alias {alias}`
- Always capture and display which object type + field caused the error
- If LoyaltyTierBenefit fails "must belong to same loyalty program": check that BenefitType.LoyaltyProgramId is set correctly

---

## Important Notes

- Use `.demo` and `.demoemail.com` suffixes on emails to avoid real contact conflicts
- Never store access tokens in files — pass them as session variables only
- Loyalty Management must be enabled: Setup → Loyalty Management → Enable
- Always use `version='68.0'` in Salesforce constructor to access all Loyalty fields
- `LoyaltyProgramProcessRule` and `LoyaltyProgramProcessParameter`: REST API and Apex Anonymous will FAIL with "entity type cannot be inserted". Use LoyaltyProgramSetup metadata XML only.
- `LoyaltyMemberCurrency.PointsBalance` is read-only. Actual balances come from TransactionJournal records.
- `BenefitType.LoyaltyProgramId` MUST point to your LoyaltyProgram for LoyaltyTierBenefit to work. This is the most non-obvious requirement in the Loyalty object model.
- `Benefit.ProgramId` references the generic `Program` object (11W prefix), NOT `LoyaltyProgram` (0lp prefix). These are completely different objects. Don't try to use LoyaltyProgram ID for ProgramId.
- In single-currency orgs (USD), do NOT set CurrencyIsoCode on TransactionJournal.
- UnitOfMeasure IDs for BenefitType (mcn-audience org): Percent=0hEHn0000008pPgMAI, Fee Waiver=0hEHn0000008pPZMAY
- `LoyaltyProgramCurrency.ExtendExpiration` is a picklist with values `['None', 'TillEndOfMonth']` — not a boolean.
- Cashback currency: use CurrencyType=NonQualifying, CostperUnit=exchange_rate (e.g. 0.01 = 100pts → R$1). Do NOT set ExpiryModel fields on Qualifying currencies.
- Experience Cloud portal: use `sf community create --template-name "Loyalty Member Portal"`. The ExperienceBundle name will be `{SiteName_no_spaces}1`. Branding is applied by modifying `brandingSets/loyaltyMemberPortal.json` (colors/logo) and `themes/loyaltyMemberPortal.json` (header/hero/CSS) then deploying via `sf project deploy start --manifest`.
- Logo upload: use ContentVersion REST API with base64-encoded body → get ContentDocumentId → URL = `/sfc/servlet.shepherd/document/download/{ContentDocumentId}` → set as `CompanyLogo` in brandingSet.
- bgImage in theme hero MUST be a valid CSS background string with a real URL — empty `url()` causes deploy failure. Use the system default: `url(/sfsites/assets/themes/loyalty-home-banner.jpg)`.
