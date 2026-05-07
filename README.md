# BCA — BRICS Clearing Architecture

![Tests](https://img.shields.io/badge/tests-11%2F11%20passing-brightgreen)
![Solidity](https://img.shields.io/badge/Solidity-0.8.24-blue)
![Foundry](https://img.shields.io/badge/Foundry-tested-orange)
![License](https://img.shields.io/badge/license-MIT-green)

Implementação on-chain do mecanismo de reconvergência elástica não-linear para tokens de clearing interbancário multilateral, descrito em:

> **"Arquitetura Monetária Elástica para o BRICS"** — Armando José Freire de Melo (Manuscrito v40, 2026)
> Submetido ao **CBlockchain 2027** (SBC)

---

## O que é

O BCA é um token de clearing exclusivamente interbancário — circula entre Bancos Centrais membros para liquidar posições de comércio bilateral sem intermediário externo, sem dólar, sem SWIFT.

O agente econômico (empresa, pessoa) nunca vê o token. Paga em moeda doméstica, recebe em moeda doméstica. O clearing acontece na camada interbancária, invisível para qualquer agente externo.

**Não é stablecoin. Não é criptomoeda especulativa. É infraestrutura monetária.**

---

## Mecanismo Central

### Fundamental F_t — Mini-PIB on-chain

O valor fundamental do token é ancorado ao volume real de clearing registrado on-chain:

```
g_t  = log(V_t / V_{t-1})
ĝ_t  = α_ema · g_t + (1 − α_ema) · ĝ_{t-1}
F_t  = F_{t-1} · exp(ĝ_t)
```

Com `α_ema = 0.70` (meia-vida ~1 mês), choques econômicos reais aparecem em `F_t` em até 30 dias — sem dependência de oráculos externos após o genesis.

### Reconvergência Elástica (Eq. 3.2)

```
P_{t+1} = P_t · [1 − α · tanh(k · (P_t/F_t − 1))]
```

A função `tanh` limita a força de correção em desvios extremos — evita sobrecorreção. O mecanismo é implementado internamente via `exp` sem dependência de versão da PRBMath.

### Condição de Estabilidade (Teorema 1)

O ponto fixo `P* = F` é localmente estável se e somente se:

```
0 < α · k < 2
```

Verificada automaticamente no constructor. Deploy com `α·k ≥ 2` é rejeitado on-chain.

---

## Resultados dos Testes (11/11 passando)

| # | Teste | O que prova |
|---|---|---|
| 1 | `test_Genesis` | Supply = 50% PIB Brasil 2025 = 6,35 tri tokens, F=P=1.0 |
| 2 | `test_AntiPyramid_SameJurisdiction` | Clearing entre mesma jurisdição rejeitado on-chain |
| 3 | `test_AntiPyramid_NonMember` | Token não transferível para endereços externos |
| 4 | `test_AdmissionRejectsOver50pct` | Teto de 50% do supply por membro funciona |
| 5 | `test_FRisesWithClearing` | Mini-PIB on-chain: F_t sobe com volume real |
| 6 | `test_Reconvergence` | P converge para F após choque de -33% em 60 passos |
| 7 | `test_BasinOfAttraction` | 7 condições iniciais extremas (P/F de 0.1 a 7.39) convergem para desvio 0% |
| 8 | `test_StabilityCondition_Theorem1` | α·k=0.075 converge; deploy com α·k≥2 rejeitado |
| 9 | `test_EMA_ResponseSpeed` | F_t sobe monotonicamente a cada passo de volume |
| 10 | `test_AttackerROI_Negative` | Atacante com 2 jurisdições coniventes: ROI = 0%, capital idêntico |
| 11 | `test_LongTermAppreciation` | F_t aprecia +38% em 10 anos (2520 dias) sem colapso |

### Trajetória de apreciação (Teste 11)

| Ano | F_t | Apreciação |
|---|---|---|
| 0 (genesis) | 1.000 | — |
| 1 | 1.032 | +3% |
| 3 | 1.101 | +10% |
| 5 | 1.174 | +17% |
| 7 | 1.253 | +25% |
| 10 | 1.382 | +38% |

Volume crescendo 7% ao ano. Sem colapso. Apreciação monotônica.

---

## Genesis — Brasil (MVP)

O protocolo nasce em 2026 com o Brasil como membro originário.

| Parâmetro | Valor | Fonte |
|---|---|---|
| Ano base de referência | 2025 | PIB do membro originário |
| PIB Brasil 2025 | R$ 12,7 trilhões | IBGE, divulgação 03/03/2026 |
| Mint genesis | 50% do PIB = 6,35 tri tokens | Regra universal |
| Valor inicial por token | R$ 1,00 | F_0 = 1.0 normalizado |
| Mercado secundário | Não existe | Token é instrumento de clearing |

**Regra universal de admissão:** todo membro minta 50% do seu PIB 2025, independente de quando aderiu. Ano base fixo = 2025 para todos.

---

## Parâmetros Baseline

| Parâmetro | Símbolo | Valor | Descrição |
|---|---|---|---|
| Velocidade de reconvergência | α | 0.50 | Controla intensidade da correção |
| Expoente de rigidez | k | 0.15 | Inclinação do tanh |
| Produto de estabilidade | α·k | 0.075 | Deve ser < 2 (Teorema 1) |
| EMA Mini-PIB | α_ema | 0.70 | Meia-vida ~1 mês |
| Teto por membro | — | 50% | Do supply total pós-admissão |
| Mint por membro | — | 50% PIB 2025 | Regra universal |

---

## Como Rodar

### Requisitos

- [Foundry](https://book.getfoundry.sh/getting-started/installation)
- Git

### Instalação

```bash
git clone https://github.com/armanfm/clearing-chain.git
cd clearing-chain

git submodule add https://github.com/OpenZeppelin/openzeppelin-contracts lib/openzeppelin-contracts
git submodule add https://github.com/PaulRBerg/prb-math lib/prb-math
```

Configure o `remappings.txt`:

```
@openzeppelin/contracts/=lib/openzeppelin-contracts/contracts/
@prb/math/=lib/prb-math/
```

### Compilar

```bash
forge build
```

### Rodar testes

```bash
# Todos os testes
forge test -vv

# Teste específico
forge test -vv --match-test test_LongTermAppreciation

# Apenas resultado (sem logs)
forge test
```

---

## Estrutura do Projeto

```
clearing-chain/
├── src/
│   └── BCA.sol              # Contrato principal (223 linhas)
├── test/
│   └── BCA.t.sol            # 11 testes Foundry
├── script/
│   └── Deploy.s.sol         # Script de deploy
├── lib/
│   ├── openzeppelin-contracts/
│   └── prb-math/
└── foundry.toml
```

---

## Propriedades do Protocolo

**Sem mercado secundário por construção**
O override do `_update` do ERC-20 impede transferências para qualquer endereço que não seja membro registrado. Tokens circulam exclusivamente entre BCs autorizados.

**Anti-pirâmide por protocolo**
Cada transação de clearing exige dois NPos de jurisdições distintas. Transações intra-jurisdição são rejeitadas on-chain antes de qualquer cálculo de volume. Um membro não pode inflar o Mini-PIB transacionando consigo mesmo.

**Genesis imutável**
S_0 é mintado uma única vez no constructor. Oferta fixa pós-genesis — sem mint de crescimento. F_t aprecia com o volume real sem diluição.

**Condição de estabilidade verificada no deploy**
O constructor rejeita qualquer combinação de parâmetros que viole `0 < α·k < 2`. Impossível deployar um contrato instável.

---

## Referências

- Keynes, J. M. (1943). *Proposals for an International Clearing Union*. British Government White Paper.
- Triffin, R. (1960). *Gold and the Dollar Crisis*. Yale University Press.
- BIS Innovation Hub (2022). *Project mBridge*. Bank for International Settlements.
- Strogatz, S. (2015). *Nonlinear Dynamics and Chaos*. 2nd ed. Westview Press.
- de Melo, A. J. F. (2026). *Arquitetura Monetária Elástica para o BRICS*. Manuscrito v40.

---

## Autor

**Armando Freire**
Pesquisa Independente
[armanfm@github.com](mailto:armanfm@github.com)
[github.com/armanfm](https://github.com/armanfm)

---

*Protocolo nascido em 2026. Ano base de referência: 2025.*

